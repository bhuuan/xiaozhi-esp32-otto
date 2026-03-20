# MCP Protocol IoT Control Usage Guide

> This document explains how to implement IoT control for ESP32 devices using MCP protocol. For detailed protocol flow, see [`mcp-protocol.md`](./mcp-protocol.md).

## Introduction

MCP (Model Context Protocol) is the recommended next-generation IoT control protocol. It uses standard JSON-RPC 2.0 to discover and call "tools" (device functions) between the backend and device, enabling flexible device control.

## Typical Usage Flow

1. Device boots and connects to backend via base protocol (WebSocket/MQTT).
2. Backend initializes MCP session with the `initialize` method.
3. Backend calls `tools/list` to get all tools (capabilities) the device supports, along with their parameter schemas.
4. Backend calls `tools/call` to invoke specific tools and control the device.

For detailed protocol format and flow, see [`mcp-protocol.md`](./mcp-protocol.md).

## Device-Side Tool Registration

Devices register callable tools via `McpServer::AddTool`. Function signature:

```cpp
void AddTool(
    const std::string& name,           // Tool name — unique, hierarchical, e.g. "self.dog.forward"
    const std::string& description,    // Tool description — concise, human/AI-readable
    const PropertyList& properties,    // Input parameter list (may be empty); types: bool, int, string
    std::function<ReturnValue(const PropertyList&)> callback  // Handler called when tool is invoked
);
```

- **name**: unique tool identifier; recommended naming: `"module.feature"` style
- **description**: natural language description for AI/user understanding
- **properties**: parameter list; supports bool, int, string types with range and default values
- **callback**: actual execution logic on invocation; return type can be bool/int/string

## Registration Example (ESP-Hi board)

```cpp
void InitializeTools() {
    auto& mcp_server = McpServer::GetInstance();

    // Example 1: no parameters — move robot forward
    mcp_server.AddTool("self.dog.forward", "Move robot forward", PropertyList(),
        [this](const PropertyList&) -> ReturnValue {
            servo_dog_ctrl_send(DOG_STATE_FORWARD, NULL);
            return true;
        });

    // Example 2: with parameters — set light RGB color
    mcp_server.AddTool("self.light.set_rgb", "Set RGB color", PropertyList({
        Property("r", kPropertyTypeInteger, 0, 255),
        Property("g", kPropertyTypeInteger, 0, 255),
        Property("b", kPropertyTypeInteger, 0, 255)
    }), [this](const PropertyList& properties) -> ReturnValue {
        int r = properties["r"].value<int>();
        int g = properties["g"].value<int>();
        int b = properties["b"].value<int>();
        led_on_ = true;
        SetLedColor(r, g, b);
        return true;
    });
}
```

## Common Tool Call JSON-RPC Examples

### 1. Get tool list
```json
{
  "jsonrpc": "2.0",
  "method": "tools/list",
  "params": { "cursor": "" },
  "id": 1
}
```

### 2. Move chassis forward
```json
{
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "self.chassis.go_forward",
    "arguments": {}
  },
  "id": 2
}
```

### 3. Switch light mode
```json
{
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "self.chassis.switch_light_mode",
    "arguments": { "light_mode": 3 }
  },
  "id": 3
}
```

### 4. Flip camera
```json
{
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "self.camera.set_camera_flipped",
    "arguments": {}
  },
  "id": 4
}
```

## Notes

- Tool names, parameters, and return values are defined by `AddTool` registrations on the device side — treat those as the source of truth.
- All new projects should use MCP protocol for IoT control.
- For advanced protocol usage, see [`mcp-protocol.md`](./mcp-protocol.md).
