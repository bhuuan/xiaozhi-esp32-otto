# Custom Board Guide

This guide explains how to create a new board initialization implementation for the Xiaozhi AI voice chatbot project. Xiaozhi AI supports 70+ ESP32-series boards; each board's initialization code lives in its own directory.

## Important Warning

> **Warning**: For custom boards, if your IO configuration differs from an existing board, do NOT directly overwrite that board's configuration and compile firmware. You must create a new board type, or use the `builds` config in `config.json` with a different `name` and `sdkconfig` macros to distinguish them. Use `python scripts/release.py [board-directory-name]` to build and package firmware.
>
> If you overwrite an existing board's config, future OTA upgrades may replace your custom firmware with the original board's standard firmware, breaking your device. Each board has a unique identifier and its own firmware upgrade channel — keeping board identifiers unique is critical.

## Directory Structure

Each board directory typically contains:

- `xxx_board.cc` — Main board initialization code; implements board-specific init and functionality
- `config.h` — Board config file; defines hardware pin mappings and other settings
- `config.json` — Build config; specifies target chip and special build options
- `README.md` — Board-specific documentation

## Steps to Create a Custom Board

### 1. Create a New Board Directory

Create a new directory under `boards/` using the naming convention `[brand]-[board-type]`, e.g. `m5stack-tab5`:

```bash
mkdir main/boards/my-custom-board
```

### 2. Create Config Files

#### config.h

Define all hardware configuration in `config.h`, including:

- Audio sample rate and I2S pin config
- Audio codec chip address and I2C pin config
- Button and LED pin config
- Display parameters and pin config

Reference example (from lichuang-c3-dev):

```c
#ifndef _BOARD_CONFIG_H_
#define _BOARD_CONFIG_H_

#include <driver/gpio.h>

// Audio config
#define AUDIO_INPUT_SAMPLE_RATE  24000
#define AUDIO_OUTPUT_SAMPLE_RATE 24000

#define AUDIO_I2S_GPIO_MCLK GPIO_NUM_10
#define AUDIO_I2S_GPIO_WS   GPIO_NUM_12
#define AUDIO_I2S_GPIO_BCLK GPIO_NUM_8
#define AUDIO_I2S_GPIO_DIN  GPIO_NUM_7
#define AUDIO_I2S_GPIO_DOUT GPIO_NUM_11

#define AUDIO_CODEC_PA_PIN       GPIO_NUM_13
#define AUDIO_CODEC_I2C_SDA_PIN  GPIO_NUM_0
#define AUDIO_CODEC_I2C_SCL_PIN  GPIO_NUM_1
#define AUDIO_CODEC_ES8311_ADDR  ES8311_CODEC_DEFAULT_ADDR

// Button config
#define BOOT_BUTTON_GPIO        GPIO_NUM_9

// Display config
#define DISPLAY_SPI_SCK_PIN     GPIO_NUM_3
#define DISPLAY_SPI_MOSI_PIN    GPIO_NUM_5
#define DISPLAY_DC_PIN          GPIO_NUM_6
#define DISPLAY_SPI_CS_PIN      GPIO_NUM_4

#define DISPLAY_WIDTH   320
#define DISPLAY_HEIGHT  240
#define DISPLAY_MIRROR_X true
#define DISPLAY_MIRROR_Y false
#define DISPLAY_SWAP_XY true

#define DISPLAY_OFFSET_X  0
#define DISPLAY_OFFSET_Y  0

#define DISPLAY_BACKLIGHT_PIN GPIO_NUM_2
#define DISPLAY_BACKLIGHT_OUTPUT_INVERT true

#endif // _BOARD_CONFIG_H_
```

#### config.json

Define build configuration in `config.json`. This file is used by the `scripts/release.py` script for automated builds:

```json
{
    "target": "esp32s3",  // Target chip: esp32, esp32s3, esp32c3, esp32c6, esp32p4, etc.
    "builds": [
        {
            "name": "my-custom-board",  // Board name, used for firmware package naming
            "sdkconfig_append": [
                // Custom flash size
                "CONFIG_ESPTOOLPY_FLASHSIZE_8MB=y",
                // Custom partition table
                "CONFIG_PARTITION_TABLE_CUSTOM_FILENAME=\"partitions/v2/8m.csv\""
            ]
        }
    ]
}
```

**Field descriptions:**
- `target`: Target chip type; must match hardware
- `name`: Firmware package output name; should match the directory name
- `sdkconfig_append`: Array of extra sdkconfig entries appended to the defaults

**Common `sdkconfig_append` options:**
```json
// Flash size
"CONFIG_ESPTOOLPY_FLASHSIZE_4MB=y"   // 4MB Flash
"CONFIG_ESPTOOLPY_FLASHSIZE_8MB=y"   // 8MB Flash
"CONFIG_ESPTOOLPY_FLASHSIZE_16MB=y"  // 16MB Flash

// Partition table
"CONFIG_PARTITION_TABLE_CUSTOM_FILENAME=\"partitions/v2/4m.csv\""  // 4MB
"CONFIG_PARTITION_TABLE_CUSTOM_FILENAME=\"partitions/v2/8m.csv\""  // 8MB
"CONFIG_PARTITION_TABLE_CUSTOM_FILENAME=\"partitions/v2/16m.csv\"" // 16MB

// Language
"CONFIG_LANGUAGE_EN_US=y"  // English
"CONFIG_LANGUAGE_ZH_CN=y"  // Simplified Chinese

// Wake word / AEC
"CONFIG_USE_DEVICE_AEC=y"          // Enable device-side AEC
"CONFIG_WAKE_WORD_DISABLED=y"      // Disable wake word
```

### 3. Write Board Initialization Code

Create a `my_custom_board.cc` file implementing all board initialization logic.

A basic board class definition includes:

1. **Class definition**: inherits from `WifiBoard` or `Ml307Board`
2. **Init functions**: I2C, display, buttons, IoT, etc.
3. **Virtual function overrides**: `GetAudioCodec()`, `GetDisplay()`, `GetBacklight()`, etc.
4. **Board registration**: use the `DECLARE_BOARD` macro

```cpp
#include "wifi_board.h"
#include "codecs/es8311_audio_codec.h"
#include "display/lcd_display.h"
#include "application.h"
#include "button.h"
#include "config.h"
#include "mcp_server.h"

#include <esp_log.h>
#include <driver/i2c_master.h>
#include <driver/spi_common.h>

#define TAG "MyCustomBoard"

class MyCustomBoard : public WifiBoard {
private:
    i2c_master_bus_handle_t codec_i2c_bus_;
    Button boot_button_;
    LcdDisplay* display_;

    void InitializeI2c() {
        i2c_master_bus_config_t i2c_bus_cfg = {
            .i2c_port = I2C_NUM_0,
            .sda_io_num = AUDIO_CODEC_I2C_SDA_PIN,
            .scl_io_num = AUDIO_CODEC_I2C_SCL_PIN,
            .clk_source = I2C_CLK_SRC_DEFAULT,
            .glitch_ignore_cnt = 7,
            .intr_priority = 0,
            .trans_queue_depth = 0,
            .flags = {
                .enable_internal_pullup = 1,
            },
        };
        ESP_ERROR_CHECK(i2c_new_master_bus(&i2c_bus_cfg, &codec_i2c_bus_));
    }

    void InitializeSpi() {
        spi_bus_config_t buscfg = {};
        buscfg.mosi_io_num = DISPLAY_SPI_MOSI_PIN;
        buscfg.miso_io_num = GPIO_NUM_NC;
        buscfg.sclk_io_num = DISPLAY_SPI_SCK_PIN;
        buscfg.quadwp_io_num = GPIO_NUM_NC;
        buscfg.quadhd_io_num = GPIO_NUM_NC;
        buscfg.max_transfer_sz = DISPLAY_WIDTH * DISPLAY_HEIGHT * sizeof(uint16_t);
        ESP_ERROR_CHECK(spi_bus_initialize(SPI2_HOST, &buscfg, SPI_DMA_CH_AUTO));
    }

    void InitializeButtons() {
        boot_button_.OnClick([this]() {
            auto& app = Application::GetInstance();
            if (app.GetDeviceState() == kDeviceStateStarting) {
                EnterWifiConfigMode();
                return;
            }
            app.ToggleChatState();
        });
    }

    // Display init example using ST7789
    void InitializeDisplay() {
        esp_lcd_panel_io_handle_t panel_io = nullptr;
        esp_lcd_panel_handle_t panel = nullptr;

        esp_lcd_panel_io_spi_config_t io_config = {};
        io_config.cs_gpio_num = DISPLAY_SPI_CS_PIN;
        io_config.dc_gpio_num = DISPLAY_DC_PIN;
        io_config.spi_mode = 2;
        io_config.pclk_hz = 80 * 1000 * 1000;
        io_config.trans_queue_depth = 10;
        io_config.lcd_cmd_bits = 8;
        io_config.lcd_param_bits = 8;
        ESP_ERROR_CHECK(esp_lcd_new_panel_io_spi(SPI2_HOST, &io_config, &panel_io));

        esp_lcd_panel_dev_config_t panel_config = {};
        panel_config.reset_gpio_num = GPIO_NUM_NC;
        panel_config.rgb_ele_order = LCD_RGB_ELEMENT_ORDER_RGB;
        panel_config.bits_per_pixel = 16;
        ESP_ERROR_CHECK(esp_lcd_new_panel_st7789(panel_io, &panel_config, &panel));

        esp_lcd_panel_reset(panel);
        esp_lcd_panel_init(panel);
        esp_lcd_panel_invert_color(panel, true);
        esp_lcd_panel_swap_xy(panel, DISPLAY_SWAP_XY);
        esp_lcd_panel_mirror(panel, DISPLAY_MIRROR_X, DISPLAY_MIRROR_Y);

        display_ = new SpiLcdDisplay(panel_io, panel,
                                    DISPLAY_WIDTH, DISPLAY_HEIGHT,
                                    DISPLAY_OFFSET_X, DISPLAY_OFFSET_Y,
                                    DISPLAY_MIRROR_X, DISPLAY_MIRROR_Y, DISPLAY_SWAP_XY);
    }

    void InitializeTools() {
        // See MCP docs
    }

public:
    MyCustomBoard() : boot_button_(BOOT_BUTTON_GPIO) {
        InitializeI2c();
        InitializeSpi();
        InitializeDisplay();
        InitializeButtons();
        InitializeTools();
        GetBacklight()->SetBrightness(100);
    }

    virtual AudioCodec* GetAudioCodec() override {
        static Es8311AudioCodec audio_codec(
            codec_i2c_bus_,
            I2C_NUM_0,
            AUDIO_INPUT_SAMPLE_RATE,
            AUDIO_OUTPUT_SAMPLE_RATE,
            AUDIO_I2S_GPIO_MCLK,
            AUDIO_I2S_GPIO_BCLK,
            AUDIO_I2S_GPIO_WS,
            AUDIO_I2S_GPIO_DOUT,
            AUDIO_I2S_GPIO_DIN,
            AUDIO_CODEC_PA_PIN,
            AUDIO_CODEC_ES8311_ADDR);
        return &audio_codec;
    }

    virtual Display* GetDisplay() override {
        return display_;
    }

    virtual Backlight* GetBacklight() override {
        static PwmBacklight backlight(DISPLAY_BACKLIGHT_PIN, DISPLAY_BACKLIGHT_OUTPUT_INVERT);
        return &backlight;
    }
};

// Register board
DECLARE_BOARD(MyCustomBoard);
```

### 4. Add Build System Configuration

#### Add board option to Kconfig.projbuild

Open `main/Kconfig.projbuild` and add a new entry inside the `choice BOARD_TYPE` block:

```kconfig
choice BOARD_TYPE
    prompt "Board Type"
    default BOARD_TYPE_BREAD_COMPACT_WIFI
    help
        Board type.

    # ... other board options ...

    config BOARD_TYPE_MY_CUSTOM_BOARD
        bool "My Custom Board"
        depends on IDF_TARGET_ESP32S3  # adjust to your target chip
endchoice
```

**Notes:**
- Config name (`BOARD_TYPE_MY_CUSTOM_BOARD`) must be ALL_CAPS with underscores
- `depends on` specifies target chip (`IDF_TARGET_ESP32S3`, `IDF_TARGET_ESP32C3`, etc.)

#### Add board to CMakeLists.txt

Open `main/CMakeLists.txt` and add your board in the `elseif` chain:

```cmake
elseif(CONFIG_BOARD_TYPE_MY_CUSTOM_BOARD)
    set(BOARD_TYPE "my-custom-board")  # must match directory name
    set(BUILTIN_TEXT_FONT font_puhui_basic_20_4)  # choose font based on screen size
    set(BUILTIN_ICON_FONT font_awesome_20_4)
    set(DEFAULT_EMOJI_COLLECTION twemoji_64)  # optional, if emoji display needed
endif()
```

**Font and emoji selection by screen resolution:**
- Small (128x64 OLED): `font_puhui_basic_14_1` / `font_awesome_14_1`
- Small-medium (240x240): `font_puhui_basic_16_4` / `font_awesome_16_4`
- Medium (240x320): `font_puhui_basic_20_4` / `font_awesome_20_4`
- Large (480x320+): `font_puhui_basic_30_4` / `font_awesome_30_4`

Emoji collections:
- `twemoji_32` — 32×32 px (small screens)
- `twemoji_64` — 64×64 px (large screens)

### 5. Configure and Build

#### Method A: Manual with idf.py

1. **Set target chip** (first time or when switching chips):
   ```bash
   idf.py set-target esp32s3   # or esp32c3, esp32, etc.
   ```

2. **Clean old config:**
   ```bash
   idf.py fullclean
   ```

3. **Open config menu:**
   ```bash
   idf.py menuconfig
   ```
   Navigate to: `Xiaozhi Assistant` → `Board Type` and select your board.

4. **Build and flash:**
   ```bash
   idf.py build
   idf.py flash monitor
   ```

#### Method B: Using release.py script (recommended)

If your board directory has a `config.json`, use the release script for automated build:

```bash
python scripts/release.py my-custom-board
```

This script automatically:
- Reads the `target` from `config.json` and sets the target chip
- Applies `sdkconfig_append` build options
- Builds and packages the firmware

### 6. Create README.md

In the board's `README.md`, document: board features, hardware requirements, build and flash steps.

---

## Common Board Components

### 1. Displays

Supported display drivers include:
- ST7789 (SPI)
- ILI9341 (SPI)
- SH8601 (QSPI)
- and more...

### 2. Audio Codecs

Supported codecs include:
- ES8311 (common)
- ES7210 (mic array)
- AW88298 (power amplifier)
- and more...

### 3. Power Management

Some boards use PMICs:
- AXP2101
- Other available PMICs

### 4. MCP Device Control

Various MCP tools can be added to let the AI control:
- Speaker (volume control)
- Screen (brightness control)
- Battery (charge level readout)
- Light (LED control)
- and more...

## Board Class Hierarchy

- `Board` — Base board class
  - `WifiBoard` — Wi-Fi connected boards
  - `Ml307Board` — Boards using 4G module
  - `DualNetworkBoard` — Boards supporting Wi-Fi and 4G switching

## Development Tips

1. **Reference similar boards**: if your board resembles an existing one, study its implementation first
2. **Incremental debugging**: implement basic features (e.g., display) first, then add complex ones (e.g., audio)
3. **Pin mapping**: ensure all pin mappings in `config.h` are correct before building
4. **Hardware compatibility**: confirm all chips and drivers are compatible

## Common Issues

1. **Display not working**: check SPI config, mirror settings, and color inversion
2. **No audio output**: check I2S config, PA enable pin, and codec I2C address
3. **Cannot connect to network**: check Wi-Fi credentials and network config
4. **Cannot communicate with server**: check MQTT or WebSocket configuration

## References

- ESP-IDF docs: https://docs.espressif.com/projects/esp-idf/
- LVGL docs: https://docs.lvgl.io/
- ESP-SR docs: https://github.com/espressif/esp-sr
