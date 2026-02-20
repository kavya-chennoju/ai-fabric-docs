# Fabric Mesh Setup

You're the administrator. You need to stand up the Fabric mesh so that devices can register and agents can discover them.

The infrastructure has three services:

| Service | What it does | Port |
|---------|-------------|------|
| **NATS** | Message routing between devices and agents (JWT auth, TLS) | 4222 |
| **etcd** | Stores device registrations with TTL leases | 2379 |
| **Device Registry Service** | Listens for device registrations, tracks heartbeats, publishes online/offline events | 8080 |

## Prerequisites

- Docker and Docker Compose
- `nsc` — the NATS CLI tool for JWT/NKey management (`brew install nsc` or `go install github.com/nats-io/nsc/v2@latest`)
- `ai-fabric-core` installed (`pip install "ai-fabric-core[security]"`)

## Start the Services

```bash
docker compose up nats-jwt etcd device-registry-service -d
```

This starts:

- **nats-jwt** — NATS 2.10 with JWT authentication enabled, using a generated resolver config
- **etcd** — v3.5 key-value store for device registration persistence
- **device-registry-service** — Python service that bridges NATS and etcd

### Verify Services Are Running

```bash
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

```
NAMES                    STATUS              PORTS
device-registry-service  Up 5 minutes        0.0.0.0:8080->8080/tcp
nats-jwt                 Up 5 minutes        0.0.0.0:4222->4222/tcp, 6222/tcp, 8222/tcp
etcd                     Up 5 minutes        0.0.0.0:2379-2380->2379-2380/tcp
```

## JWT Authentication Setup

NATS uses JWT + NKey authentication. Every device and agent gets a signed JWT that defines what NATS subjects it can publish and subscribe to.

### One-Time Setup

Initialize the NSC operator, account, and signing keys:

```bash
# Set up the NATS JWT infrastructure
./setup_jwt_auth.sh dev

# Generate credentials for all built-in roles
./gen_creds.sh --all --force
```

This creates:

```
security_infra/
├── credentials/
│   ├── devctl.creds.json          # Admin CLI credentials
│   ├── orchestrator.creds.json    # Agent/orchestrator credentials
│   └── registry.creds.json        # Registry service credentials
├── certs/
│   └── ca-cert.pem                # TLS CA certificate (if TLS enabled)
└── nats-jwt-generated.conf        # Generated NATS server config
```

!!! warning "nats-jwt-generated.conf Must Be a File"
    Docker mounts this file into the NATS container. If it doesn't exist when you first run `docker compose up`, Docker will auto-create it as a **directory**, which breaks the mount. Always run `setup_jwt_auth.sh` before starting Docker. If you see mount errors, remove the directory and regenerate:

    ```bash
    rm -rf security_infra/nats-jwt-generated.conf
    ./setup_jwt_auth.sh dev
    ./gen_creds.sh --all --force
    ```

### Credential File Format

Each `.creds.json` file contains everything a client needs to authenticate:

```json
{
  "device_id": "devctl",
  "auth_type": "jwt",
  "tenant": "default",
  "nats": {
    "urls": ["nats://nats-jwt:4222"],
    "jwt": "eyJ0eXAiOiJKV1Qi...",
    "nkey_seed": "SUAE..."
  }
}
```

The JWT encodes:

- Which NATS subjects this client can publish to
- Which subjects it can subscribe to
- Expiry time (default: 90 days)
- Account and user identity

## Device Registry Service

The registry service is a Python process that runs inside Docker. It:

1. Subscribes to `fabric.{tenant}.registry` for device registration RPCs
2. Subscribes to `fabric.{tenant}.discovery` for device discovery queries
3. Subscribes to `fabric.{tenant}.*.heartbeat` for device heartbeats
4. Stores device records in etcd with TTL leases
5. Publishes `device/online` and `device/offline` events when devices join or leave

### Configuration

| Environment Variable | Description | Default |
|---------------------|-------------|---------|
| `TENANT` | Single tenant namespace | `default` |
| `TENANTS` | Comma-separated list for multi-tenant | — |
| `NATS_CREDENTIALS_FILE` | Path to registry `.creds.json` | — |
| `NATS_URLS` | NATS server URLs | `nats://nats-jwt:4222` |
| `ETCD_HOST` | etcd hostname | `etcd` |
| `ETCD_PORT` | etcd port | `2379` |

### What Gets Stored in etcd

Each device registration creates a key at `/fabric/{tenant}/devices/{device_id}`:

```json
{
  "device_id": "robot-001",
  "device_type": "cleaning_robot",
  "device_ttl": 15,
  "capabilities": {
    "functions": [
      {
        "name": "start_cleaning",
        "description": "Start cleaning a zone.",
        "parameters": { "type": "object", "properties": { "zone": { "type": "string" } } }
      }
    ],
    "events": [
      { "name": "cleaning_complete", "description": "Cleaning finished." }
    ]
  },
  "identity": {
    "device_type": "cleaning_robot",
    "manufacturer": "RoboCorp",
    "model": "CleanBot-3000",
    "firmware_version": "4.2.1"
  },
  "status": {
    "location": "warehouse-B",
    "availability": "available"
  },
  "registry": {
    "device_registration_id": "a1b2c3d4-...",
    "registered_at": "2026-02-20T10:30:00Z"
  }
}
```

The record has an etcd lease with TTL. If the device stops sending heartbeats, the lease expires and the registry publishes a `device/offline` event.

### NATS Topic Structure

All messaging follows this pattern:

```
fabric.{tenant}.{device_id}.cmd              ← RPC calls to a device
fabric.{tenant}.{device_id}.event.{name}     ← Events from a device
fabric.{tenant}.{device_id}.heartbeat        ← Device heartbeats
fabric.{tenant}.registry                     ← Device registration RPCs
fabric.{tenant}.discovery                    ← Device discovery queries
fabric.{tenant}.device.online                ← Registry: device came online
fabric.{tenant}.device.offline               ← Registry: device went offline
```

## Device Commissioning

New devices need credentials before they can join the mesh. The commissioning flow is:

### 1. Factory Provisioning

Generate a device identity with a factory PIN:

```bash
python provision_device.py \
  --device-id robot-001 \
  --device-type cleaning_robot \
  --capabilities start_cleaning,stop_cleaning,get_battery_level
```

This produces a `robot-001.identity.json` that gets burned onto the device. It contains:

- Device ID and type
- Ed25519 NKey keypair (public key + seed)
- 8-digit factory PIN (bcrypt-hashed)
- `commissioned: false`

### 2. Commission the Device

The device boots, reads its identity file, sees `commissioned: false`, and starts an HTTP commissioning server on port 5540.

From your admin workstation:

=== "Auto-discover via mDNS"

    ```bash
    # Find uncommissioned devices on the local network
    devctl discover --timeout 5

    # Commission with PIN (device found via mDNS)
    devctl commission robot-001 --pin 12345678
    ```

=== "Specify device IP"

    ```bash
    devctl commission robot-001 \
      --pin 12345678 \
      --device-ip 192.168.1.50
    ```

What happens:

1. `devctl` contacts the device at `http://{ip}:5540/info` to get its NKey public key
2. Generates a JWT signed for this device
3. POSTs the JWT + credentials to `http://{ip}:5540/commission` with the PIN
4. Device validates the PIN (bcrypt, 3 attempts max, 1-hour lockout)
5. Device saves credentials to disk (`~/.fabric/credentials/robot-001.creds.json`)
6. Device connects to NATS and self-registers

### 3. Verify

```bash
devctl list
```

The device should appear in the registry.

## Admin CLI — devctl

`devctl` is the admin command-line tool for managing the Fabric mesh. Installed with `ai-fabric-core`.

### List Devices

```bash
# Full output
devctl list

# Compact — one line per device with functions and events
devctl list --compact
```

### Discover Uncommissioned Devices

```bash
devctl discover --timeout 10
```

Scans the local network via mDNS for devices running commissioning servers.

!!! note "Requires ai-fabric-core[security]"
    Discovery and commissioning require the security extras: `pip install "ai-fabric-core[security]"`

### Commission a Device

```bash
devctl commission <device_id> --pin <pin> [--device-ip <ip>]
```

### Register a Test Device

For testing without real hardware:

```bash
# Register and immediately exit
devctl register --id test-device-001

# Register and keep alive (sends heartbeats)
devctl register --id test-device-001 --keepalive
```

### Interactive REPL

```bash
devctl interactive
```

Opens an interactive session for calling device functions, listing devices, and debugging the mesh.

### Configuration

`devctl` reads from environment variables:

| Variable | Description | Default |
|----------|-------------|---------|
| `NATS_URL` | NATS server URL | `nats://localhost:4222` |
| `NATS_CREDENTIALS_FILE` | Admin credentials file | auto-discover |
| `TENANT` | Namespace | `default` |
| `DEVICE_TTL` | Heartbeat timeout (seconds) | `15` |

## Multi-Tenancy

The mesh supports multiple tenants. Each tenant is a separate namespace — devices in one tenant are invisible to another.

```bash
# Registry service handles multiple tenants
TENANTS=warehouse,factory,lab docker compose up device-registry-service -d
```

NATS subject scoping ensures isolation:

- Warehouse agent credentials: can only access `fabric.warehouse.*`
- Factory agent credentials: can only access `fabric.factory.*`

Tenant scoping is enforced at the NATS JWT level. Even if an agent knows a device ID in another tenant, the NATS server will deny the message.

## Monitoring

### NATS Monitoring

NATS exposes an HTTP monitoring endpoint on port 8222:

```bash
# Server info
curl http://localhost:8222/varz

# Active connections
curl http://localhost:8222/connz

# Active subscriptions
curl http://localhost:8222/subsz
```

### etcd — Check Device Registrations

```bash
# List all devices in default tenant
etcdctl get /fabric/default/devices/ --prefix --print-value-only | python -m json.tool

# Watch for registration changes
etcdctl watch /fabric/default/devices/ --prefix
```

### Device Online/Offline Events

Subscribe to registry lifecycle events:

```bash
# Using nats CLI
nats sub "fabric.default.device.>"
```

You'll see `device/online` when a device registers and `device/offline` when its heartbeat lease expires.

## Security Checklist

Before going to production:

- [ ] TLS enabled on NATS (`tls://` URLs, CA certificates distributed)
- [ ] Credential files have `0600` permissions (owner read/write only)
- [ ] NSC keys backed up and stored securely
- [ ] JWT expiry set to a reasonable window (not indefinite)
- [ ] Each device has unique credentials (no shared JWTs)
- [ ] `allow_insecure=False` on all production devices
- [ ] Multi-tenant scoping if running multiple fleets
- [ ] etcd authentication enabled
- [ ] Commissioning PINs are single-use and randomly generated
