# Build Guide — Otto Robot

**Target chip**: ESP32-S3
**Framework**: ESP-IDF v5.3+

---

## Prerequisites

- ESP-IDF v5.3 or later installed and activated (`idf.py` available in PATH)
- USB cable connected to the board's UART/JTAG port
- Serial port identified (e.g. `COM3` on Windows, `/dev/ttyUSB0` on Linux/Mac)

---

## Build Steps

### 1. Set the target chip

```bash
cd D:/otto/xiaozhi-esp32-otto
idf.py set-target esp32s3
```

This generates a fresh `sdkconfig` for ESP32-S3 and applies `sdkconfig.defaults.esp32s3`.

---

### 2. Select the Otto board in menuconfig

```bash
idf.py menuconfig
```

Navigate to:
```
(Top) → Xiaozhi AI Assistant Configuration → Board Type
```

Select **`ottoRobot`**, then save and exit (`S` → `Q`).

This sets `CONFIG_BOARD_TYPE_OTTO_ROBOT=y`, which tells the build system to:
- Compile `main/boards/otto-robot/`
- Use the `otto-gif` emoji collection
- Include the WebSocket control server (`CONFIG_HTTPD_WS_SUPPORT=y`)
- Include OV2640 and OV3660 camera drivers

---

### 3. Build

```bash
idf.py build
```

---

### 4. Flash and monitor

```bash
idf.py -p COM3 flash monitor
```

Replace `COM3` with your actual port. To flash and monitor separately:

```bash
idf.py -p COM3 flash
idf.py -p COM3 monitor
```

---

### One-liner (after first-time menuconfig)

Once `sdkconfig` already has `CONFIG_BOARD_TYPE_OTTO_ROBOT=y`:

```bash
idf.py build flash monitor -p COM3
```

---

## Notes

| Topic | Detail |
|---|---|
| **Hardware variant** | Camera vs non-camera is auto-detected at runtime — one firmware works for both |
| **First boot** | Device enters WiFi provisioning mode; configure WiFi via BluFi or the WeChat Mini Program |
| **Switching board types** | Run `idf.py fullclean` before changing board type in menuconfig, then rebuild from step 1 |
| **Monitor baud rate** | Set in `sdkconfig`, typically 115200 |
