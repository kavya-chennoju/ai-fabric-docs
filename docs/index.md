# AI Fabric

AI Fabric is a coordination layer for physical systems. Robots, cameras, sensors, and industrial controllers connect to a shared NATS mesh, expose typed functions and events, and become discoverable and controllable by AI agents — or by each other.

There are three packages. Use the ones you need.

## Packages

### ai-fabric-device

The device SDK. Install this on your edge hardware — Raspberry Pi, robot, sensor, controller. Write a `DeviceDriver` subclass, decorate your methods, and the runtime handles NATS connection, registration, heartbeats, and command routing.

```bash
pip install ai-fabric-device
```

```python
from fabric_device import DeviceRuntime
from fabric_device.drivers import DeviceDriver, rpc, emit

class TemperatureSensor(DeviceDriver):
    device_type = "temperature_sensor"

    @rpc()
    async def get_reading(self) -> dict:
        """Get current temperature and humidity."""
        return {"temperature_c": 22.5, "humidity_pct": 45}

    @emit()
    async def threshold_alert(self, temperature_c: float, severity: str):
        """Temperature exceeded a threshold."""
        pass
```

[Device Connectivity Guide →](device-connectivity.md)

---

### ai-fabric-tools

The agent SDK. Install this alongside your AI agent framework. Gives your agent four tool functions to discover devices and call their functions over the mesh.

```bash
pip install ai-fabric-tools[strands]
```

```python
from ai_fabric_tools import connect
from ai_fabric_tools.adapters.strands import discover_devices, invoke_device
from strands import Agent

connect()
agent = Agent(tools=[discover_devices, invoke_device])
agent("What devices are online? Check the battery on any robots.")
```

[Strands Integration Guide →](strands-integration.md)

---

### ai-fabric-core

The server package. Extends `ai-fabric-device` with the device registry, security (commissioning, ACLs), distributed state (etcd), audit logging (MongoDB), and CLI tools (`devctl`, `statectl`). Install this on your server or admin workstation.

```bash
pip install "ai-fabric-core[all]"
```

## How It Fits Together

```
┌──────────────┐       ┌──────────────┐       ┌──────────────┐
│  Strands     │       │   Fabric     │       │   Devices    │
│  Agent       │       │   Server     │       │              │
│              │       │              │       │  robot-001   │
│  ai-fabric-  │──────▶│  NATS + JWT  │◀──────│  camera-002  │
│  tools       │ JSON- │  Registry    │ self- │  sensor-003  │
│              │ RPC   │  etcd        │ reg.  │              │
│              │       │              │       │  ai-fabric-  │
│              │       │  ai-fabric-  │       │  device      │
│              │       │  core        │       │              │
└──────────────┘       └──────────────┘       └──────────────┘
```

**Devices** run `ai-fabric-device`, connect to NATS, self-register, and expose `@rpc` functions and `@emit` events.

**Agents** run `ai-fabric-tools`, discover devices on the mesh, and invoke their functions as part of the agent's normal reasoning loop.

**The server** runs `ai-fabric-core` with NATS (messaging), etcd (state), and the device registry. It routes messages and tracks which devices are online.

No component knows about the internals of the other. Devices don't know they're being called by an AI agent. Agents don't know what protocol the device uses internally. NATS handles the routing.
