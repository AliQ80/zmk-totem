## Why

The Totem split keyboard currently operates in BLE peripheral mode on both halves, with one half acting as the central. Adding a dedicated BLE dongle (Prospector architecture: Seeed Studio XIAO nRF52840 + Waveshare 1.69" LCD) as the central receiver improves battery life on both keyboard halves and reduces wireless latency. The YADS (Yet Another Dongle Screen) module provides a real-time status display showing active layer, modifiers, WPM, BLE output state, and battery levels for both halves -- all of which are currently invisible to the user. With ZMK `main` now on Zephyr 4.1, a compatible integration path exists via YADS's `upgrade-4.1` branch and the new `xiao_ble//zmk` board target format.

## What Changes

- **West manifest**: Add `janpfischer/zmk-dongle-screen` module pinned to the `upgrade-4.1` branch for Zephyr 4.1 / ZMK main compatibility, plus required LVGL dependency.
- **Build matrix**: Introduce new build targets for `totem_dongle` shield (dongle hardware definition) with `dongle_screen` shield (YADS display module), using the `xiao_ble//zmk` board target. Convert keyboard halves to peripheral-only via cmake-args.
- **Dongle shield**: Create new `boards/shields/totem_dongle/` directory with Kconfig files, devicetree overlay defining the dongle's mock kscan + matrix transform + physical layout (inherited from existing Totem keyboard definition), and a `.conf` file with YADS display configuration.
- **Display configuration**: Provide Kconfig defaults for the ST7789V SPI display (YADS native backlight PWM control via `pwm-leds`, brightness management via keyboard F-keys, idle timeout), explicitly omitting the APDS9960 ambient light sensor.
- **Peripheral conversion**: Update existing `totem_left` and `totem_right` build targets with `CONFIG_ZMK_SPLIT_ROLE_CENTRAL=n` cmake-args so they operate as peripheral-only halves.

## Capabilities

### New Capabilities
- `dongle-screen-shield`: Defines the Prospector dongle hardware (XIAO nRF52840 + ST7789V display) as a ZMK shield, imports YADS module for LVGL-based status screen rendering, and integrates with the existing Totem keyboard configuration.

### Modified Capabilities
- *(none -- this is a greenfield addition; no existing specs are modified)*

## Impact

- **config/west.yml**: Adds two new project entries (zmk-dongle-screen + lvgl module) with `upgrade-4.1` branch pinning
- **build.yaml**: Adds `totem_dongle dongle_screen` build target; modifies existing `totem_left`/`totem_right` targets with peripheral cmake-args; adds dongle `settings_reset` target
- **boards/shields/totem_dongle/** (new): Kconfig.shield, Kconfig.defconfig, totem_dongle.conf, totem_dongle.overlay, totem_dongle.zmk.yml metadata
- **Keymap**: No changes required -- keymap is shared between dongle and peripherals
- **README.md**: Updated with dongle build/flash instructions and pairing procedure
