# Strands Integration

Give your Strands agent the ability to discover and control physical devices — robots, cameras, sensors, industrial controllers — over a NATS messaging mesh.

Your devices are already on the network, exposing typed functions. The `ai-fabric-tools` SDK lets your agent see them and call them.

## What You Get

- **`discover_devices`** — see every device on the mesh, its functions, parameter schemas, and current status
- **`invoke_device`** — call any function on any device by name and get the result back
- **`invoke_device_with_fallback`** — try a list of devices in order until one succeeds
- **`get_device_status`** — check a specific device before acting on it

These are standard Strands tools. They show up in your agent's tool list alongside anything else you provide. The agent decides when to use them as part of its normal reasoning.

## Installation

```bash
pip install ai-fabric-tools[strands]
```

## Prerequisites

You need:

1. A running NATS server with devices connected (your Fabric mesh)
2. A credentials file (`.creds.json`) that grants your agent access to the devices it needs
3. An `ANTHROPIC_API_KEY` for your Strands agent

## Connect and Go

```python
from ai_fabric_tools import connect
from ai_fabric_tools.adapters.strands import discover_devices, invoke_device
from strands import Agent
from strands.models.anthropic import AnthropicModel

connect(
    nats_url="nats://your-server:4222",
    credentials="/path/to/agent.creds.json",
)

agent = Agent(
    model=AnthropicModel(model_id="us.anthropic.claude-sonnet-4-5-20250929-v1:0"),
    tools=[discover_devices, invoke_device],
    system_prompt="You are an assistant that monitors and controls IoT devices.",
)

agent("What devices are online? Check the battery level of any robots you find.")
```

The agent will call `discover_devices` to see what's available, then call `invoke_device` to check battery levels. You didn't write that logic — the model figured it out from the tool descriptions and the device function schemas.

### Connection Options

You can configure the connection explicitly or through environment variables:

=== "Environment Variables"

    ```bash
    export NATS_URL=nats://your-server:4222
    export NATS_CREDENTIALS_FILE=/path/to/agent.creds.json
    export TENANT=default
    ```

    ```python
    from ai_fabric_tools import connect
    connect()  # picks up env vars automatically
    ```

=== "Explicit"

    ```python
    from ai_fabric_tools import connect
    connect(
        nats_url="nats://your-server:4222",
        credentials="/path/to/agent.creds.json",
        zone="warehouse-east",
    )
    ```

The connection is lazy — it opens on the first tool call, not at import time.

## Tool Reference

### discover_devices

See what's on the network.

```python
from ai_fabric_tools.adapters.strands import discover_devices
```

| Parameter | Type | Description | Default |
|-----------|------|-------------|---------|
| `device_type` | `str \| None` | Filter by type, e.g. `"robot"`, `"camera"` (fuzzy match) | `None` (all) |
| `refresh` | `bool` | Bypass cache and query registry directly | `False` |

**Example response:**

```python
[
    {
        "device_id": "robot-001",
        "device_type": "cleaning_robot",
        "location": "warehouse-B",
        "status": {"availability": "available"},
        "functions": [
            {
                "name": "start_cleaning",
                "description": "Start cleaning a specified zone.",
                "parameters": {
                    "type": "object",
                    "properties": {
                        "zone": {"type": "string", "description": "Target zone ID"}
                    }
                }
            },
            {
                "name": "get_battery_level",
                "description": "Returns current battery percentage.",
                "parameters": {}
            }
        ],
        "events": ["cleaning_complete", "battery_low"]
    },
    {
        "device_id": "camera-002",
        "device_type": "vision_camera",
        "location": "entrance",
        "status": {"availability": "available"},
        "functions": [
            {
                "name": "capture_image",
                "description": "Capture an image from the camera.",
                "parameters": {
                    "type": "object",
                    "properties": {
                        "resolution": {"type": "string", "default": "1080p"}
                    }
                }
            }
        ],
        "events": ["motion_detected"]
    }
]
```

The model reads these function schemas and knows exactly what it can call and with what parameters. No manual tool definitions needed on your side.

!!! tip "Inject Device Context into the System Prompt"
    For faster first responses, discover devices at startup and put the result in the system prompt. The model starts the conversation already knowing what's available.

    ```python
    from ai_fabric_tools import connect, discover_devices as discover

    connect()
    devices = discover()

    agent = Agent(
        tools=[discover_devices, invoke_device],
        system_prompt=f"You control IoT devices. Available devices:\n\n{devices}",
    )
    ```

    Or let the agent call `discover_devices` itself when it needs to — both patterns work.

### invoke_device

Call a function on a device.

```python
from ai_fabric_tools.adapters.strands import invoke_device
```

| Parameter | Type | Description | Default |
|-----------|------|-------------|---------|
| `device_id` | `str` | Target device, e.g. `"robot-001"` | *required* |
| `function` | `str` | Function to call, e.g. `"start_cleaning"` | *required* |
| `params` | `dict \| None` | Function parameters | `None` |
| `llm_reasoning` | `str \| None` | Why the agent is making this call — logged for observability | `None` |

**Example responses:**

```python
# Success
{"success": True, "result": {"zone": "B-7", "status": "cleaning_in_progress"}}

# Failure
{"success": False, "error": "Device robot-001 is offline"}
```

The `llm_reasoning` field is not sent to the device. It's logged on the agent side so you can trace back *why* the model made a particular call.

### invoke_device_with_fallback

Same as `invoke_device`, but tries a list of devices in order. Useful when the agent identifies multiple candidates and wants automatic failover.

```python
from ai_fabric_tools.adapters.strands import invoke_device_with_fallback
```

| Parameter | Type | Description | Default |
|-----------|------|-------------|---------|
| `device_ids` | `list[str]` | Devices to try, in preference order | *required* |
| `function` | `str` | Function to call | *required* |
| `params` | `dict \| None` | Function parameters | `None` |
| `llm_reasoning` | `str \| None` | Decision rationale | `None` |

**Example responses:**

```python
# Second device succeeded
{"success": True, "device_id": "robot-002", "result": {"status": "cleaning"}}

# All failed
{
    "success": False,
    "error": "All devices failed",
    "failed_devices": [
        {"device_id": "robot-001", "error": "offline"},
        {"device_id": "robot-002", "error": "timeout"}
    ]
}
```

### get_device_status

Check a single device's current state before acting on it.

```python
from ai_fabric_tools.adapters.strands import get_device_status
```

| Parameter | Type | Description | Default |
|-----------|------|-------------|---------|
| `device_id` | `str` | Device to query | *required* |

**Example response:**

```python
{
    "device_id": "robot-001",
    "device_type": "cleaning_robot",
    "location": "warehouse-B",
    "status": {"availability": "available"},
    "functions": ["start_cleaning", "stop_cleaning", "get_battery_level"]
}
```

## Example: Agent That Reacts to Device Events

A complete working example. Devices emit events (sensor readings, alerts, status changes). The agent subscribes, batches them, and reasons about what to do.

```python
import asyncio
from concurrent.futures import ThreadPoolExecutor

from ai_fabric_tools import connect, get_connection
from ai_fabric_tools import discover_devices as discover
from ai_fabric_tools.adapters.strands import (
    discover_devices,
    invoke_device,
    invoke_device_with_fallback,
    get_device_status,
)
from strands import Agent
from strands.models.anthropic import AnthropicModel


def build_prompt(events: list[dict]) -> str:
    lines = []
    for e in events:
        device_id = e.get("device_id", "?")
        event_name = e.get("event_name", "?")
        payload = {k: v for k, v in e.items()
                   if k not in ("device_id", "event_name", "event_id", "ts")}
        lines.append(f"  {device_id}::{event_name} → {payload}")

    return (
        f"{len(events)} new device events:\n"
        + "\n".join(lines)
        + "\n\nAnalyze these events and take action if needed."
    )


async def main():
    connect()
    conn = get_connection()

    devices = discover()
    agent = Agent(
        model=AnthropicModel(model_id="us.anthropic.claude-sonnet-4-5-20250929-v1:0"),
        tools=[discover_devices, invoke_device, invoke_device_with_fallback, get_device_status],
        system_prompt=f"You monitor IoT devices. Available:\n\n{devices}",
    )

    executor = ThreadPoolExecutor(max_workers=1)

    async for batch in conn.subscribe_events(batch_window=12.0):
        if not batch:
            continue

        prompt = build_prompt(batch)
        print(f"\n--- {len(batch)} events received ---")

        loop = asyncio.get_running_loop()
        response = await loop.run_in_executor(
            executor, lambda p=prompt: str(agent(p))
        )
        print(f"Agent: {response}")


if __name__ == "__main__":
    asyncio.run(main())
```

**What happens:**

1. Agent connects to the NATS mesh and discovers all devices
2. Subscribes to every device event on the network
3. Events accumulate over a 12-second window
4. Each batch becomes a prompt: "Here are 5 new events. Analyze and act."
5. The agent reads the events, decides what to do, and calls `invoke_device` if action is needed

!!! note "Sync/Async Bridge"
    Strands agents are synchronous. NATS event subscriptions are async. The `ThreadPoolExecutor` bridges the two — the agent runs in a worker thread while the event loop handles NATS I/O.

## Access Control

Credentials define what your agent can see and do. A credential scoped to `warehouse-east` devices will:

- **`discover_devices`** — only returns warehouse-east devices
- **`invoke_device`** — calls to out-of-scope devices return an authorization error, not a silent failure
- **`get_device_status`** — same scoping

This is enforced at the NATS layer, not in the SDK. The tools don't filter — the messaging infrastructure does.

## How It Works Under the Hood

The Strands adapter is thin. Each tool is a plain Python function wrapped with `@strands.tool`:

```python
from strands import tool as strands_tool
from ai_fabric_tools.tools import discover_devices as _discover_devices

discover_devices = strands_tool(_discover_devices)
```

The core functions use `nats-py` to send JSON-RPC requests over NATS subjects:

- Discovery: `fabric.{zone}.discovery` → registry responds with device list
- Invoke: `fabric.{zone}.{device_id}.cmd` → device executes function and responds
- Events: `fabric.{zone}.{device_id}.event.{event_name}` → device publishes, agent subscribes

No HTTP, no REST, no WebSocket. Direct NATS pub/sub with request-reply for RPCs.

## What These Tools Don't Do

The tools are stateless primitives. They do not:

- Subscribe to events (use `conn.subscribe_events()` directly — see example above)
- Maintain conversation or device state between calls
- Batch multiple device calls into one request
- Run any reasoning or decision logic

All orchestration lives in your agent. Fabric gives your agent reach into the physical world. Your agent decides what to do with it.
