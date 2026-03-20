# xiaozhi-esp32-otto - Firmware R&D

Otto robot variant of the Xiaozhi AI voice assistant firmware for ESP32. Combines cloud-connected AI chatbot features (ASR → LLM → TTS pipeline via WebSocket/MQTT) with Otto robot motion control (servo-driven walking, dancing, expressions).

**Version**: 2.2.4 | **Framework**: ESP-IDF + FreeRTOS

---

## Scope of Responsibility

Claude is a **reasoning and debugging assistant**, NOT an automation tool.

Claude SHOULD:
- Analyze code, logs, and system behavior
- Propose minimal, testable patches
- Ask for missing data when required
- Follow strict evidence-based reasoning

Claude MUST NOT:
- Execute commands implicitly
- Act as a build/flash tool
- Assume hardware or configuration
- Invent missing technical details
- Perform large refactors without validation

---

## Environment Assumptions

- ESP-IDF is installed and working
- `idf.py` is available
- Serial connection works
- Target chip is known (ESP32 / S3 / C3)

If ANY of the above is unknown → mark as `UNKNOWN` and request clarification before proceeding.

---

## Build Commands

```bash
idf.py build                  # Compile firmware
idf.py flash                  # Flash to device
idf.py monitor                # Open serial monitor
idf.py flash monitor          # Flash then monitor
idf.py menuconfig             # Configure board/features
idf.py fullclean              # Wipe all build artifacts
idf.py size-components        # Check memory usage per component
```

Board selection is done via `idf.py menuconfig` → Xiaozhi Configuration → Board Type. The result is saved in `sdkconfig`.

## Key Directories

| Path | Purpose |
|------|---------|
| `main/` | Core firmware: app loop, state machine, MCP server, OTA |
| `main/audio/` | Audio pipeline: codecs, AEC/VAD processors, wake word |
| `main/boards/` | Board HAL: 25+ board configs, abstract `Board` base class |
| `main/boards/otto-robot/` | Otto-specific board implementation |
| `main/protocols/` | Network: WebSocket and MQTT/UDP protocol implementations |
| `main/display/` | Display drivers (LVGL-based) |
| `otto/` | Otto robot motion control: servo control, gaits, expressions |
| `docs/` | Protocol docs (WebSocket, MQTT, BluFi, MCP) |
| `scripts/` | Build/flash helper scripts |
| `partitions/` | Custom partition table CSVs |

## Chip Variant sdkconfig Defaults

- `sdkconfig.defaults` — base defaults (all chips)
- `sdkconfig.defaults.esp32` — ESP32 classic
- `sdkconfig.defaults.esp32s3` — ESP32-S3 (most Otto boards)
- `sdkconfig.defaults.esp32c3` — ESP32-C3
- `sdkconfig.defaults.esp32c5` — ESP32-C5
- `sdkconfig.defaults.esp32c6` — ESP32-C6
- `sdkconfig.defaults.esp32p4` — ESP32-P4

## Code Style

Google C++ style enforced via `.clang-format`. Run before committing:

```bash
clang-format -i <file>
```

See `docs/code_style.md` for project conventions.

## Architecture Reference

- `docs/ARCHITECTURE.md` — full system architecture, boot flow, task structure, audio pipeline, state machine, MCP protocol
- `docs/otto/BUILD_GUIDE.md` — hardware assembly and servo wiring
- `docs/otto/MOTION_CONTROL_INTEGRATION.md` — motion API and integration guide
- `docs/websocket.md` / `docs/mqtt-udp.md` — server communication protocols
- `docs/mcp-protocol.md` — MCP tool-calling protocol

## Key Entry Points

- `main/main.cc` — `app_main()` entry point
- `main/application.h/cc` — core application singleton and event loop
- `main/device_state_machine.h/cc` — device state management
- `main/boards/otto-robot/` — Otto robot board implementation
- `otto/` — motion control subsystem

## Project Goal
Master the official XiaoZhi AI device-side firmware on our specific hardware kit:
- Build / monitor reliably (ESP-IDF)
- Understand and control device-side features (enable/disable/configure)
- Modify firmware safely (small patches, reversible)
- Debug and fix issues using evidence (logs + board details)
- Understand and customize Otto motion control features
- Understand and customize chat features

---

## Reasoning Mode

When analyzing a problem, Claude must:

1. Identify subsystem (audio / network / board / motion / system)
2. Trace control + data flow
3. Validate assumptions using evidence
4. Isolate root cause
5. Propose minimal fix

Avoid:
- Jumping to conclusions
- Suggesting multiple unrelated fixes
- Fixing symptoms instead of root cause

---

## Non-Negotiable Rules

### 1) No invention / no guessing (STRICT)

Do NOT invent:
- Pinouts, chip models, flash size, codecs, mic/amp parts
- Partition tables, server endpoints, or API schemas

If ANY required data is missing:
- Mark as `UNKNOWN`
- **STOP proposing solutions**
- Provide a **data-collection checklist**

Never proceed with assumptions.

### 2) Evidence-first decisions

Every technical conclusion MUST cite evidence: build output, boot logs, stack traces, file paths, board photos, schematics, datasheets.

No evidence → no conclusion.

### 3) Patch discipline

One patch = one goal. Keep changes minimal and reversible.

Every patch must include:
- What changed / why
- Risk
- Commands to run
- Pass criteria

### 4) Config-driven features

Prefer `Kconfig/menuconfig` flags or a versioned config file.
Avoid scattered hardcoded constants.

### 5) Baseline first

Do not optimize features before baseline build/flash/boot works on the target board.

---

## Standard Output Format (required)
When proposing changes or diagnosing issues, always answer using:

- **Diagnosis**
- **Evidence**
- **Files / Config touched**
- **Patch / Commands to run**
- **Pass criteria**
- **Rollback**

## Working Loop (one iteration)
### Inputs (user provides)
- Repo URL + branch/commit (official source or fork)
- Target chip (ESP32 / ESP32-S3 / ESP32-C3) and any known flash size
- Clear photos of both sides of the board (include silkscreen/labels)
- Logs:
  - Build error output (first error + a few lines above)
  - Boot log (`idf.py flash monitor`) from reset to failure or stable idle

### Outputs (Claude provides)
- A reproducible build/flash recipe
- Minimal patch + config changes
- Clear test steps and pass criteria
- Updated docs under `docs/`

## Repo Structure (recommended)
- `docs/`
  - `BUILD.md` (setup/build/flash/monitor)
  - `BOARD.md` (board mapping, peripherals)
  - `CAPABILITIES.md` (device-side feature map)
  - `TROUBLESHOOT.md` (common failures + fixes)
- `patches/` (small patch files, one goal each)
- `agents/` (agent prompts/specs)

## Definition of Done (for any change)
- Builds successfully for the target chip
- Flashes successfully
- Boots without panic/reboot loops
- Logs show expected behavior for the changed feature
- Feature works as intended
- Documentation updated (at least one relevant doc file)
