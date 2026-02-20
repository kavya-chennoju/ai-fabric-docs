# Device Connectivity to Fabric

You manufacture a device — a robot, a sensor, a camera, a controller. You want it to be discoverable and controllable by AI agents and other devices on a Fabric mesh.

This page shows you how to write a device driver, connect your device to a NATS server, and expose typed functions and events that agents can discover and call.

## What Your Device Becomes

Once connected, your device:

- **Self-registers** with the device registry (device type, functions, events, identity)
- **Maintains a heartbeat** so the mesh knows it's alive
- **Receives RPC calls** from agents or other devices (JSON-RPC over NATS)
- **Emits events** that agents and other devices can subscribe to
- **Communicates device-to-device** without going through a central orchestrator

You write the device logic. The runtime handles registration, heartbeats, messaging, and command routing.

## Installation

```bash
pip install ai-fabric-device
```

Four dependencies: `nats-py`, `pydantic`, `nkeys`, `pyyaml`. Runs on a Raspberry Pi.

Optional: `pip install "ai-fabric-device[telemetry]"` adds OpenTelemetry distributed tracing.

## Your First Device

A complete working device in one file. This temperature sensor exposes one RPC function and emits one event:

```python
import asyncio
from fabric_device import DeviceRuntime
from fabric_device.drivers import DeviceDriver, rpc, emit
from fabric_device.types import DeviceIdentity, DeviceStatus


class TemperatureSensor(DeviceDriver):
    device_type = "temperature_sensor"

    @property
    def identity(self) -> DeviceIdentity:
        return DeviceIdentity(
            device_type="temperature_sensor",
            manufacturer="Acme Sensors",
            model="TH-200",
            firmware_version="2.1.0",
            description="Industrial temperature and humidity sensor",
        )

    @property
    def status(self) -> DeviceStatus:
        return DeviceStatus(
            location="warehouse-B",
            availability="available",
        )

    @rpc()
    async def get_reading(self) -> dict:
        """Get current temperature and humidity.

        Returns:
            Dictionary with temperature_c, humidity_pct, and unit.
        """
        # Replace with your actual hardware read
        return {"temperature_c": 22.5, "humidity_pct": 45, "unit": "celsius"}

    @emit()
    async def threshold_alert(self, temperature_c: float, threshold: float, severity: str):
        """Temperature exceeded a configured threshold.

        Args:
            temperature_c: Current temperature reading.
            threshold: The threshold that was crossed.
            severity: Alert severity — warning or critical.
        """
        pass

    async def connect(self) -> None:
        # Initialize your hardware, open serial ports, etc.
        pass

    async def disconnect(self) -> None:
        # Clean up hardware resources
        pass


async def main():
    device = DeviceRuntime(
        driver=TemperatureSensor(),
        device_id="temp-sensor-001",
        messaging_urls=["nats://your-server:4222"],
        nats_credentials_file="/path/to/temp-sensor-001.creds.json",
    )
    await device.run()


asyncio.run(main())
```

When this runs:

1. Connects to NATS with JWT authentication
2. Registers with the device registry: *"I am temp-sensor-001, type temperature_sensor, I have `get_reading` and `threshold_alert`"*
3. Starts sending heartbeats
4. Listens for incoming RPC calls on `fabric.default.temp-sensor-001.cmd`
5. When an agent calls `invoke_device("temp-sensor-001", "get_reading")`, your `get_reading` method executes and the result is sent back

## Writing a Device Driver

Your driver is a Python class that extends `DeviceDriver`. Everything the device can do is declared with decorators.

### Device Identity and Status

Every device reports who it is and what state it's in. Agents see this when they call `discover_devices`.

```python
from fabric_device.types import DeviceIdentity, DeviceStatus

class MyRobot(DeviceDriver):
    device_type = "cleaning_robot"

    @property
    def identity(self) -> DeviceIdentity:
        return DeviceIdentity(
            device_type="cleaning_robot",
            manufacturer="RoboCorp",
            model="CleanBot-3000",
            firmware_version="4.2.1",
            arch="aarch64",
            description="Autonomous floor cleaning robot with chemical handling",
        )

    @property
    def status(self) -> DeviceStatus:
        return DeviceStatus(
            location="building-A-floor-2",
            availability="available",  # or "busy", "offline", "maintenance"
        )
```

### @rpc — Functions Agents Can Call

Mark any async method with `@rpc()` to make it callable over the mesh. The method's name, docstring, and type hints are automatically converted into a function schema that agents can read.

```python
from fabric_device.drivers import DeviceDriver, rpc

class MyRobot(DeviceDriver):
    device_type = "cleaning_robot"

    @rpc()
    async def start_cleaning(self, zone: str, priority: str = "normal") -> dict:
        """Start cleaning a specified zone.

        Args:
            zone: Target zone ID (e.g., "zone-A", "loading-dock").
            priority: Cleaning priority — normal or urgent.

        Returns:
            Dictionary with status and estimated_duration_minutes.
        """
        await self._hardware.start(zone, priority)
        return {"status": "cleaning", "estimated_duration_minutes": 15}

    @rpc()
    async def get_battery_level(self) -> dict:
        """Get current battery percentage.

        Returns:
            Dictionary with battery_pct and charging status.
        """
        return {"battery_pct": 80, "charging": False}

    @rpc()
    async def emergency_stop(self) -> dict:
        """Immediately stop all movement and operations.

        Returns:
            Dictionary with confirmation status.
        """
        await self._hardware.halt()
        return {"status": "stopped"}
```

An agent that calls `discover_devices(device_type="cleaning_robot")` will see:

```json
{
  "functions": [
    {
      "name": "start_cleaning",
      "description": "Start cleaning a specified zone.",
      "parameters": {
        "type": "object",
        "properties": {
          "zone": {"type": "string", "description": "Target zone ID"},
          "priority": {"type": "string", "default": "normal"}
        }
      }
    },
    {
      "name": "get_battery_level",
      "description": "Get current battery percentage."
    },
    {
      "name": "emergency_stop",
      "description": "Immediately stop all movement and operations."
    }
  ]
}
```

!!! tip "Docstrings Matter"
    The function name, docstring, and `Args:` descriptions are what the AI agent reads to decide when and how to call your function. Write them like you're explaining the function to a person. Clear descriptions lead to better agent decisions.

### @emit — Events Your Device Publishes

Mark a method with `@emit()` to declare an event. Call the method from your code to publish it. Agents and other devices can subscribe.

```python
from fabric_device.drivers import DeviceDriver, emit

class MyRobot(DeviceDriver):
    device_type = "cleaning_robot"

    @emit()
    async def cleaning_complete(self, zone: str, duration_minutes: float):
        """Cleaning finished in a zone.

        Args:
            zone: Zone that was cleaned.
            duration_minutes: How long the cleaning took.
        """
        pass  # the body is empty — the runtime handles publishing

    @emit()
    async def battery_low(self, battery_pct: int):
        """Battery dropped below threshold.

        Args:
            battery_pct: Current battery percentage.
        """
        pass
```

Emit events from anywhere in your driver:

```python
await self.cleaning_complete(zone="zone-A", duration_minutes=12.5)
await self.battery_low(battery_pct=15)
```

Events are published to `fabric.{tenant}.{device_id}.event.{event_name}` on NATS. Any subscriber — agent or device — receives them.

### @periodic — Background Tasks

Run a method on a fixed interval. Useful for polling sensors, emitting telemetry, or running health checks.

```python
from fabric_device.drivers import DeviceDriver, rpc, emit, periodic

class TemperatureSensor(DeviceDriver):
    device_type = "temperature_sensor"

    @emit()
    async def reading(self, temperature_c: float, humidity_pct: float):
        """A new sensor reading."""
        pass

    @emit()
    async def threshold_alert(self, temperature_c: float, threshold: float, severity: str):
        """Temperature exceeded a threshold."""
        pass

    @periodic(interval=10.0)
    async def poll_sensor(self):
        """Read sensor every 10 seconds and emit events."""
        data = await self._read_hardware()
        await self.reading(temperature_c=data["temp"], humidity_pct=data["humidity"])

        if data["temp"] > 40.0:
            await self.threshold_alert(
                temperature_c=data["temp"],
                threshold=40.0,
                severity="critical",
            )

    @periodic(interval=60.0)
    async def health_check(self):
        """Run self-diagnostics every minute."""
        if not self._hardware.is_healthy():
            self.logger.warning("Hardware health check failed")
```

### @on — Listen to Other Devices

Subscribe to events from other devices on the mesh. Your device can react to what's happening around it without going through a central agent.

```python
from fabric_device.drivers import DeviceDriver, rpc, on

class CleaningRobot(DeviceDriver):
    device_type = "cleaning_robot"

    @on(device_type="camera", event_name="spill_detected")
    async def on_spill(self, device_id: str, event_name: str, payload: dict):
        """React when any camera detects a spill."""
        zone = payload.get("zone")
        self.logger.info(f"Spill detected by {device_id} in {zone}")
        await self.start_cleaning(zone=zone)

    @on(device_type="cleaning_robot", event_name="cleaning_complete")
    async def on_peer_done(self, device_id: str, event_name: str, payload: dict):
        """Track when other robots finish cleaning."""
        self.logger.info(f"{device_id} finished cleaning {payload.get('zone')}")

    @rpc()
    async def start_cleaning(self, zone: str) -> dict:
        """Start cleaning a zone."""
        return {"status": "cleaning", "zone": zone}
```

This is device-to-device communication. No agent involved. The camera emits `spill_detected`, the robot receives it directly over NATS and acts.

### @before_emit — Intercept Events

Filter or modify events before they're published. Return `False` to suppress an event.

```python
from fabric_device.drivers import DeviceDriver, emit, before_emit

class NoisySensor(DeviceDriver):
    device_type = "sensor"

    @emit()
    async def reading(self, value: float):
        """A sensor reading."""
        pass

    @before_emit("reading")
    async def filter_noise(self, value: float, **kwargs):
        """Suppress readings that haven't changed significantly."""
        if hasattr(self, "_last_value") and abs(value - self._last_value) < 0.5:
            return False  # suppress — don't publish
        self._last_value = value
```

## Lifecycle Methods

Every driver has `connect` and `disconnect`. Use them to initialize and clean up hardware resources.

```python
class MyRobot(DeviceDriver):
    device_type = "robot"

    async def connect(self) -> None:
        """Called after NATS connection is established, before RPCs start."""
        self._hardware = await RobotSDK.connect("/dev/ttyUSB0")
        self.logger.info("Hardware initialized")

    async def disconnect(self) -> None:
        """Called during shutdown, before NATS disconnects."""
        await self._hardware.close()
        self.logger.info("Hardware released")
```

## Running the Device

### DeviceRuntime

`DeviceRuntime` handles everything outside your driver logic: NATS connection, JWT authentication, device registration, heartbeats, command routing.

```python
from fabric_device import DeviceRuntime

device = DeviceRuntime(
    driver=MyRobot(),
    device_id="robot-001",
    messaging_urls=["nats://your-server:4222"],
    nats_credentials_file="~/.fabric/credentials/robot-001.creds.json",
    tenant="default",
)
await device.run()
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `driver` | `DeviceDriver` | Your driver instance |
| `device_id` | `str` | Unique device identifier on the mesh |
| `messaging_urls` | `list[str]` | NATS server URLs |
| `nats_credentials_file` | `str \| None` | Path to `.creds.json` for JWT auth |
| `tenant` | `str` | Namespace / zone (default: `"default"`) |
| `allow_insecure` | `bool` | Skip auth for local development (default: `False`) |

### Credentials

Devices authenticate to NATS using JWT + NKey credentials. A `.creds.json` file:

```json
{
  "device_id": "robot-001",
  "auth_type": "jwt",
  "tenant": "default",
  "nats": {
    "urls": ["nats://your-server:4222"],
    "jwt": "eyJ0eXAiOiJKV1Qi...",
    "nkey_seed": "SUAE..."
  }
}
```

Credentials are provisioned by the Fabric administrator and determine:

- Which NATS subjects the device can publish/subscribe to
- Which tenant namespace it operates in
- Its identity for JWT authentication

!!! note "Development Mode"
    For local development without auth, use `allow_insecure=True`. Never use this in production.

    ```python
    device = DeviceRuntime(
        driver=MyRobot(),
        device_id="robot-001",
        messaging_urls=["nats://localhost:4222"],
        allow_insecure=True,
    )
    ```

## Device-to-Device Communication

Every `DeviceDriver` has built-in methods for talking to other devices on the mesh. No agent required.

### invoke_remote — Call Another Device

```python
class Camera(DeviceDriver):
    device_type = "camera"

    @rpc()
    async def on_motion(self, zone: str) -> dict:
        """Motion detected — dispatch a robot."""
        robots = await self.list_devices(device_type="cleaning_robot")
        if robots:
            result = await self.invoke_remote(
                robots[0]["device_id"],
                "start_cleaning",
                zone=zone,
            )
            return {"dispatched": True, "result": result}
        return {"dispatched": False, "reason": "no robots available"}
```

### list_devices — Query the Registry

```python
# All devices
all_devices = await self.list_devices()

# Filter by type
robots = await self.list_devices(device_type="cleaning_robot")

# Filter by location
local = await self.list_devices(location="building-A")
```

## What Happens at Runtime

When `device.run()` is called, this is the sequence:

1. **Connect to NATS** — authenticates with JWT + NKey
2. **Connect driver** — calls your `connect()` method
3. **Register** — sends device identity, functions, and events to the registry
4. **Subscribe to commands** — listens on `fabric.{tenant}.{device_id}.cmd`
5. **Start periodic tasks** — any `@periodic` methods begin running
6. **Start event subscriptions** — any `@on` handlers are wired up
7. **Heartbeat loop** — periodic heartbeats keep the registration alive

From this point, your device is live on the mesh. Agents can discover it and call its functions. Other devices can send it events. Your periodic tasks run in the background.

When `device.stop()` is called (or Ctrl+C):

1. Periodic tasks stop
2. Event subscriptions tear down
3. Your `disconnect()` method runs
4. NATS connection closes

## Decorator Summary

| Decorator | What it does |
|-----------|-------------|
| `@rpc()` | Expose a method as a remotely callable function (agent or D2D) |
| `@emit()` | Declare an event that can be published to subscribers |
| `@periodic(interval=N)` | Run a method every N seconds in the background |
| `@on(device_type=..., event_name=...)` | Subscribe to events from other devices |
| `@before_emit("event_name")` | Intercept and filter/modify an event before publishing |
