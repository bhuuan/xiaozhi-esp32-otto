# Xiaozhi ESP32 Firmware — System Architecture & Firmware Flow

**Version**: 2.2.4 | **Platform**: ESP32/ESP32-S3 | **Framework**: ESP-IDF + FreeRTOS

---

## Table of Contents

1. [Directory Structure](#1-directory-structure)
2. [System Overview](#2-system-overview)
3. [Boot & Initialization Flow](#3-boot--initialization-flow)
4. [Concurrent Tasks Architecture](#4-concurrent-tasks-architecture)
5. [Audio Pipeline](#5-audio-pipeline)
6. [AI / LLM Integration](#6-ai--llm-integration)
7. [State Machine](#7-state-machine)
8. [Network Protocols](#8-network-protocols)
9. [Hardware Abstraction Layer](#9-hardware-abstraction-layer)
10. [MCP Server (AI Tool Calling)](#10-mcp-server-ai-tool-calling)
11. [OTA & Activation](#11-ota--activation)
12. [Key Interface Summary](#12-key-interface-summary)
13. [End-to-End Data Flow](#13-end-to-end-data-flow)

---

## 1. Directory Structure

```
xiaozhi-esp32-otto/
├── main/
│   ├── main.cc                    # Entry point (app_main)
│   ├── application.h/cc           # Core application singleton & event loop
│   ├── device_state_machine.h/cc  # Device state management
│   ├── mcp_server.h/cc            # MCP protocol server (AI tool calls)
│   ├── ota.h/cc                   # OTA firmware/asset update
│   ├── settings.h                 # NVS key-value storage wrapper
│   ├── system_info.h              # Device UUID, version, memory stats
│   │
│   ├── audio/                     # Audio subsystem
│   │   ├── audio_codec.h/cc       # Abstract codec interface + I2S base
│   │   ├── audio_service.h/cc     # Audio pipeline orchestrator
│   │   ├── audio_processor.h      # Abstract processor (AEC, VAD, noise)
│   │   ├── wake_word.h            # Abstract wake word detector
│   │   ├── codecs/                # ES8311, ES8374, ES8388, ES8389, Box, Dummy
│   │   ├── processors/            # AfeAudioProcessor, NoAudioProcessor
│   │   ├── wake_words/            # EspWakeWord, AfeWakeWord
│   │   └── demuxer/               # OGG demuxer for server audio
│   │
│   ├── protocols/                 # Network communication
│   │   ├── protocol.h             # Abstract Protocol base class
│   │   ├── mqtt_protocol.h/cc     # MQTT + UDP audio transport
│   │   └── websocket_protocol.h/cc # WebSocket binary protocol
│   │
│   ├── boards/                    # Board-specific hardware configs (25+ boards)
│   │   └── common/
│   │       ├── board.h/cc         # Abstract Board base class
│   │       ├── wifi_board.cc      # WiFi + BluFi provisioning
│   │       ├── ml307_board.cc     # ML307 4G cellular modem
│   │       ├── nt26_board.cc      # NT26 cellular modem
│   │       └── [battery, button, knob, camera drivers]
│   │
│   ├── display/                   # Display drivers
│   │   ├── display.h              # Abstract Display interface
│   │   ├── lcd_display.h/cc       # SPI/I2C LCD (ST7789, SSD1315, etc.)
│   │   ├── oled_display.h/cc      # I2C OLED (SSD1306, SH1106)
│   │   ├── lvgl_display/          # LVGL 9.2 full GUI with emoji + GIF
│   │   └── emote_display.h/cc     # Emoji/expression renderer
│   │
│   ├── led/                       # LED drivers (GPIO, WS2812B strip)
│   └── assets/                    # Localized strings, fonts, images
│
├── Kconfig.projbuild              # User-configurable build options
├── sdkconfig.defaults*            # ESP-IDF default settings
└── CMakeLists.txt                 # Build system
```

---

## 2. System Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        APPLICATION LAYER                        │
│  Application (event loop)  ←→  DeviceStateMachine               │
│  McpServer (AI tool calls) ←→  OtaClient (activation/updates)  │
└────────────┬──────────────────────────────┬────────────────────┘
             │                              │
┌────────────▼──────────┐    ┌─────────────▼──────────────────┐
│   AUDIO SUBSYSTEM     │    │       NETWORK SUBSYSTEM        │
│                       │    │                                │
│  AudioService         │    │  Protocol (abstract)           │
│  ├─ AudioCodec (I2S)  │◄──►│  ├─ MqttProtocol              │
│  ├─ AudioProcessor    │    │  └─ WebsocketProtocol          │
│  │  └─ AFE/AEC/VAD   │    │                                │
│  ├─ WakeWordDetector  │    │  NetworkInterface (abstract)   │
│  └─ Opus Codec        │    │  ├─ WiFiBoard (+ BluFi)        │
└───────────────────────┘    │  ├─ ML307Board (4G)            │
                             │  └─ NT26Board (4G)             │
                             └────────────────────────────────┘
             │
┌────────────▼──────────────────────────────────────────────────┐
│                     HARDWARE ABSTRACTION                       │
│  Board (abstract)                                             │
│  ├─ AudioCodec   (ES8311/ES8374/ES8388/ES8389)                │
│  ├─ Display      (LCD/OLED/LVGL/Emote/None)                  │
│  ├─ Led          (GPIO/WS2812B)                               │
│  ├─ Button/Knob  (GPIO input)                                 │
│  └─ PowerManager (AXP2101/SY6970/Battery ADC)                │
└───────────────────────────────────────────────────────────────┘
```

---

## 3. Boot & Initialization Flow

### Entry Point — `main.cc`

```cpp
extern "C" void app_main(void) {
    nvs_flash_init();               // NVS for WiFi credentials / settings
    auto& app = Application::GetInstance();
    app.Initialize();               // Setup all subsystems
    app.Run();                      // Enter event loop (never returns)
}
```

### `Application::Initialize()` Sequence

```
1. SetDeviceState(kDeviceStateStarting)
2. Board::GetInstance()             ← Board factory (25+ implementations)
   └─ Creates: AudioCodec, Display, LED, Network
3. SetupUI()
   └─ Display::SetStatus("Starting...")
4. AudioService::Initialize()
   ├─ AudioCodec::Initialize()      ← I2C codec init + I2S DMA setup
   ├─ AudioProcessor::Initialize()  ← AFE setup (AEC model, VAD, noise)
   └─ WakeWord::Initialize()        ← Load wake word model
5. Register AudioService callbacks:
   ├─ on_send_queue_available → signal MAIN_EVENT_SEND_AUDIO
   ├─ on_wake_word_detected   → signal MAIN_EVENT_WAKE_WORD_DETECTED
   └─ on_vad_change           → signal MAIN_EVENT_VAD_CHANGE
6. AddStateChangeListener()         ← LED, Display, AudioService react
7. StartClockTimer()                ← 1 Hz tick → MAIN_EVENT_CLOCK_TICK
8. McpServer::Initialize()          ← Register built-in + user tools
9. Board::SetNetworkEventCallback() ← Capture network events
10. Board::StartNetwork()           ← Async — returns immediately
    └─ WiFi/cellular starts in background, signals MAIN_EVENT_NETWORK_CONNECTED
```

---

## 4. Concurrent Tasks Architecture

The firmware runs multiple FreeRTOS tasks that communicate via event groups and queues:

```
┌──────────────────────────────────────────────────────────────┐
│                      FREERTOS TASKS                          │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  MAIN TASK (priority 10)                            │    │
│  │  application.cc::Run()                              │    │
│  │  - Handles all system events via xEventGroupWaitBits│    │
│  │  - Owns protocol send/receive dispatch              │    │
│  │  - Drives state machine transitions                 │    │
│  └──────────────────────┬──────────────────────────────┘    │
│                         │ EventGroup signals                 │
│  ┌──────────────────────▼──────────────────────────────┐    │
│  │  AUDIO INPUT TASK                                   │    │
│  │  audio_service.cc::AudioInputTask()                 │    │
│  │  - Reads I2S DMA → AudioCodec::Read()               │    │
│  │  - Feeds AudioProcessor (AFE/AEC)                   │    │
│  │  - Feeds WakeWordDetector                           │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  OPUS CODEC TASK                                    │    │
│  │  audio_service.cc::OpusCodecTask()                  │    │
│  │  - Encodes PCM → Opus (60ms frames, VBR, DTX)       │    │
│  │  - Decodes Opus → PCM (server TTS audio)            │    │
│  │  - Manages audio testing mode                       │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  AUDIO OUTPUT TASK                                  │    │
│  │  audio_service.cc::AudioOutputTask()                │    │
│  │  - Pulls from audio_playback_queue_                 │    │
│  │  - Writes PCM → AudioCodec::Write() → I2S DMA      │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  ACTIVATION TASK (temporary, background)            │    │
│  │  application.cc::ActivationTask()                   │    │
│  │  - OTA activation handshake with server             │    │
│  │  - Asset download (emoji, fonts, voices)            │    │
│  │  - Protocol initialization (MQTT/WebSocket)         │    │
│  │  - Signals MAIN_EVENT_ACTIVATION_DONE when complete │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  NETWORK TASK (board-managed)                       │    │
│  │  WiFi / Cellular modem driver tasks                 │    │
│  │  - Manages connection lifecycle                     │    │
│  │  - Signals network events via callback              │    │
│  └─────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────┘
```

### Inter-Task Communication

| Queue / Signal | Producer | Consumer | Capacity |
|---|---|---|---|
| `audio_encode_queue_` | AudioInputTask | OpusCodecTask | 2 packets |
| `audio_decode_queue_` | Protocol (incoming) | OpusCodecTask | 40 packets (~2.4s) |
| `audio_send_queue_` | OpusCodecTask | Main Task | 40 packets |
| `audio_playback_queue_` | OpusCodecTask | AudioOutputTask | 2 packets |
| `EventGroup MAIN_EVENT_*` | All tasks | Main Task | Bitfield |

---

## 5. Audio Pipeline

### Input Pipeline (Microphone → Server)

```
MIC
 │
 ▼ I2S DMA (6 descriptors, 240 samples/frame)
AudioCodec::Read(int16_t* buf, samples)
 │
 ▼ [AudioInputTask]
AudioProcessor::Feed(buf)            ← AFE: AEC reference signal injection
 │
 ├──► WakeWordDetector::Feed(buf)    ← Wake word model inference
 │       │
 │       └── on_wake_word_detected("小智") ──► MAIN_EVENT_WAKE_WORD_DETECTED
 │
 └──► on_output callback (VAD-gated processed PCM)
         │
         ▼
   audio_encode_queue_
         │
         ▼ [OpusCodecTask]
   OpusEncoder (60ms, 16kHz, VBR, DTX)
         │
         ▼
   audio_send_queue_
         │
         ▼ MAIN_EVENT_SEND_AUDIO
   protocol_->SendAudio(packet)      ← MQTT or WebSocket
         │
         ▼
   [SERVER: ASR → LLM]
```

### Output Pipeline (Server TTS → Speaker)

```
[SERVER: LLM → TTS]
 │
 ▼ MQTT / WebSocket binary frame
protocol_->OnIncomingAudio(packet)
 │
 ▼
audio_decode_queue_
 │
 ▼ [OpusCodecTask]
OpusDecoder → PCM (output_sample_rate)
 │
 ▼ Resampler (if server rate ≠ codec rate)
 │
 ▼
audio_playback_queue_
 │
 ▼ [AudioOutputTask]
AudioCodec::Write(int16_t* buf, samples)
 │
 ▼ I2S DMA → PA Amplifier → SPEAKER
```

### Opus Configuration

| Parameter | Value |
|---|---|
| Frame duration | 60 ms |
| Encoder sample rate | 16000 Hz |
| Bitrate | Auto (VBR) |
| DTX (Discontinuous Tx) | Enabled |
| Channels | Mono |

### Audio Codecs

| Codec | Interface | Typical Use |
|---|---|---|
| ES8388 | I2C + I2S | Most development boards |
| ES8311 | I2C + I2S | Low-power designs |
| ES8374 | I2C + I2S | Mid-range boards |
| ES8389 | I2C + I2S | High-performance |
| Box Audio | Custom | ESP32-S3 Box |
| Dummy | None | Headless / testing |

---

## 6. AI / LLM Integration

The device does **not** run an LLM locally. It streams audio to a cloud server which handles ASR → LLM → TTS, then streams the response back.

### Activation & Server Handshake

```
Device boots → Network connected
    ↓
OtaClient::CheckVersion()           HTTP GET /ota
    ├─ Server returns:
    │   ├─ protocol: "mqtt" | "websocket"
    │   ├─ endpoint / credentials
    │   ├─ firmware_url (if update available)
    │   └─ activation_code (first-run only)
    ↓
[If update available] → OTA download → reboot
    ↓
Protocol::Initialize()              ← MQTT or WebSocket
    ↓
Protocol::SendHello()               ← Device identification
    {
      "type": "hello",
      "version": 3,
      "audio_params": { "format": "opus", "sample_rate": 16000 },
      "features": { "mcp": true },
      "device_id": "<uuid>"
    }
    ↓
Server → SendServerHello()
    {
      "type": "hello",
      "session_id": "...",
      "audio_params": { "sample_rate": 24000 }
    }
    ↓
Device ready → kDeviceStateIdle
```

### Conversation Message Types (JSON)

| `type` | Direction | Description |
|---|---|---|
| `hello` | Both | Session handshake |
| `listen` | Device→Server | Start/stop listening notification |
| `stt` | Server→Device | Speech-to-text result |
| `llm` | Server→Device | LLM response chunk + emotion |
| `tts` | Server→Device | TTS state (`start`/`stop`) |
| `mcp` | Both | MCP tool call request/response |
| `abort` | Both | Cancel current interaction |
| `error` | Server→Device | Error notification |

### Example JSON Exchange

```json
// Server sends STT result
{"type": "stt", "text": "今天天气怎么样"}

// Server sends LLM emotion
{"type": "llm", "emotion": "thinking"}

// Server sends TTS state
{"type": "tts", "state": "start"}
// ... Opus audio binary frames arrive ...
{"type": "tts", "state": "stop"}
```

---

## 7. State Machine

**File**: `device_state_machine.h/cc`

### States

| State | Description |
|---|---|
| `kDeviceStateUnknown` | Initial / undefined |
| `kDeviceStateStarting` | Boot initialization |
| `kDeviceStateWifiConfiguring` | BluFi provisioning (no credentials) |
| `kDeviceStateIdle` | Connected, ready for wake word |
| `kDeviceStateConnecting` | Opening audio channel to server |
| `kDeviceStateListening` | Recording and streaming user speech |
| `kDeviceStateSpeaking` | Playing back server TTS response |
| `kDeviceStateUpgrading` | Downloading OTA update |
| `kDeviceStateActivating` | First-run activation |
| `kDeviceStateAudioTesting` | Hardware audio test mode |
| `kDeviceStateFatalError` | Unrecoverable error (terminal) |

### State Transition Diagram

```
                    ┌─────────────────────┐
              ┌────►│  WifiConfiguring     │
              │     └──────────┬──────────┘
              │                │ credentials saved
              │                ▼
  [boot] ────►│  Starting ────►│  Activating
              │                │       │
              │         OTA update?    │ done
              │            ↓           ▼
              │         Upgrading    Idle ◄──────────────────┐
              │            │           │                      │
              │            └───────────┤ wake word / button   │
              │                        ▼                      │
              │                    Connecting                 │
              │                        │                      │
              │                        ▼                      │
              │                    Listening ─── VAD silence ─►Speaking
              │                        │                      │
              │                        └──── abort ──────────►┘
              │
              └───── FatalError (no exit)
```

### Main Event Loop

The `Application::Run()` loop handles these events via FreeRTOS EventGroup:

| Event | Handler |
|---|---|
| `MAIN_EVENT_NETWORK_CONNECTED` | Start activation task |
| `MAIN_EVENT_NETWORK_DISCONNECTED` | Close audio channel |
| `MAIN_EVENT_WAKE_WORD_DETECTED` | Transition → Listening |
| `MAIN_EVENT_VAD_CHANGE` | Update VAD state, end-of-speech detection |
| `MAIN_EVENT_SEND_AUDIO` | Flush audio_send_queue_ to protocol |
| `MAIN_EVENT_STATE_CHANGED` | Notify LED/Display/AudioService |
| `MAIN_EVENT_TOGGLE_CHAT` | Toggle Idle ↔ Listening |
| `MAIN_EVENT_START_LISTENING` | Force start listening |
| `MAIN_EVENT_STOP_LISTENING` | Force stop listening |
| `MAIN_EVENT_ACTIVATION_DONE` | Complete activation, → Idle |
| `MAIN_EVENT_CLOCK_TICK` | Update status bar time |
| `MAIN_EVENT_ERROR` | Show error, potentially → FatalError |
| `MAIN_EVENT_SCHEDULE` | Run deferred callbacks on main thread |

---

## 8. Network Protocols

### Protocol Abstraction (`protocol.h`)

```cpp
class Protocol {
    // Lifecycle
    virtual bool Start() = 0;
    virtual void SendHello() = 0;
    virtual bool OpenAudioChannel() = 0;
    virtual bool CloseAudioChannel() = 0;
    virtual bool IsAudioChannelOpened() = 0;

    // Audio I/O
    virtual bool SendAudio(std::vector<uint8_t>& data) = 0;
    virtual void OnIncomingAudio(AudioHandler) = 0;

    // JSON I/O
    virtual bool SendJson(cJSON* json) = 0;
    virtual void OnIncomingJson(JsonHandler) = 0;

    // Events
    virtual void OnAudioChannelOpened(Callback) = 0;
    virtual void OnAudioChannelClosed(Callback) = 0;
    virtual void OnNetworkError(ErrorHandler) = 0;
};
```

### Binary Protocol Frame (v2/v3)

```
Byte offset  Field           Size   Description
──────────── ────────────    ──── ─────────────────────────────
0            version         2    Protocol version (2 or 3)
2            type            2    0=Opus audio, 1=JSON
4            reserved        4    Padding
8            timestamp       4    Microseconds (for server AEC)
12           payload_size    4    Bytes of payload
16           payload         N    Opus frames or JSON text
```

### MQTT Protocol

- **Broker**: Provided by OTA activation response
- **Audio transport**: Publish/subscribe topics or UDP (AES-128 encrypted)
- **Keep-alive**: Ping every 90 seconds
- **Reconnect**: 60 second backoff
- **QoS**: 0 for audio, 1 for control messages

### WebSocket Protocol

- **Transport**: Binary WebSocket frames
- **Frame content**: Binary Protocol v2/v3 packets
- **JSON compression**: Optional gzip for large JSON payloads
- **Reconnect**: Managed by EventGroup synchronization

### Network Interfaces

| Interface | File | Connectivity |
|---|---|---|
| WiFi | `wifi_board.cc` | ESP-IDF WiFi, BluFi provisioning via BLE |
| ML307 4G | `ml307_board.cc` | UART AT-command cellular modem |
| NT26 4G | `nt26_board.cc` | UART AT-command cellular modem |

### Network Events

```
Scanning → Connecting → Connected
                    ↕
              Disconnected

WifiConfigModeEnter / WifiConfigModeExit  (BluFi provisioning)
ModemDetecting / ModemErrorNoSim / ModemErrorRegDenied  (cellular)
```

---

## 9. Hardware Abstraction Layer

### Board Abstraction (`board.h`)

Each of the 25+ supported boards implements:

```cpp
class Board {
    virtual AudioCodec*   CreateAudioCodec() = 0;
    virtual Display*      CreateDisplay() = 0;
    virtual Led*          CreateLed() = 0;
    virtual NetworkInterface* GetNetwork() = 0;

    // Power management
    virtual void SetPowerSaveLevel(PowerSaveLevel level) = 0;
    virtual bool IsVoiceWakeupEnabled() = 0;

    // Optional
    virtual Camera*   CreateCamera()  { return nullptr; }
    virtual Button*   GetButton()     { return nullptr; }
};
```

### AudioCodec I2S Configuration

```
I2S DMA: 6 descriptors × 240 samples
Input:   16 kHz mono, 16-bit PCM
Output:  24 kHz stereo (board-dependent), 16-bit PCM
Control: I2C codec register writes (volume, gain, routing)
PA amp:  GPIO-controlled power amplifier
```

### Display Drivers

| Driver | Protocol | Features |
|---|---|---|
| `LcdDisplay` | SPI / I2C | Color LCD panels (ST7789, ILI9341, etc.) |
| `OledDisplay` | I2C | Monochrome OLED (SSD1306, SH1106) |
| `LvglDisplay` | SPI | LVGL 9.2 GUI, emoji font, GIF, JPEG, PNG |
| `EmoteDisplay` | SPI | Dedicated emoji/expression renderer |
| None | — | Headless operation |

**Display interface:**
```cpp
Display::SetStatus(const char* text)           // Status bar text
Display::ShowNotification(text, duration_ms)   // Temporary popup
Display::SetEmotion(const char* emotion)       // Emoji: "happy", "thinking", etc.
Display::SetChatMessage(role, content)         // Show chat bubble
Display::SetPowerSaveMode(bool)                // Dim / sleep display
```

### LED Drivers

| Driver | Description |
|---|---|
| `SingleLed` | One GPIO LED, blink patterns |
| `GpioLed` | Multi-GPIO LED patterns |
| `CircularStrip` | WS2812B addressable RGB ring |

LEDs automatically react to state machine transitions via `OnStateChanged()`.

### Power Management

| Component | Purpose |
|---|---|
| `AXP2101` | PMIC (ESP32-S3 Box family) |
| `SY6970` | USB charging controller |
| Battery ADC | Voltage monitoring |
| Sleep Timer | Auto-sleep after inactivity |
| `PowerSaveLevel` | `LOW_POWER` / `BALANCED` / `PERFORMANCE` |

---

## 10. MCP Server (AI Tool Calling)

**File**: `mcp_server.h/cc`

The firmware implements a **Model Context Protocol** server, allowing the cloud LLM to call functions on the device during conversation.

### Flow

```
LLM (cloud) decides to call a tool
    ↓
Server sends JSON: {"type": "mcp", "method": "tools/call", "params": {...}}
    ↓
Protocol::OnIncomingJson() → McpServer::HandleRequest()
    ↓
Execute registered tool handler on device
    ↓
McpServer::SendResponse() → protocol_->SendJson()
    ↓
Server continues LLM inference with tool result
```

### Tool Registration

```cpp
// Common tools (visible to AI)
mcp_server_.AddTool("set_volume", "Set speaker volume", {{"value", "integer"}},
    [](auto& args) { SetVolume(args["value"]); return "ok"; });

mcp_server_.AddTool("get_time", "Get current time", {},
    [](auto& args) { return GetCurrentTimeString(); });

// User-only tools (hidden from AI, callable by user code only)
mcp_server_.AddUserTool("factory_reset", ...);
```

---

## 11. OTA & Activation

**File**: `ota.h/cc`

### First-Run Activation

```
1. Device connects to network
2. HTTP GET /ota?device_id=<uuid>&version=<fw_ver>
3. Server responds:
   {
     "activation_code": "XXXX",    // Shown on display for user to enter
     "websocket": { "url": "...", "token": "..." }
     // or
     "mqtt": { "broker": "...", "username": "...", "password": "..." }
   }
4. If activation_code present → display it, wait for user activation
5. Once activated → store credentials in NVS
6. Protocol::Start() with received config
```

### Firmware Update

```
1. OTA check returns firmware_url
2. SetDeviceState(kDeviceStateUpgrading)
3. HTTP download to OTA partition
4. Verify SHA-256 hash
5. esp_ota_set_boot_partition()
6. esp_restart()
```

### Asset Update

Assets (emoji fonts, TTS voices, UI images) can be updated independently:

```
1. HTTP GET /assets/manifest.json
2. Compare with current asset hashes in NVS
3. Download changed assets to SPIFFS / LittleFS
4. Update NVS hash records
```

---

## 12. Key Interface Summary

| Interface | Header | Implementations |
|---|---|---|
| `AudioCodec` | `audio/audio_codec.h` | ES8311, ES8374, ES8388, ES8389, Box, Dummy |
| `AudioProcessor` | `audio/audio_processor.h` | AfeAudioProcessor, NoAudioProcessor |
| `WakeWord` | `audio/wake_word.h` | EspWakeWord, AfeWakeWord |
| `Protocol` | `protocols/protocol.h` | MqttProtocol, WebsocketProtocol |
| `Display` | `display/display.h` | LcdDisplay, OledDisplay, LvglDisplay, EmoteDisplay |
| `Board` | `boards/common/board.h` | 25+ board-specific classes |
| `Led` | `led/` | SingleLed, GpioLed, CircularStrip |

### Design Patterns Used

| Pattern | Where |
|---|---|
| Singleton | `Application`, `Board`, `AudioCodec` via `GetInstance()` |
| Abstract Factory | `Board` creates codec/display/LED/network instances |
| Observer | State machine notifies listeners; audio service uses callbacks |
| Event-Driven | FreeRTOS EventGroup drives main loop |
| Producer-Consumer | Audio queues between tasks |
| Strategy | Pluggable audio processor, wake word, display, protocol |
| Template Method | `Protocol` defines interface; MQTT/WebSocket implement it |

---

## 13. End-to-End Data Flow

### Full Conversation Cycle

```
╔══════════════════════════════════════════════════════════════╗
║                        DEVICE BOOT                          ║
╠══════════════════════════════════════════════════════════════╣
║  app_main()                                                  ║
║    └─ nvs_flash_init()                                       ║
║    └─ Application::Initialize()                             ║
║        ├─ Board, Display, AudioCodec, AudioService          ║
║        └─ StartNetwork() [async]                            ║
║  Application::Run()  ← BLOCKS HERE                          ║
╚══════════════════════════════════════════════════════════════╝
                 │
                 │ Network connects
                 ▼
╔══════════════════════════════════════════════════════════════╗
║                       ACTIVATION                            ║
╠══════════════════════════════════════════════════════════════╣
║  ActivationTask()                                            ║
║    ├─ OTA::CheckVersion()   → HTTP                          ║
║    ├─ OTA::DownloadAssets() → HTTP (if changed)             ║
║    ├─ Protocol::Initialize() → MQTT or WebSocket            ║
║    └─ signal MAIN_EVENT_ACTIVATION_DONE                     ║
║                                                              ║
║  → SetDeviceState(Idle)                                     ║
╚══════════════════════════════════════════════════════════════╝
                 │
                 │ User speaks wake word
                 ▼
╔══════════════════════════════════════════════════════════════╗
║                   WAKE WORD DETECTION                       ║
╠══════════════════════════════════════════════════════════════╣
║  [AudioInputTask] → AudioProcessor → WakeWordDetector       ║
║    └─ on_wake_word_detected("小智")                          ║
║         └─ MAIN_EVENT_WAKE_WORD_DETECTED                    ║
║              └─ SetDeviceState(Connecting)                  ║
║              └─ protocol_->OpenAudioChannel()               ║
║              └─ SetDeviceState(Listening)                   ║
╚══════════════════════════════════════════════════════════════╝
                 │
                 │ User speaks
                 ▼
╔══════════════════════════════════════════════════════════════╗
║                    AUDIO STREAMING                          ║
╠══════════════════════════════════════════════════════════════╣
║  MIC → I2S → AudioProcessor (AEC/VAD)                       ║
║    └─ audio_encode_queue_                                   ║
║         └─ [OpusCodecTask] → Opus encode (60ms frames)     ║
║              └─ audio_send_queue_                           ║
║                   └─ MAIN_EVENT_SEND_AUDIO                  ║
║                        └─ protocol_->SendAudio()            ║
║                             └─ MQTT pub / WebSocket binary  ║
╚══════════════════════════════════════════════════════════════╝
                 │
                 │ VAD silence → end of speech
                 ▼
╔══════════════════════════════════════════════════════════════╗
║                    SERVER PROCESSING                        ║
╠══════════════════════════════════════════════════════════════╣
║  Cloud: ASR (speech → text)                                  ║
║       → LLM (reasoning)                                      ║
║       → TTS (text → speech, Opus-encoded)                   ║
║                                                              ║
║  Server → Device:                                           ║
║    {"type":"stt", "text":"今天天气怎么样"}                     ║
║    {"type":"llm", "emotion":"thinking"}                      ║
║    {"type":"tts", "state":"start"}                          ║
║    [binary Opus frames...]                                   ║
║    {"type":"tts", "state":"stop"}                           ║
╚══════════════════════════════════════════════════════════════╝
                 │
                 │ Server sends audio
                 ▼
╔══════════════════════════════════════════════════════════════╗
║                     AUDIO PLAYBACK                          ║
╠══════════════════════════════════════════════════════════════╣
║  OnIncomingAudio() → audio_decode_queue_                    ║
║    └─ [OpusCodecTask] → Opus decode → PCM                   ║
║         └─ [Resampler if needed]                            ║
║              └─ audio_playback_queue_                       ║
║                   └─ [AudioOutputTask]                      ║
║                        └─ I2S DMA → PA → SPEAKER           ║
║                                                             ║
║  Display: shows emotion emoji, chat text                    ║
║  LED: reacts to Speaking state                              ║
║                                                             ║
║  TTS stop → SetDeviceState(Idle)                            ║
╚══════════════════════════════════════════════════════════════╝
```

---

*Generated for xiaozhi-esp32-otto v2.2.4*
