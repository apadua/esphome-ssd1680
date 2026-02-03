# CLAUDE.md - AI Assistant Guide for ESPHome SSD1680 Component

## Project Overview

This repository contains a custom ESPHome component for controlling SSD1680-based e-paper displays. The primary target hardware is the **Elecrow CrowPanel ESP32 2.9" E-Paper HMI Display** (ESP32-S3 based).

**Repository:** https://github.com/apadua/esphome-ssd1680
**License:** MIT (Copyright 2026 Andre Padua)

### Key Features
- Full driver for SSD1680 e-paper controller
- Handles inverted pixel polarity quirk
- Deferred initialization for reliable debugging
- Watchdog-friendly long refresh operations
- Works with or without BUSY pin

## Directory Structure

```
esphome-ssd1680/
├── CLAUDE.md                    # This file
├── .gitignore                   # Repository-wide ignore rules
└── components/
    └── ssd1680_epaper/          # ESPHome component package
        ├── __init__.py          # Empty marker file (required by ESPHome)
        ├── display.py           # Python config schema & code generation
        ├── ssd1680_epaper.h     # C++ header - class interface
        ├── ssd1680_epaper.cpp   # C++ implementation - driver logic
        ├── crowpanel-clock.yaml # Complete example configuration
        ├── README.md            # User documentation
        ├── LICENSE              # MIT License
        └── .gitignore           # Component-level ignores
```

## Architecture

### Component Model (Three-Layer Design)

1. **Python Layer** (`display.py`): Validates YAML configuration and generates C++ initialization code
2. **C++ Header** (`ssd1680_epaper.h`): Defines class interface inheriting from ESPHome base classes
3. **C++ Implementation** (`ssd1680_epaper.cpp`): Contains driver logic and hardware communication

### Class Hierarchy

```cpp
SSD1680EPaper : public display::DisplayBuffer,
                public spi::SPIDevice<BIT_ORDER_MSB_FIRST, CLOCK_POLARITY_LOW,
                                      CLOCK_PHASE_LEADING, DATA_RATE_4MHZ>
```

The class inherits from:
- `DisplayBuffer` - Provides drawing primitives and buffer management
- `SPIDevice` - Provides SPI communication (4 MHz, Mode 0)

## Key Files

### `display.py` (Configuration Layer)

- **DEPENDENCIES:** `["spi"]`
- **CONFIG_SCHEMA:** Defines valid YAML options
  - `dc_pin` (required) - Data/Command control GPIO
  - `reset_pin` (optional) - Hardware reset GPIO
  - `busy_pin` (optional) - Status signal GPIO
  - Default polling interval: 60 seconds
- **`to_code()`:** Async function generating C++ initialization

### `ssd1680_epaper.h` (Interface)

- Display dimensions: 128x296 pixels
- Display type: Binary (black/white only)
- Key methods: `setup()`, `update()`, `dump_config()`
- Protected methods for SPI commands and display control

### `ssd1680_epaper.cpp` (Implementation)

Key implementation details:

| Function | Purpose |
|----------|---------|
| `setup()` | Enables GPIO7 power, initializes pins/SPI, allocates buffer |
| `init_display_()` | Sends initialization command sequence to display |
| `display_frame_()` | Writes buffer to display RAM with pixel inversion |
| `full_update_()` | Triggers refresh using 0xF7 sequence |
| `update()` | Called on polling interval, handles deferred init |

## Critical Hardware Details

### CrowPanel-Specific Requirements

1. **GPIO7 Power Control**: MUST be set HIGH or display won't work
   ```cpp
   gpio_set_level(GPIO_NUM_7, 1);  // Required!
   ```

2. **Pin Assignments**:
   | Function | GPIO |
   |----------|------|
   | SPI CLK  | 12   |
   | SPI MOSI | 11   |
   | CS       | 45   |
   | DC       | 46   |
   | RESET    | 47   |
   | BUSY     | 48   |

### Display Quirks

1. **Inverted Pixel Polarity**: Data must be XORed before sending
   ```cpp
   this->data_(~this->buffer_[i]);  // Line 322 - critical!
   ```

2. **BUSY Pin Behavior**: May not go LOW reliably after refresh. The driver handles this gracefully with timeouts.

3. **Refresh Time**: Full refresh takes 2-4 seconds (5 second timeout is normal)

### SSD1680 Commands Used

| Command | Purpose |
|---------|---------|
| 0x12 | Software Reset |
| 0x01 | Driver Output Control |
| 0x11 | Data Entry Mode |
| 0x18 | Temperature Sensor |
| 0x22/0x20 | Display Update Control |
| 0x24 | Write B/W RAM |
| 0x26 | Write RED RAM |
| 0x3C | Border Waveform |
| 0x44/0x45 | Set RAM Address |
| 0x4E/0x4F | Set RAM Counters |
| 0xF7 | Full refresh with internal LUT |

## Development Workflow

### Adding to an ESPHome Project

```yaml
external_components:
  - source: github://apadua/esphome-ssd1680
    components: [ssd1680_epaper]
```

### Local Development

1. Clone the repository
2. Edit files in `components/ssd1680_epaper/`
3. Test by pointing ESPHome to local path:
   ```yaml
   external_components:
     - source:
         type: local
         path: /path/to/esphome-ssd1680/components
   ```

### Build Process

1. ESPHome parses YAML configuration
2. `display.py` validates against `CONFIG_SCHEMA`
3. `to_code()` generates C++ initialization
4. ESP-IDF compiles C++ with platformio
5. Binary uploaded to ESP32-S3

## Coding Conventions

### C++ Style

- **Namespace:** `esphome::ssd1680_epaper`
- **Method naming:** `snake_case` with trailing underscore for private methods
- **Logging:** Use `ESP_LOGI()`, `ESP_LOGD()`, `ESP_LOGE()` macros
- **Tag:** `static const char *const TAG = "ssd1680_epaper";`

### Python Style

- Follow ESPHome configuration patterns
- Use `cv.*` validators for schema validation
- Async `to_code()` with `cg.*` code generation

### YAML Style

- Standard ESPHome format
- Use `id()` for component references
- Use `!secret` for sensitive values
- Use `gfonts://` for Google Fonts

## Testing

### Minimal Test Configuration

```yaml
display:
  - platform: ssd1680_epaper
    id: epaper
    cs_pin: GPIO45
    dc_pin: GPIO46
    reset_pin: GPIO47
    busy_pin: GPIO48
    lambda: |-
      it.fill(COLOR_OFF);
      it.print(0, 0, id(font), "Hello World");
```

### Debug Logging

The driver includes extensive logging. Enable debug logging in ESPHome:
```yaml
logger:
  level: DEBUG
```

Look for log lines starting with `[ssd1680_epaper]`.

## Common Issues and Solutions

| Issue | Cause | Solution |
|-------|-------|----------|
| Display blank | GPIO7 not enabled | Verify GPIO7 HIGH (handled in setup) |
| Inverted colors | Pixel polarity | Already handled via XOR in driver |
| Update timeout warnings | Normal BUSY behavior | Safe to ignore - display still works |
| Ghosting/artifacts | E-paper characteristic | Use full refresh (already implemented) |

## Dependencies

### ESPHome Components (Required)
- `spi` - SPI bus communication
- `display` - Display buffer and drawing primitives

### External Libraries
- None required - uses ESPHome framework only

### ESP-IDF APIs Used
- `driver/gpio.h` - For GPIO7 power control

## File Metrics

| File | Lines | Purpose |
|------|-------|---------|
| `ssd1680_epaper.cpp` | 405 | Core driver implementation |
| `display.py` | 55 | Configuration schema |
| `ssd1680_epaper.h` | 47 | Class interface |
| `crowpanel-clock.yaml` | 142 | Working example |
| `README.md` | 262 | User documentation |

## Making Changes

### Modifying Display Initialization

Edit `init_display_()` in `ssd1680_epaper.cpp`. The sequence follows the SSD1680 datasheet.

### Adding Configuration Options

1. Add constant to `display.py` imports or define new one
2. Add to `CONFIG_SCHEMA`
3. Handle in `to_code()` function
4. Add setter in `.h` file
5. Use value in `.cpp` implementation

### Changing Display Dimensions

Modify these locations:
- `ssd1680_epaper.cpp`: `WIDTH` and `HEIGHT` constants (lines 13-14)
- `ssd1680_epaper.h`: `get_width_internal()` and `get_height_internal()` return values

## Git Workflow

- Main branch: `main`
- Feature branches: `claude/` prefix for AI-assisted development
- Commit messages: Descriptive, explaining what changed

## Resources

- [SSD1680 Datasheet](https://www.good-display.com/companyfile/101.html)
- [ESPHome Custom Components](https://esphome.io/custom/custom_component.html)
- [ESPHome Display Component](https://esphome.io/components/display/index.html)
- [Elecrow CrowPanel](https://www.elecrow.com/esp32-display-2-9-inch-296-128-e-paper-hmi-display.html)
