# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a ZMK firmware configuration for **roBa**, a custom split wireless keyboard. The right half has a PMW3610 trackball sensor; both halves use Seeeduino XIAO BLE controllers. There is no local build toolchain — all firmware is compiled via GitHub Actions.

## Building Firmware

Firmware builds are triggered automatically on push/PR via GitHub Actions using the ZMK hosted workflow:

```yaml
# .github/workflows/build.yml
uses: zmkfirmware/zmk/.github/workflows/build-user-config.yml@v0.3-branch
```

The build matrix in [build.yaml](build.yaml) produces three artifacts:
- `seeeduino_xiao_ble` + `roBa_R` (with `studio-rpc-usb-uart` snippet) — right half (central)
- `seeeduino_xiao_ble` + `roBa_L` — left half (peripheral)
- `seeeduino_xiao_ble` + `settings_reset` — BLE bond reset utility

To regenerate the keymap visualization SVG, manually trigger the **Draw Keymap** workflow in GitHub Actions. It reads `config/*.keymap` and outputs to `keymap-drawer/`.

## Architecture

### Key Files

| File | Purpose |
|------|---------|
| [config/roBa.keymap](config/roBa.keymap) | All keymap logic: layers, combos, behaviors, macros |
| [config/west.yml](config/west.yml) | ZMK dependency manifest (zmk v0.3-branch, pmw3610 driver) |
| [boards/shields/roBa/roBa.dtsi](boards/shields/roBa/roBa.dtsi) | Shared hardware: kscan matrix (4 rows × 11 cols), encoders, trackball listener |
| [boards/shields/roBa/roBa_L.overlay](boards/shields/roBa/roBa_L.overlay) | Left half: 6 col-gpios, encoder enabled |
| [boards/shields/roBa/roBa_R.overlay](boards/shields/roBa/roBa_R.overlay) | Right half: 5 col-gpios, col-offset=6, PMW3610 trackball via SPI |

### Split Design

- **Right half = central** (`ZMK_SPLIT_ROLE_CENTRAL`): USB connection, trackball, ZMK Studio RPC
- **Left half = peripheral**: encoder only
- Matrix is 4 rows × 11 columns total; right overlay uses `col-offset = <6>` to address columns 6–10

### Layers

Layer numbers are defined as macros at the top of [config/roBa.keymap](config/roBa.keymap):

| Define | Number | Purpose |
|--------|--------|---------|
| `DEFAULT_L` | 0 | Base QWERTY layer |
| `SYMBOL_L` | 1 | Symbols, brackets |
| `MOUSE_CTRL_L` | 2 | Mouse buttons, arrows, clipboard |
| `NUM_FUNCTION_L` | 3 | Numbers (top row), F-keys |
| `MOUSE_L` | 4 | Mouse buttons only |
| `SCROLL_L` | 5 | Trackball scroll mode (triggered by `scroll-layers = <5>` in PMW3610) |
| `BT_L` | 6 | Bluetooth profile selection and clearing |

### Japanese Key Mappings

All `JP_*` defines at the top of the keymap translate US keycodes to produce correct characters under a Japanese OS keyboard layout (e.g., `JP_AT` → `LEFT_BRACKET`, `JP_COLON` → `SINGLE_QUOTE`). Always use these defines when adding Japanese-layout symbols to the keymap.

### Custom Behaviors

- `lt_to_layer_0` — hold-tap that activates a layer on hold and returns to layer 0 on tap
- `long_hold_mod_tap` (350ms, tap-preferred) — for shift/Z and shift/slash
- `medium_hold_mod_tap` (250ms, balanced) — general mod-tap
- `scroll_up_down` / `mouse_scroll` — encoder bindings for scroll wheel
- `mo2`–`moI` — mod-morph behaviors for shifted symbol variants
