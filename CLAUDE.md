# CLAUDE.md - AI Assistant Guide for WLED

## Project Overview

WLED is a firmware for ESP32 and ESP8266 microcontrollers that controls addressable LEDs (WS2812B, SK6812, APA102, etc.) via a web interface. It supports 100+ lighting effects, multiple lighting protocols (E1.31, Art-Net, DDP, MQTT), and integrates with home automation systems (Alexa, Hue). Licensed under EUPL v1.2.

**Version:** 0.15.0-dev
**Upstream:** https://github.com/Aircoookie/WLED
**Docs:** https://kno.wled.ge

## Repository Structure

```
wled00/              Main firmware source (C/C++ for Arduino framework)
  data/              Web UI assets (HTML/CSS/JS) - compiled into C headers
  src/               Bundled library dependencies (ArduinoJson, E1.31, DMX, etc.)
usermods/            68+ modular extensions (sensors, displays, relays, etc.)
tools/               Node.js build tooling for web UI
  cdata.js           Inlines, minifies, gzips web UI into C header files
pio-scripts/         PlatformIO build scripts (Python)
lib/                 Additional libraries (ESP8266PWM)
images/              Project logos and images
.github/workflows/   CI/CD (build.yml, release.yml)
.devcontainer/       VS Code dev container config
```

## Build System

### PlatformIO (Firmware)

The primary build tool is PlatformIO. Configuration is in `platformio.ini`.

```bash
# Build for a specific environment
pio run -e esp32dev

# Build all default environments (17 targets)
pio run

# Upload to connected board
pio run -e esp32dev -t upload
```

**Default build environments:** nodemcuv2, esp8266_2m, esp01_1m_full, (160MHz variants), (compat variants), esp32dev, esp32_eth, lolin_s2_mini, esp32c3dev, esp32s3dev_16MB_opi, esp32s3dev_8MB_opi, esp32s3_4M_qspi, esp32_wrover

To build with custom settings, copy `platformio_override.sample.ini` to `platformio_override.ini` and modify as needed. This file is gitignored.

### Node.js (Web UI)

The web UI build compiles HTML/CSS/JS from `wled00/data/` into C header files (`wled00/html_*.h`) that are embedded in the firmware.

```bash
npm ci               # Install dependencies (Node.js >= 20 required)
npm run build        # Build web UI (runs tools/cdata.js)
npm run dev          # Watch mode - rebuilds on file changes
```

**Generated headers** (gitignored): `html_ui.h`, `html_pixart.h`, `html_cpal.h`, `html_pxmagic.h`, `html_settings.h`, `html_other.h`

### Python

```bash
pip install -r requirements.txt   # Installs PlatformIO and dependencies
```

## Running Tests

```bash
npm test             # Runs Node.js tests (tools/cdata-test.js) via `node --test`
pio test             # PlatformIO unit tests (limited)
```

CI runs both `npm test` (cdata.js tests) and firmware builds for all 17 environments.

## Code Style

### C/C++ (firmware in `wled00/`)

- **Indentation:** 2 spaces
- **Braces:** Opening brace on same line as condition (preferred)
- **Spacing:** Space between keyword and condition (`if (x)`), no space between function name and parens (`doStuff(a)`)
- **Comments:** Space after delimiter (`// comment`, not `//comment`)
- **Line length:** ~120 chars soft limit for comments
- Single-statement blocks may omit braces: `if (a == b) doStuff(a);`

### Web files (HTML/CSS/JS in `wled00/data/`)

- **Indentation:** Tabs

### Python (build scripts)

- **Indentation:** 2 spaces

## Key Source Files

### Core firmware (`wled00/`)

| File | Purpose |
|------|---------|
| `wled.h` | Main header - global definitions, feature flags, includes |
| `wled.cpp` | WLED main class, setup and loop |
| `FX.h` / `FX.cpp` | LED effect definitions and implementations (100+ effects) |
| `FX_fcn.cpp` / `FX_2Dfcn.cpp` | Effect helper functions and 2D matrix support |
| `bus_manager.cpp/.h` | Multi-output LED bus management |
| `json.cpp` | JSON API (REST) implementation |
| `cfg.cpp` | Configuration save/load |
| `set.cpp` | Settings management |
| `const.h` | Global constants and enumerations |
| `fcn_declare.h` | Forward declarations for all functions |
| `pin_manager.cpp/.h` | GPIO pin allocation and management |
| `udp.cpp` | UDP protocols (E1.31, Art-Net, TPM2.net, DDP) |
| `mqtt.cpp` | MQTT client integration |
| `ws.cpp` | WebSocket live control |
| `ir.cpp` | Infrared remote control |
| `alexa.cpp` | Amazon Alexa integration |
| `button.cpp` | Physical button input handling |
| `colors.cpp` | Color manipulation utilities |
| `presets.cpp` | Preset save/load/apply |
| `network.cpp` / `ntp.cpp` | WiFi, mDNS, NTP |
| `um_manager.cpp` | Usermod lifecycle manager |

### Web UI (`wled00/data/`)

| File | Purpose |
|------|---------|
| `index.htm` / `index.css` / `index.js` | Main control interface |
| `settings_leds.htm` | LED hardware configuration |
| `settings_wifi.htm` | WiFi/network settings |
| `settings_sync.htm` | Sync and protocol settings |
| `common.js` | Shared UI utilities |

## Architecture Notes

- **Segments:** LEDs are divided into independently-controllable segments, each with its own effect, colors, and palette.
- **Bus Manager:** Abstracts multiple LED output types (WS2812, APA102, PWM) behind a unified interface. Supports up to 10 outputs on ESP32.
- **Usermod System:** Extensions implement the `Usermod` class and register via `um_manager`. Each usermod lives in its own directory under `usermods/` with a `usermod_v2_*.h` header.
- **Feature Flags:** 120+ compile-time flags (prefixed `WLED_DISABLE_*` or `WLED_ENABLE_*`) control firmware size. Defined in `wled.h` and `platformio.ini`.
- **Config Storage:** JSON-based configuration stored on LittleFS filesystem. Presets and settings persist across reboots.
- **Web UI Pipeline:** HTML/CSS/JS files in `wled00/data/` are inlined, minified, gzipped, and converted to C byte arrays by `tools/cdata.js` during build.

## CI/CD

GitHub Actions workflows in `.github/workflows/`:

- **build.yml** - Compiles firmware for all 17 environments in parallel; runs `npm test`
- **release.yml** - Triggered on tag push; builds all environments and creates a draft GitHub release with `.bin` and `.bin.gz` artifacts
- **wled-ci.yml** - CI trigger configuration
- **stale.yml** - Marks inactive issues/PRs as stale

## Contributing Conventions

- PRs target the `0_15` branch
- Never force-push on open PRs
- PR descriptions should explain what, how, and testing performed
- Keep changes focused; avoid unrelated modifications
- Generated files (`wled00/html_*.h`) are gitignored - never commit them
- Custom board configs go in `platformio_override.ini` (gitignored)

## Common Tasks

### Adding a new LED effect

1. Define the effect mode constant in `FX.h`
2. Implement the effect function in `FX.cpp`
3. Register it in the mode array and add its name to the names string
4. Rebuild to test

### Adding a new usermod

1. Create a directory under `usermods/` (e.g., `usermods/my_feature/`)
2. Implement a class extending `Usermod` in a `usermod_v2_*.h` file
3. Register it in `usermods_list.cpp` or via `platformio_override.ini`
4. Include a `readme.md` in the usermod directory

### Modifying the web UI

1. Edit files in `wled00/data/`
2. Run `npm run build` to regenerate C headers (or use `npm run dev` for watch mode)
3. The generated `html_*.h` files are included by the firmware build automatically

### Building a custom firmware

1. Copy `platformio_override.sample.ini` to `platformio_override.ini`
2. Configure your board, features, and usermods
3. Run `pio run -e your_env`
