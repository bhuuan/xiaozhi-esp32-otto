# Motion Control — Otto Robot

**Project**: xiaozhi-esp32-otto
**Board**: `otto-robot` (`main/boards/otto-robot/`)

---

## Table of Contents

1. [Overview](#1-overview)
2. [Otto Motion Infrastructure](#2-otto-motion-infrastructure)
3. [Servo Driver — The Oscillator Class](#3-servo-driver--the-oscillator-class)
4. [Motion Library — The Otto Class](#4-motion-library--the-otto-class)
5. [OttoController — Task Architecture](#5-ottocontroller--task-architecture)
6. [MCP Tools — AI Motion Commands](#6-mcp-tools--ai-motion-commands)
7. [Custom Servo Sequence Protocol](#7-custom-servo-sequence-protocol)
8. [Hardware Configuration](#8-hardware-configuration)
9. [Servo Calibration (Trim System)](#9-servo-calibration-trim-system)
10. [Architecture Diagram](#10-architecture-diagram)
11. [Quick Reference](#11-quick-reference)

---

## 1. Overview

Motion control is structured as five layers stacked on top of each other:

```
AI (Cloud LLM)
    │ calls MCP tools via JSON
    ▼
OttoController  ──  registers MCP tools, queues actions
    │ xQueueSend → action_queue_ (FreeRTOS queue, 10 items)
    ▼
ActionTask (FreeRTOS, highest priority)  ──  executes blocking moves
    │ calls
    ▼
Otto class  ──  high-level gait and pose library
    │ calls
    ▼
Oscillator class  ──  LEDC PWM, 50 Hz, one instance per servo
    │ LEDC hardware registers
    ▼
Servo motors
```

The AI calls MCP tools. The tool callback immediately enqueues the action and returns. A dedicated FreeRTOS task picks up the action and runs it to completion. The main audio task is never blocked, so speech, wake-word detection, and AI streaming remain fully operational during movement.

---

## 2. Otto Motion Infrastructure

All motion-related source files live in `main/boards/otto-robot/`:

| File | Purpose |
|---|---|
| `oscillator.h/cc` | LEDC PWM servo driver — generates sinusoidal motion |
| `otto_movements.h/cc` | High-level gait library (walk, jump, dance, etc.) |
| `otto_controller.cc` | FreeRTOS action queue + MCP tool registration |
| `otto_robot.cc` | Board subclass — constructs the controller |
| `config.h` | GPIO pin maps for camera and non-camera hardware variants |
| `power_manager.h` | Battery ADC + charge detection integrated with motion |
| `websocket_control_server.cc` | Optional local WebSocket for real-time direct servo control |
| `otto_emoji_display.cc` | Custom GIF/emoji face display reacting to motion state |

---

## 3. Servo Driver — The Oscillator Class

**File**: `main/boards/otto-robot/oscillator.h`

Each servo is driven by one `Oscillator` instance. It uses the ESP32's **LEDC (LED PWM Controller)** hardware to generate a 50 Hz PWM signal. The pulse width maps to servo angle.

### Key Constants

```cpp
#define SERVO_MIN_PULSEWIDTH_US  500    // 500µs  → -90°
#define SERVO_MAX_PULSEWIDTH_US  2500   // 2500µs → +90°
#define SERVO_MIN_DEGREE         -90
#define SERVO_MAX_DEGREE         90
#define SERVO_TIMEBASE_RESOLUTION_HZ  1000000  // 1MHz (1µs per tick)
#define SERVO_TIMEBASE_PERIOD         20000    // 20000 ticks = 20ms = 50Hz
```

### API

```cpp
class Oscillator {
public:
    Oscillator(int trim = 0);

    // Attach to a GPIO pin. rev=true physically mirrors the servo direction.
    void Attach(int pin, bool rev = false);
    void Detach();

    // Oscillation parameters (used by gait algorithms)
    void SetA(unsigned int amplitude);  // Swing range in degrees (0–90)
    void SetO(int offset);              // Center position offset in degrees
    void SetPh(double Ph);              // Initial phase in radians
    void SetT(unsigned int period);     // Full cycle period in milliseconds

    // Direct position control (bypasses oscillation)
    void SetPosition(int position);     // -90 to +90 degrees
    int  GetPosition();

    // Calibration
    void SetTrim(int trim);             // Mechanical offset correction in degrees
    void SetLimiter(int diff_limit);    // Max rate of change, degrees/sec
    void DisableLimiter();

    // Oscillation state
    void Stop();    // Freeze at current position
    void Play();    // Resume sinusoidal output
    void Reset();   // Reset phase accumulator to 0

    // Called in a tight loop to advance phase and update PWM
    void Refresh();
};
```

### Sinusoidal Motion Formula

At each `Refresh()` call the output position is:

```
position = offset + amplitude × sin(phase + phase0) + trim
```

`phase` advances by `2π / (period / sampling_period)` per tick, so one full oscillation takes exactly `period` milliseconds.

### LEDC Channel Allocation

Each `Attach()` call claims one LEDC low-speed channel in order:

```cpp
// oscillator.cc — static counter shared across all instances
static ledc_channel_t next_channel = LEDC_CHANNEL_0;
```

The ESP32-S3 has 8 low-speed channels (0–7). Otto uses 4 (legs/feet only) or 6 (with arms), leaving headroom for backlight PWM and other peripherals.

---

## 4. Motion Library — The Otto Class

**File**: `main/boards/otto-robot/otto_movements.h`

`Otto` owns an array of 6 `Oscillator` instances and provides named motion functions built from sinusoidal patterns and linear interpolation.

### Servo Index Mapping

```cpp
#define LEFT_LEG    0   // Lateral leg swing (inner ↔ outer)
#define RIGHT_LEG   1   // Lateral leg swing
#define LEFT_FOOT   2   // Foot pitch (toe up ↔ toe down)
#define RIGHT_FOOT  3   // Foot pitch
#define LEFT_HAND   4   // Arm swing (down ↔ up) — optional
#define RIGHT_HAND  5   // Arm swing              — optional
#define SERVO_COUNT 6
```

### Servo Angle Convention

The sequence protocol uses **absolute 0–180°**. The oscillator API uses **-90° to +90°** (deviation from centre). The `Otto` class converts between them internally.

| Servo | 0° | 90° | 180° |
|---|---|---|---|
| `ll` left leg | fully outward | centre | fully inward |
| `rl` right leg | fully inward | centre | fully outward |
| `lf` left foot | fully up | horizontal | fully down |
| `rf` right foot | fully down | horizontal | fully up |
| `lh` left hand | fully down | horizontal | fully up |
| `rh` right hand | fully up | horizontal | fully down |

### Initialization

```cpp
// Legs + feet only (4 servos)
otto_.Init(left_leg_pin, right_leg_pin, left_foot_pin, right_foot_pin);

// Full body including arms (6 servos)
otto_.Init(left_leg_pin, right_leg_pin, left_foot_pin, right_foot_pin,
           left_hand_pin, right_hand_pin);
```

Pins come from `HardwareConfig` in `config.h` (auto-detected camera vs non-camera variant).

### Predefined Motion Functions

#### Locomotion

```cpp
void Walk(float steps=4, int period=1000, int dir=FORWARD, int amount=0);
    // dir: FORWARD(1) or BACKWARD(-1)
    // amount: arm swing amplitude (0 = arms stationary)

void Turn(float steps=4, int period=2000, int dir=LEFT, int amount=0);
    // dir: LEFT(1) or RIGHT(-1)

void Jump(float steps=1, int period=2000);
```

#### Body Motion

```cpp
void Swing(float steps=1, int period=1000, int height=20);
void TiptoeSwing(float steps=1, int period=900, int height=20);
void UpDown(float steps=1, int period=1000, int height=20);
void Jitter(float steps=1, int period=500, int height=20);
void AscendingTurn(float steps=1, int period=900, int height=20);
void Moonwalker(float steps=1, int period=900, int height=20, int dir=LEFT);
void Crusaito(float steps=1, int period=900, int height=20, int dir=FORWARD);
void Flapping(float steps=1, int period=1000, int height=20, int dir=FORWARD);
void WhirlwindLeg(float steps=1, int period=300, int amplitude=30);
void Bend(int steps=1, int period=1400, int dir=LEFT);
void ShakeLeg(int steps=1, int period=2000, int dir=RIGHT);
void Sit();
```

#### Arm / Hand Motions (requires hand servos)

```cpp
void HandsUp(int period=1000, int dir=0);    // dir: LEFT(1), RIGHT(-1), BOTH(0)
void HandsDown(int period=1000, int dir=0);
void HandWave(int dir=LEFT);
void Windmill(float steps=10, int period=500, int amplitude=90);
void Takeoff(float steps=5, int period=300, int amplitude=40);
void Fitness(float steps=5, int period=1000, int amplitude=25);
void Greeting(int dir=LEFT, float steps=5);
void Shy(int dir=LEFT, float steps=5);
void RadioCalisthenics();   // Full body exercise sequence
void MagicCircle();         // Spinning arm dance
void Showcase();            // Demo: chains several motions back-to-back
```

#### Utility

```cpp
void Home(bool hands_down=true);    // Return all servos to rest
void SetTrims(int ll, int rl, int lf, int rf, int lh=0, int rh=0);
void EnableServoLimit(int speed_limit_degree_per_sec=240);
void DisableServoLimit();
```

### Low-Level Motion Primitives

These are the two building blocks used by all higher-level motion functions:

```cpp
// Linearly interpolate all servos from current positions to servo_target[]
// time_ms: total transition duration
void MoveServos(int time_ms, int servo_target[SERVO_COUNT]);

// Run sinusoidal oscillation for `cycle` full periods
// All arrays are per-servo (indexed LEFT_LEG … RIGHT_HAND)
void OscillateServos(int amplitude[], int offset[], int period,
                     double phase_diff[], float cycle);

// Same as OscillateServos but center_angle[] uses absolute 0–180° values
void Execute2(int amplitude[], int center_angle[], int period,
              double phase_diff[], float steps);

// Move a single servo directly
void MoveSingle(int position, int servo_number);
```

---

## 5. OttoController — Task Architecture

**File**: `main/boards/otto-robot/otto_controller.cc`

`OttoController` is the bridge between the MCP tool layer (AI commands) and the `Otto` motion library. Its core responsibility is ensuring that every blocking motion function runs on a dedicated FreeRTOS task rather than on the main audio task.

### Class Members

```cpp
class OttoController {
private:
    Otto otto_;                            // Motion library instance
    TaskHandle_t action_task_handle_;      // nullptr until first action queued
    QueueHandle_t action_queue_;           // FreeRTOS queue, 10 items
    bool has_hands_;                       // false if hand pins are GPIO_NUM_NC
    bool is_action_in_progress_;           // true while ActionTask is executing a move

    struct OttoActionParams {
        int  action_type;                  // ActionType enum value
        int  steps;
        int  speed;
        int  direction;
        int  amount;
        char servo_sequence_json[512];     // Only used for ACTION_SERVO_SEQUENCE
    };
};
```

### Constructor Sequence

```cpp
OttoController(const HardwareConfig& hw_config) {
    // 1. Initialize servo pins from board hardware config
    otto_.Init(hw_config.left_leg_pin,  hw_config.right_leg_pin,
               hw_config.left_foot_pin, hw_config.right_foot_pin,
               hw_config.left_hand_pin, hw_config.right_hand_pin);

    // 2. Determine if hand servos are wired
    has_hands_ = (hw_config.left_hand_pin  != GPIO_NUM_NC &&
                  hw_config.right_hand_pin != GPIO_NUM_NC);

    // 3. Restore servo trim calibration from NVS flash
    LoadTrimsFromNVS();

    // 4. Create the action queue (10 slots)
    action_queue_ = xQueueCreate(10, sizeof(OttoActionParams));

    // 5. Queue a HOME action — robot centres all servos on boot
    //    direction=1 means hand servos also go to rest position
    QueueAction(ACTION_HOME, 1, 1000, 1, 0);

    // 6. Register all MCP tools so the AI can call them
    RegisterMcpTools();
}
```

At this point `action_task_handle_` is still `nullptr`. The ActionTask is **lazily created** on the first `QueueAction()` call via `StartActionTaskIfNeeded()`.

### The Action Queue

```cpp
action_queue_ = xQueueCreate(10, sizeof(OttoActionParams));
```

- **Capacity**: 10 items. Each item is an `OttoActionParams` struct (~520 bytes).
- **Send**: `xQueueSend(action_queue_, &params, portMAX_DELAY)` — if the queue is full, the caller **blocks** until a slot is available.
- **Receive**: `xQueueReceive(..., pdMS_TO_TICKS(1000))` — ActionTask blocks for up to 1 second waiting for the next item.

The queue allows the AI to chain several actions back-to-back without waiting for each one to finish. Up to 10 commands can be buffered before the caller starts blocking.

### QueueAction — Enqueuing a Named Motion

```cpp
void QueueAction(int action_type, int steps, int speed,
                 int direction, int amount) {
    // Guard: silently reject hand-only actions if no hand servos present
    if ((action_type >= ACTION_HANDS_UP && action_type <= ACTION_HAND_WAVE) ||
        action_type == ACTION_WINDMILL  || action_type == ACTION_TAKEOFF   ||
        action_type == ACTION_FITNESS   || action_type == ACTION_GREETING  ||
        action_type == ACTION_SHY       || action_type == ACTION_RADIO_CALISTHENICS ||
        action_type == ACTION_MAGIC_CIRCLE) {
        if (!has_hands_) return;   // drop silently
    }

    OttoActionParams params = {action_type, steps, speed, direction, amount, ""};
    xQueueSend(action_queue_, &params, portMAX_DELAY);
    StartActionTaskIfNeeded();
}
```

### QueueServoSequence — Enqueuing a JSON Motion

```cpp
void QueueServoSequence(const char* json_str) {
    // Validate length — buffer is exactly 512 bytes
    if (strlen(json_str) >= 512) { ESP_LOGE(...); return; }

    OttoActionParams params = {ACTION_SERVO_SEQUENCE, 0, 0, 0, 0, ""};
    strncpy(params.servo_sequence_json, json_str, 511);
    params.servo_sequence_json[511] = '\0';

    xQueueSend(action_queue_, &params, portMAX_DELAY);
    StartActionTaskIfNeeded();
}
```

### Lazy Task Creation

```cpp
void StartActionTaskIfNeeded() {
    if (action_task_handle_ == nullptr) {
        xTaskCreate(
            ActionTask,               // Task function
            "otto_action",            // Name (visible in task monitor)
            1024 * 3,                 // Stack: 3 KB
            this,                     // Argument passed to task
            configMAX_PRIORITIES - 1, // Priority: highest available
            &action_task_handle_      // Handle stored for later deletion
        );
    }
}
```

The task is created **once** and then runs forever. It is only re-created if `self.otto.stop` deletes it.

**Task properties:**
| Property | Value |
|---|---|
| Name | `otto_action` |
| Stack size | 3 072 bytes (3 KB) |
| Priority | `configMAX_PRIORITIES - 1` (highest, preempts everything except ISRs) |
| Created | On first `QueueAction()` call |
| Destroyed | Only by `self.otto.stop` |

### ActionTask — The Motion Execution Loop

This is the heart of the motion system. The full logic:

```cpp
static void ActionTask(void* arg) {
    OttoController* ctrl = static_cast<OttoController*>(arg);
    OttoActionParams params;

    // AttachServos() claims LEDC channels for all configured servo pins.
    // Must be called from the task that will use them.
    ctrl->otto_.AttachServos();

    while (true) {
        // Block here until an action arrives, or 1 second passes.
        // The 1-second timeout keeps the task alive even when idle.
        if (xQueueReceive(ctrl->action_queue_, &params,
                          pdMS_TO_TICKS(1000)) == pdTRUE) {

            // --- PRE-MOTION ---
            PowerManager::PauseBatteryUpdate();   // ADC reads would interfere
            ctrl->is_action_in_progress_ = true;

            // --- EXECUTE ---
            if (params.action_type == ACTION_SERVO_SEQUENCE) {
                // Parse and execute custom JSON sequence (see Section 7)
                ExecuteServoSequence(ctrl, params.servo_sequence_json);

            } else {
                // Dispatch to the matching Otto motion function
                switch (params.action_type) {
                    case ACTION_WALK:
                        ctrl->otto_.Walk(params.steps, params.speed,
                                         params.direction, params.amount);
                        break;
                    case ACTION_TURN:
                        ctrl->otto_.Turn(params.steps, params.speed,
                                         params.direction, params.amount);
                        break;
                    case ACTION_JUMP:
                        ctrl->otto_.Jump(params.steps, params.speed);
                        break;
                    case ACTION_SWING:
                        ctrl->otto_.Swing(params.steps, params.speed, params.amount);
                        break;
                    case ACTION_MOONWALK:
                        ctrl->otto_.Moonwalker(params.steps, params.speed,
                                               params.amount, params.direction);
                        break;
                    // ... (all 28 action types handled) ...
                    case ACTION_HOME:
                        ctrl->otto_.Home(true);
                        break;
                }

                // --- AUTO-HOME ---
                // After every action except sit/home/servo_sequence,
                // return to the neutral standing pose.
                // For HANDS_UP specifically, hands_down=false keeps arms raised.
                if (params.action_type != ACTION_SIT  &&
                    params.action_type != ACTION_HOME  &&
                    params.action_type != ACTION_SERVO_SEQUENCE) {
                    ctrl->otto_.Home(params.action_type != ACTION_HANDS_UP);
                }
            }

            // --- POST-MOTION ---
            ctrl->is_action_in_progress_ = false;
            PowerManager::ResumeBatteryUpdate();

            // 20ms yield before checking the queue again
            vTaskDelay(pdMS_TO_TICKS(20));
        }
        // If xQueueReceive timed out (no action in 1s), the while loop
        // immediately calls xQueueReceive again — task stays alive but idle.
    }
}
```

### Auto-Home Behaviour

After every action completes, the robot returns to its standing rest pose automatically. The only exceptions are:

| Action | Auto-home? | Reason |
|---|---|---|
| All locomotion and body moves | Yes, `Home(true)` | Clean starting position for next move |
| `hands_up` | Yes, `Home(false)` | Body homes but **arms stay raised** |
| `sit` | No | Intentional resting pose |
| `home` | No | Is itself the home action |
| `servo_sequence` | No | Caller controls final position; use `self.otto.action {"action":"home"}` when done |

### Stop Mechanism

`self.otto.stop` performs a hard stop:

```cpp
// 1. Delete the running task immediately — any in-progress motion is cut short
vTaskDelete(action_task_handle_);
action_task_handle_ = nullptr;
is_action_in_progress_ = false;

// 2. Drain the queue so no pending actions execute
xQueueReset(action_queue_);

// 3. Queue a fresh HOME action and re-create the task
QueueAction(ACTION_HOME, 1, 1000, 1, 0);
// StartActionTaskIfNeeded() re-creates the task
```

The robot snaps to home position as quickly as `Otto::Home()` allows.

### PowerManager Integration

The `PowerManager` measures battery voltage using an ADC. During a motion, servo current spikes cause voltage droop that would corrupt the ADC reading. The controller brackets every action:

```cpp
PowerManager::PauseBatteryUpdate();   // before motion starts
// ... motion runs ...
PowerManager::ResumeBatteryUpdate();  // after motion + auto-home completes
```

Battery level readings are therefore momentarily stale during movement, which is acceptable since the display update rate is 1 Hz.

### Task Interaction Summary

```
MCP TOOL CALLBACK          ACTION TASK              AUDIO TASK
(runs on main thread)      (highest priority)       (audio pipeline)

AI calls self.otto.action
    │
    ▼
QueueAction(ACTION_WALK)
xQueueSend(...)  ──────────► queue slot filled
return true  ◄─────────────  (immediately)
    │                              │
    │                        xQueueReceive unblocks
    │                        PauseBatteryUpdate()
    │                        otto_.Walk()   ← blocking ~2–3s
    │                        otto_.Home()
    │                        ResumeBatteryUpdate()
    │                        vTaskDelay(20ms)
    ▼                              │
AI receives "true" immediately     ▼
continues speaking          waits for next item
```

The MCP callback returns before the motion even starts. The AI's TTS response plays while Otto is walking.

---

## 6. MCP Tools — AI Motion Commands

All tools are registered in `OttoController::RegisterMcpTools()` during boot.

### `self.otto.action` — Execute a Named Motion

| Parameter | Type | Default | Range | Description |
|---|---|---|---|---|
| `action` | string | `"sit"` | — | Motion name (see table below) |
| `steps` | int | 3 | 1–100 | Repetitions / cycles |
| `speed` | int | 700 | 100–3000 | Cycle duration ms (lower = faster) |
| `direction` | int | 1 | -1, 0, 1 | 1=forward/left, -1=backward/right, 0=both |
| `amount` | int | 30 | 0–170 | Oscillation amplitude in degrees |
| `arm_swing` | int | 50 | 0–170 | Arm swing for walk/turn only |

**Action names:**

| Category | Name | Key params |
|---|---|---|
| Locomotion | `walk` | steps, speed, direction, arm_swing |
| | `turn` | steps, speed, direction, arm_swing |
| | `jump` | steps, speed |
| | `moonwalk` | steps, speed, direction, amount |
| Body | `swing` | steps, speed, amount |
| | `updown` | steps, speed, amount |
| | `tiptoe_swing` | steps, speed, amount |
| | `jitter` | steps, speed, amount |
| | `ascending_turn` | steps, speed, amount |
| | `crusaito` | steps, speed, amount, direction |
| | `flapping` | steps, speed, amount, direction |
| | `whirlwind_leg` | steps, speed, amount |
| | `bend` | steps, speed, direction |
| | `shake_leg` | steps, speed, direction |
| Fixed | `sit` | — |
| | `showcase` | — |
| | `home` | — |
| Arms* | `hands_up` | speed, direction |
| | `hands_down` | speed, direction |
| | `hand_wave` | direction |
| | `windmill` | steps, speed, amount |
| | `takeoff` | steps, speed, amount |
| | `fitness` | steps, speed, amount |
| | `greeting` | direction, steps |
| | `shy` | direction, steps |
| | `radio_calisthenics` | — |
| | `magic_circle` | — |

*Arm actions are silently dropped if hand servos are not wired (`GPIO_NUM_NC`).

### `self.otto.servo_sequences` — Custom AI-Programmed Motion

Lets the AI define arbitrary motions in JSON. Multiple calls are queued and executed in order.

See [Section 7](#7-custom-servo-sequence-protocol) for the full JSON format.

### `self.otto.stop` — Emergency Stop

Deletes the action task, drains the queue, homes the robot immediately.

### `self.otto.get_status` — Query Motion State

Returns `"moving"` or `"idle"` based on `is_action_in_progress_`.

### `self.otto.set_trim` / `self.otto.get_trims` — Servo Calibration

See [Section 9](#9-servo-calibration-trim-system).

### `self.otto.get_ip` — Network Info

Returns `{"ip":"192.168.x.x","connected":true}`.

### `self.battery.get_level` — Battery Status

Returns battery percentage and charging state.

---

## 7. Custom Servo Sequence Protocol

`self.otto.servo_sequences` accepts a JSON string (max 511 bytes) describing an arbitrary motion sequence. Multiple calls are queued; the AI can chain them to build longer animations.

### Top-Level Format

```json
{
  "a": [ <action>, <action>, ... ],
  "d": 500
}
```

- `"a"` — array of action objects (required)
- `"d"` — delay in ms after the whole sequence finishes, applied only if more sequences are waiting in the queue (optional)

### Action Object — Direct Move Mode

```json
{
  "s": { "ll": 100, "rl": 80, "lf": 90, "rf": 90, "lh": 45, "rh": 135 },
  "v": 1000,
  "d": 200
}
```

- `"s"` — servo target positions in absolute degrees 0–180 (only specify servos you want to move; others hold their current position)
- `"v"` — transition duration in ms (100–3000, default 1000)
- `"d"` — pause after this step in ms (optional)

### Action Object — Oscillator Mode

```json
{
  "osc": {
    "a":  { "lh": 30, "rh": 30 },
    "o":  { "lh": 90, "rh": 90 },
    "ph": { "rh": 180 },
    "p":  500,
    "c":  5.0
  }
}
```

- `"a"` — amplitude per servo in degrees (10–90, default 0)
- `"o"` — oscillation centre in absolute degrees 0–180 (default 90)
- `"ph"` — phase offset in degrees 0–360 (default 0)
- `"p"` — period in ms (100–3000, default 500)
- `"c"` — number of oscillation cycles (0.1–20.0, default 5.0)

### Execution Flow in ActionTask

```
xQueueReceive → ACTION_SERVO_SEQUENCE
    └─ cJSON_Parse(servo_sequence_json)
    └─ iterate "a" array
        ├─ "osc" key present? → otto_.Execute2(amplitude, center_angle, period, phase, steps)
        └─ "s" key present?  → otto_.MoveServos(speed, servo_target[])
        └─ apply per-step "d" delay (except after the last step)
    └─ if "d" top-level && queue still has items → vTaskDelay(d ms)
    └─ cJSON_Delete(json)
```

### Safety Rule for Oscillator Mode

Simultaneously oscillating both legs (or both feet) with amplitude ≥ 40° can stress the frame. The firmware enforces:

```
if amplitude[LEFT_LEG]  >= 40 && amplitude[RIGHT_LEG]  >= 40 → RIGHT_LEG  forced to 0
if amplitude[LEFT_FOOT] >= 40 && amplitude[RIGHT_FOOT] >= 40 → RIGHT_FOOT forced to 0
```

Always keep one leg at 90° neutral when oscillating the other.

### Examples

**Wave both arms in antiphase:**
```json
{"a":[{"osc":{"a":{"lh":30,"rh":30},"o":{"lh":90,"rh":90},"ph":{"rh":180},"p":500,"c":5.0}}],"d":0}
```

**Step forward (weight shift then leg swing):**
```json
{"a":[{"s":{"ll":100},"v":800},{"s":{"rf":60},"v":500},{"s":{"ll":90,"rf":90},"v":800}],"d":0}
```

**Safe left-leg oscillation (right leg stays neutral):**
```json
{"a":[{"osc":{"a":{"ll":45},"o":{"ll":90,"lf":90},"p":400,"c":4.0}}],"d":0}
```

**Multi-call chaining pattern:**
```
Call 1: {"sequence": "{\"a\":[{\"s\":{\"ll\":100},\"v\":1000}],\"d\":500}"}
Call 2: {"sequence": "{\"a\":[{\"s\":{\"ll\":80},\"v\":800}],\"d\":500}"}
Call 3: {"sequence": "{\"a\":[{\"s\":{\"ll\":90},\"v\":800}]}"}
Then: self.otto.action {"action": "home"}
```

---

## 8. Hardware Configuration

**File**: `main/boards/otto-robot/config.h`

Two hardware variants are auto-detected at runtime via camera presence detection. One firmware binary supports both.

### Camera Version GPIO

| Signal | GPIO |
|---|---|
| Left Leg | 5 |
| Left Foot | 6 |
| Right Leg | 43 |
| Right Foot | 44 |
| Left Hand | 4 |
| Right Hand | 7 |
| I2S WS | 40 |
| I2S BCLK | 42 |
| I2S DIN | 41 |
| I2S DOUT | 39 |
| Display BL | 38 |
| I2C SDA | 15 |
| I2C SCL | 16 |

### Non-Camera Version GPIO

| Signal | GPIO |
|---|---|
| Left Leg | 17 |
| Left Foot | 18 |
| Right Leg | 39 |
| Right Foot | 38 |
| Left Hand | 8 |
| Right Hand | 12 |
| MIC WS | 4 |
| MIC SCK | 5 |
| MIC DIN | 6 |
| SPK DOUT | 7 |
| SPK BCLK | 15 |
| SPK LRCK | 16 |
| Display BL | 3 |
| Charge Detect | 21 |

### Hand Servo Detection

```cpp
has_hands_ = (hw_config.left_hand_pin  != GPIO_NUM_NC &&
              hw_config.right_hand_pin != GPIO_NUM_NC);
```

Setting either hand pin to `GPIO_NUM_NC` in the config disables all arm actions. They will be silently dropped by `QueueAction()`.

---

## 9. Servo Calibration (Trim System)

Mechanical tolerances mean servos do not sit exactly at their intended neutral angles. The trim system adds a per-servo degree offset stored persistently in NVS flash.

### How It Works

```
actual_pwm_position = commanded_position + trim
```

Each `Oscillator` instance holds its trim value and applies it on every `Refresh()` and `SetPosition()` call.

### Loading from NVS at Boot

```cpp
void LoadTrimsFromNVS() {
    Settings settings("otto_trims", false);   // NVS namespace "otto_trims"

    otto_.SetTrims(
        settings.GetInt("left_leg",   0),
        settings.GetInt("right_leg",  0),
        settings.GetInt("left_foot",  0),
        settings.GetInt("right_foot", 0),
        settings.GetInt("left_hand",  0),
        settings.GetInt("right_hand", 0)
    );
}
```

Called during `OttoController` constructor before the first `HOME` action.

### Adjusting Trims via MCP

```
self.otto.set_trim
  servo_type:  left_leg | right_leg | left_foot | right_foot | left_hand | right_hand
  trim_value:  -50 to +50 (degrees)
```

The new value is written to NVS immediately and `SetTrims()` is called so the change takes effect without a reboot. Followed by a `QueueAction(ACTION_HOME)` to visually confirm the result.

### Calibration Procedure

1. `self.otto.action {"action": "home"}` — observe neutral standing pose
2. Identify the misaligned servo
3. `self.otto.set_trim {"servo_type": "left_foot", "trim_value": -5}` — adjust in steps
4. Repeat until level
5. `self.otto.get_trims` — verify all values are saved

---

## 10. Architecture Diagram

```
┌──────────────────────────────────────────────────────────────────────┐
│                        CLOUD AI (LLM)                               │
│  "Walk forward 4 steps, then wave hello"                            │
└───────────────────────────┬──────────────────────────────────────────┘
                            │ MCP JSON over WebSocket / MQTT
                            ▼
┌──────────────────────────────────────────────────────────────────────┐
│                  MAIN TASK — McpServer::HandleRequest()              │
│  Parses JSON, validates parameters, calls tool callback             │
│  → QueueAction() → xQueueSend (non-blocking, returns immediately)   │
│  → return true to AI                                                │
└───────────────────────────┬──────────────────────────────────────────┘
                            │
              ┌─────────────▼──────────────────┐
              │  action_queue_  (10 slots)      │
              │  OttoActionParams structs       │
              └─────────────┬──────────────────┘
                            │ xQueueReceive (blocks up to 1s)
                            ▼
┌──────────────────────────────────────────────────────────────────────┐
│              ActionTask  (priority: configMAX_PRIORITIES-1)          │
│                                                                      │
│  1. AttachServos()            — claim LEDC channels on first run    │
│  2. PauseBatteryUpdate()      — prevent ADC noise from servo draw   │
│  3. is_action_in_progress_ = true                                   │
│  4. switch(action_type)       — dispatch to Otto motion function    │
│  5. otto_.Home() if needed    — auto-return to neutral pose         │
│  6. is_action_in_progress_ = false                                  │
│  7. ResumeBatteryUpdate()                                           │
│  8. vTaskDelay(20ms)          — yield before next queue check       │
└───────────────────────────┬──────────────────────────────────────────┘
                            │
                            ▼
┌──────────────────────────────────────────────────────────────────────┐
│                 Otto class  (otto_movements.h/cc)                    │
│                                                                      │
│  Walk()   → Execute()  → OscillateServos() → servo_[i].Refresh()   │
│  MoveServos() → interpolation loop → servo_[i].SetPosition()       │
│  Home()   → MoveServos(to rest positions)                          │
└───────────────────────────┬──────────────────────────────────────────┘
                            │ one call per servo
                            ▼
┌──────────────────────────────────────────────────────────────────────┐
│                 Oscillator  (oscillator.h/cc)                        │
│                                                                      │
│  Refresh()     → phase += inc_ → sin() → Write() → ledc_set_duty() │
│  SetPosition() → AngleToCompare(deg) → ledc_set_duty()             │
│                   maps [-90°…+90°] to [500µs…2500µs] pulse width   │
└───────────────────────────┬──────────────────────────────────────────┘
                            │ LEDC peripheral (50 Hz PWM)
                            ▼
              [Servo motors: ll, rl, lf, rf, lh, rh]
```

---

## 11. Quick Reference

### Servo Short Names

| Key | Full Name | Physical Motion |
|---|---|---|
| `ll` | left_leg | Lateral swing (inner/outer) |
| `rl` | right_leg | Lateral swing |
| `lf` | left_foot | Foot pitch (toe up/down) |
| `rf` | right_foot | Foot pitch |
| `lh` | left_hand | Arm up/down |
| `rh` | right_hand | Arm up/down |

### Direction Parameter

| Value | Walk/Turn | Bend/ShakeLeg/Moonwalk | Hand moves |
|---|---|---|---|
| `1` | Forward / Left | Left | Left arm |
| `-1` | Backward / Right | Right | Right arm |
| `0` | — | — | Both arms |

### Speed Guide

| speed (ms) | Feel |
|---|---|
| 100–300 | Very fast / snappy |
| 400–700 | Natural |
| 800–1200 | Slow and deliberate |
| 1500–3000 | Very slow |

### LEDC Channel Budget

| Servos | Channels used | Free |
|---|---|---|
| 4 (legs + feet only) | 4 | 4 |
| 6 (+ arms) | 6 | 2 |

### Key Source Files

| File | Role |
|---|---|
| `otto-robot/oscillator.h/cc` | Servo PWM driver |
| `otto-robot/otto_movements.h/cc` | All motion functions |
| `otto-robot/otto_controller.cc` | FreeRTOS queue + MCP tools |
| `otto-robot/otto_robot.cc` | Board class, calls `InitializeOttoController()` |
| `otto-robot/config.h` | `HardwareConfig` struct + GPIO pinouts |
| `otto-robot/power_manager.h` | Battery ADC pause/resume during motion |

---

*Firmware version 2.2.4 — `main/boards/otto-robot/`*
