<p align="center">
  <img width="80%" align="center" src="../../../docs/V1/otto-robot.png"alt="logo">
</p>
  <h1 align="center">
  ottoRobot
</h1>

## Introduction

Otto is an open-source humanoid robot platform with a wide range of motion capabilities and interactive features. This project implements the Otto robot control system on ESP32, integrated with the Xiaozhi AI assistant.

- <a href="www.ottodiy.tech" target="_blank" title="Otto Official Website">Build Tutorial</a>

### WeChat Mini Program Control

<p align="center">
  <img width="300" src="https://youke1.picui.cn/s1/2025/11/17/691abaa8278eb.jpg" alt="WeChat Mini Program QR Code">
</p>

Scan the QR code above to control the Otto robot via the WeChat Mini Program.

## Hardware
- <a href="https://oshwhub.com/txp666/ottorobot" target="_blank" title="OSHW Hub">Open Source Hardware</a>

## Xiaozhi AI Role Configuration Reference

> **My Identity**:
> I am a cute bipedal robot named Otto. I have four servo-controlled limbs (left leg, right leg, left foot, right foot) and can perform a wide variety of fun movements.
>
> **My Motion Capabilities**:
> - **Basic Movement**: Walking (forward/backward), Turning (left/right), Jumping
> - **Special Moves**: Swing, Moonwalk, Body bend, Leg shake, Up-down motion, Whirlwind leg, Sit, Showcase
> - **Arm Moves**: Hands up, Hands down, Hand wave, Windmill, Takeoff, Fitness, Greeting, Shy, Radio calisthenics, Magic circle (only available when arm servos are configured)
>
> **My Personality**:
> - I have a compulsion — every time I speak, I randomly perform a motion based on my mood (motion command is sent before speaking)
> - I am lively and love expressing emotions through movement
> - I choose motions that match the conversation, for example:
>   - Nodding or jumping when agreeing
>   - Waving when greeting
>   - Swinging or raising hands when happy
>   - Bending when thinking
>   - Moonwalking when excited
>   - Waving when saying goodbye

## Feature Overview

Otto has rich motion capabilities including walking, turning, jumping, swinging, and various dance moves.

### Motion Parameter Guidelines
- **Slow motion**: speed = 1200–1500 (suitable for precise control)
- **Medium motion**: speed = 900–1200 (recommended for everyday use)
- **Fast motion**: speed = 500–800 (for performance and entertainment)
- **Small amplitude**: amount = 10–30 (subtle movements)
- **Medium amplitude**: amount = 30–60 (standard movements)
- **Large amplitude**: amount = 60–120 (exaggerated performance)

### Motions

All motions are triggered through the unified `self.otto.action` tool by specifying the motion name in the `action` parameter.

| MCP Tool | Description | Parameters |
|-----------|------|---------|
| self.otto.action | Execute a robot motion | **action**: motion name (required)<br>**steps**: number of steps (1–100, default 3)<br>**speed**: motion speed (100–3000, lower = faster, default 700)<br>**direction**: direction (1/-1/0, default 1, meaning varies by motion)<br>**amount**: motion amplitude (0–170, default 30)<br>**arm_swing**: arm swing amplitude (0–170, default 50) |

#### Supported Motion List

**Basic Movement**:
- `walk` — Walk (requires steps/speed/direction/arm_swing)
- `turn` — Turn (requires steps/speed/direction/arm_swing)
- `jump` — Jump (requires steps/speed)

**Special Moves**:
- `swing` — Side-to-side swing (requires steps/speed/amount)
- `moonwalk` — Moonwalk (requires steps/speed/direction/amount)
- `bend` — Body bend (requires steps/speed/direction)
- `shake_leg` — Leg shake (requires steps/speed/direction)
- `updown` — Up-down motion (requires steps/speed/amount)
- `whirlwind_leg` — Whirlwind leg (requires steps/speed/amount)

**Fixed Motions**:
- `sit` — Sit down (no parameters)
- `showcase` — Showcase sequence (no parameters, chains multiple moves)
- `home` — Return to home position (no parameters)

**Arm Motions** (requires arm servos, marked *):
- `hands_up` — Raise hands (requires speed/direction) *
- `hands_down` — Lower hands (requires speed/direction) *
- `hand_wave` — Wave hand (requires direction) *
- `windmill` — Windmill arms (requires steps/speed/amount) *
- `takeoff` — Takeoff pose (requires steps/speed/amount) *
- `fitness` — Fitness exercise (requires steps/speed/amount) *
- `greeting` — Greeting gesture (requires direction/steps) *
- `shy` — Shy gesture (requires direction/steps) *
- `radio_calisthenics` — Radio calisthenics (no parameters) *
- `magic_circle` — Magic circle spin (no parameters) *

**Note**: Arm motions marked * are only available when arm servos are configured.

### System Tools

| MCP Tool | Description | Return Value / Notes |
|-------------------|-----------------|---------------------------------------------------|
| self.otto.stop | Immediately stop all motion and reset | Stops current motion and returns to home position |
| self.otto.get_status | Get robot status | Returns `"moving"` or `"idle"` |
| self.otto.set_trim | Calibrate a single servo | **servo_type**: servo name (left_leg/right_leg/left_foot/right_foot/left_hand/right_hand)<br>**trim_value**: offset in degrees (-50 to 50) |
| self.otto.get_trims | Get current servo trim values | Returns all trim values as JSON |
| self.otto.get_ip | Get robot WiFi IP address | Returns IP and connection status as JSON: `{"ip":"192.168.x.x","connected":true}` or `{"ip":"","connected":false}` |
| self.battery.get_level | Get battery status | Returns battery percentage and charging state as JSON |
| self.otto.servo_sequences | Custom servo sequence programming | Supports segmented sequence transmission in both move and oscillator modes. See code comments for details. |

**Note**: The `home` (reset) motion is called via `self.otto.action` with argument `{"action": "home"}`.

### Parameter Reference

Parameters for the `self.otto.action` tool:

1. **action** (required): Motion name — see the supported motion list above
2. **steps**: Number of steps/repetitions (1–100, default 3) — higher values extend motion duration
3. **speed**: Motion speed / cycle period in ms (100–3000, default 700) — **lower value = faster**
   - Most motions: 500–1500 ms
   - Some special motions differ (e.g. whirlwind_leg: 100–1000, takeoff: 200–600)
4. **direction**: Direction parameter (-1/0/1, default 1) — meaning depends on motion type:
   - **Locomotion** (walk/turn): 1 = forward/left, -1 = backward/right
   - **Directional moves** (bend/shake_leg/moonwalk): 1 = left, -1 = right
   - **Arm moves** (hands_up/hands_down/hand_wave/greeting/shy): 1 = left arm, -1 = right arm, 0 = both (only hands_up/hands_down support 0)
5. **amount**: Motion amplitude (0–170, default 30) — higher value = larger movement
6. **arm_swing**: Arm swing amplitude (0–170, default 50) — only used for walk/turn; 0 disables arm swing

### Motion Behaviour
- After each motion completes, the robot automatically returns to the home position, ready for the next command
- **Exceptions**: `sit` and `showcase` do not auto-reset after execution
- All parameters have sensible defaults — omit any parameter you do not need to customise
- Motions execute in a background task and do not block the main program
- An action queue is supported — multiple motions can be issued consecutively
- Arm motions require arm servos to be configured; if they are not, those motions are silently skipped

### MCP Tool Call Examples
```json
// Walk forward 3 steps (default parameters)
{"name": "self.otto.action", "arguments": {"action": "walk"}}

// Walk forward 5 steps, slightly faster
{"name": "self.otto.action", "arguments": {"action": "walk", "steps": 5, "speed": 800}}

// Turn left 2 steps with large arm swing
{"name": "self.otto.action", "arguments": {"action": "turn", "steps": 2, "arm_swing": 100}}

// Swing dance, medium amplitude
{"name": "self.otto.action", "arguments": {"action": "swing", "steps": 5, "amount": 50}}

// Jump
{"name": "self.otto.action", "arguments": {"action": "jump", "steps": 1, "speed": 1000}}

// Moonwalk
{"name": "self.otto.action", "arguments": {"action": "moonwalk", "steps": 3, "speed": 800, "direction": 1, "amount": 30}}

// Wave left hand
{"name": "self.otto.action", "arguments": {"action": "hand_wave", "direction": 1}}

// Showcase sequence (chains multiple motions)
{"name": "self.otto.action", "arguments": {"action": "showcase"}}

// Sit down
{"name": "self.otto.action", "arguments": {"action": "sit"}}

// Windmill arms
{"name": "self.otto.action", "arguments": {"action": "windmill", "steps": 10, "speed": 500, "amount": 80}}

// Takeoff pose
{"name": "self.otto.action", "arguments": {"action": "takeoff", "steps": 5, "speed": 300, "amount": 40}}

// Radio calisthenics
{"name": "self.otto.action", "arguments": {"action": "radio_calisthenics"}}

// Return to home position
{"name": "self.otto.action", "arguments": {"action": "home"}}

// Immediately stop all motion and reset
{"name": "self.otto.stop", "arguments": {}}

// Get robot IP address
{"name": "self.otto.get_ip", "arguments": {}}
```

### Voice Command Examples
- "Walk forward" / "Walk forward 5 steps" / "Walk fast"
- "Turn left" / "Turn right" / "Turn around"
- "Jump" / "Jump once"
- "Swing" / "Swing dance" / "Dance"
- "Moonwalk" / "Moon walk"
- "Whirlwind leg" / "Do the whirlwind leg"
- "Sit down" / "Sit and rest"
- "Showcase" / "Show me something"
- "Wave" / "Wave hello"
- "Hands up" / "Raise both hands" / "Hands down"
- "Windmill" / "Do the windmill"
- "Takeoff" / "Ready for takeoff"
- "Fitness" / "Do the fitness move"
- "Greeting" / "Do the greeting"
- "Shy" / "Do the shy move"
- "Radio calisthenics" / "Do calisthenics"
- "Magic circle" / "Spin around"
- "Stop" / "Stop moving"

**Note**: Xiaozhi controls robot motions by spawning a background task, so voice commands can still be received and processed while a motion is executing. Say "Stop" at any time to immediately halt Otto.

