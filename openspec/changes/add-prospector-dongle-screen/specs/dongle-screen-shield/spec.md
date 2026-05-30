## ADDED Requirements

### Requirement: YADS Module Import
The `config/west.yml` manifest SHALL import `janpfischer/zmk-dongle-screen` as a Zephyr module pinned to the `upgrade-4.1` branch, which is compatible with ZMK `main` (Zephyr 4.1).

#### Scenario: West manifest contains YADS project entry
- **WHEN** the project's `config/west.yml` is loaded by the ZMK build system
- **THEN** the `zmk-dongle-screen` module is resolved from `https://github.com/janpfischer/zmk-dongle-screen` at the `upgrade-4.1` branch

#### Scenario: YADS module is discoverable by ZMK shield search
- **WHEN** a build target includes `dongle_screen` in its shield list
- **THEN** ZMK's shield search path includes the YADS module's `boards/shields/dongle_screen/` directory

### Requirement: Dongle Build Target
The `build.yaml` file SHALL include a build target for `board: xiao_ble//zmk` with shield list `totem_dongle dongle_screen`, producing a UF2 firmware image for the Prospector dongle hardware.

#### Scenario: Dongle firmware is built via GitHub Actions
- **WHEN** the `build.yaml` is processed by ZMK's GitHub Actions workflow
- **THEN** an artifact named `totem_dongle-xiao_ble-zmk.uf2` is produced containing the dongle's firmware with YADS display support

#### Scenario: Dongle build target uses correct board format
- **WHEN** the dongle build target is evaluated
- **THEN** the board resolves to `xiao_ble` with ZMK board revision (`//zmk` suffix), matching ZMK main's modern board naming convention

### Requirement: Peripheral-Only Keyboard Halves
The existing `totem_left` and `totem_right` build targets SHALL be updated with `cmake-args` that set `CONFIG_ZMK_SPLIT=y` and `CONFIG_ZMK_SPLIT_ROLE_CENTRAL=n`, configuring them as peripheral-only devices that connect to the dongle.

#### Scenario: Left half builds as peripheral
- **WHEN** the `totem_left` build target is compiled
- **THEN** the resulting firmware has `CONFIG_ZMK_SPLIT_ROLE_CENTRAL` set to `n`, causing it to advertise as a BLE peripheral only

#### Scenario: Right half builds as peripheral
- **WHEN** the `totem_right` build target is compiled
- **THEN** the resulting firmware has `CONFIG_ZMK_SPLIT_ROLE_CENTRAL` set to `n`, causing it to advertise as a BLE peripheral only

#### Scenario: Existing non-dongle build remains possible
- **WHEN** a user removes the `cmake-args` lines from `totem_left` and `totem_right` build targets
- **THEN** the halves build as standalone split keyboard (one central, one peripheral) with no dongle dependency

### Requirement: Dongle Settings Reset
The `build.yaml` SHALL include a `settings_reset` build target for the `xiao_ble//zmk` board to allow clearing BLE bond data on the dongle.

#### Scenario: Dongle settings reset firmware is produced
- **WHEN** the `build.yaml` is processed
- **THEN** a `settings_reset-xiao_ble-zmk.uf2` artifact is produced that, when flashed to the dongle, clears all stored BLE pairings

### Requirement: Totem Dongle Shield Kconfig Definition
A `boards/shields/totem_dongle/Kconfig.shield` file SHALL define `SHIELD_TOTEM_DONGLE` as a boolean config active when `totem_dongle` is in the shield list. A corresponding `Kconfig.defconfig` SHALL set dongle role defaults: `ZMK_SPLIT_ROLE_CENTRAL=y`, `ZMK_SPLIT=y`, `ZMK_SPLIT_BLE_CENTRAL_PERIPHERALS=2`, `BT_MAX_CONN=7`, `BT_MAX_PAIRED=7`, and `ZMK_KEYBOARD_NAME="Totem"`.

#### Scenario: Shield config is activated when totem_dongle is listed
- **WHEN** the build system processes a shield list containing `totem_dongle`
- **THEN** `CONFIG_SHIELD_TOTEM_DONGLE` is set to `y`

#### Scenario: Dongle acts as split central with two peripherals
- **WHEN** `SHIELD_TOTEM_DONGLE` is active
- **THEN** the firmware is configured as a BLE central expecting two peripheral keyboard halves

### Requirement: Dongle Devicetree Overlay
The `boards/shields/totem_dongle/totem_dongle.overlay` file SHALL define a mock kscan node (the dongle has no physical keys), the Totem keyboard's matrix transform and physical layout (copied from `boards/shields/totem/totem.dtsi`), and select the mock kscan and physical layout in the `chosen` node.

#### Scenario: Dongle overlay provides mock kscan
- **WHEN** the dongle firmware boots
- **THEN** `zmk,kscan` resolves to a `zmk,kscan-mock` node with zero rows, columns, and events

#### Scenario: Dongle overlay includes Totem matrix transform
- **WHEN** the dongle processes key events from peripherals
- **THEN** the `zmk,matrix-transform` node maps 10 columns x 4 rows matching the Totem keyboard layout exactly

#### Scenario: Dongle overlay includes Totem physical layout
- **WHEN** ZMK Studio (or other layout-aware tool) queries the dongle
- **THEN** the physical layout node maps each logical key position to a physical coordinate on the Totem keyboard

### Requirement: Display Enablement Configuration
The `boards/shields/totem_dongle/totem_dongle.conf` file SHALL enable ZMK display support (`CONFIG_ZMK_DISPLAY=y`), select YADS custom status screen (`CONFIG_ZMK_DISPLAY_STATUS_SCREEN_CUSTOM=y`), enable central battery level fetching (`CONFIG_ZMK_SPLIT_BLE_CENTRAL_BATTERY_LEVEL_FETCHING=y`), and set display thread stack size to 4096 bytes.

#### Scenario: Display subsystem is activated
- **WHEN** the dongle firmware is built with `CONFIG_ZMK_DISPLAY=y`
- **THEN** LVGL, the ST7789V display driver, and YADS display modules are compiled into the firmware

#### Scenario: Battery level fetching is active on dongle
- **WHEN** the dongle is connected to keyboard peripherals
- **THEN** the dongle periodically polls peripheral battery levels and reports them to the YADS battery widget

### Requirement: ST7789V SPI Display Initialization
The dongle firmware SHALL initialize the ST7789V display via SPI3 using the pin configuration defined by YADS's `xiao_ble_zmk.overlay` board overlay: SCK on P1.13, MOSI on P1.15, CS on P0.9, DC on P0.7, RST on P0.3, with a MIPI DBI SPI 4-wire interface at up to 30 MHz. The display SHALL be configured at 240x280 resolution with RGB565 pixel format and YADS-provided ST7789V init register parameters.

#### Scenario: Display is selected as zephyr,display
- **WHEN** the `dongle_screen.overlay` is applied (imported from YADS module)
- **THEN** the `zephyr,display` chosen node points to the `st7789` device on `spi3`

#### Scenario: Display renders YADS widgets after boot
- **WHEN** the dongle boots and connects to at least one peripheral
- **THEN** the ST7789V display shows YADS status widgets (layer, modifier, output, WPM, battery) updated in real time

### Requirement: PWM Backlight Brightness Control
The dongle firmware SHALL control display backlight brightness via PWM1 channel 0 on pin P1.11, as defined by YADS's `disp_bl` PWM LED node. Backlight SHALL respond to brightness up/down keycodes (default F23/F24) and toggle on/off keycode (default F22). The backlight SHALL support idle timeout dimming (default 600s) and configurable max/min/default brightness levels.

#### Scenario: Backlight is active after boot
- **WHEN** the dongle firmware boots
- **THEN** the display backlight is set to `CONFIG_DONGLE_SCREEN_DEFAULT_BRIGHTNESS` (default: max brightness)

#### Scenario: Brightness increases via keyboard shortcut
- **WHEN** the user presses the brightness-up keycode (default F24)
- **THEN** the display backlight brightness increases by `CONFIG_DONGLE_SCREEN_BRIGHTNESS_STEP` (default 10), capped at `CONFIG_DONGLE_SCREEN_MAX_BRIGHTNESS` (default 80)

#### Scenario: Display toggles off via keyboard shortcut
- **WHEN** the user presses the display toggle keycode (default F22)
- **THEN** the display backlight turns off completely (brightness 0) and the screen is considered toggled off

#### Scenario: Display wakes on key activity after idle timeout
- **WHEN** the display has been dimmed due to idle timeout AND a key event is received from any paired peripheral
- **THEN** the display backlight is restored to the last active brightness level

### Requirement: APDS9960 Sensor Omission
The dongle firmware SHALL NOT enable the APDS9960 ambient light sensor. The `CONFIG_DONGLE_SCREEN_AMBIENT_LIGHT` Kconfig option SHALL remain at its default value (`n`), and no APDS9960-related Kconfig settings SHALL be specified in `totem_dongle.conf`.

#### Scenario: Sensor is absent from build
- **WHEN** the dongle firmware is compiled
- **THEN** neither the APDS9960 driver nor the I2C sensor polling logic is included in the binary

#### Scenario: Brightness operates in fixed/manual mode
- **WHEN** the dongle is in use without an ambient light sensor
- **THEN** backlight brightness is controlled exclusively by keyboard shortcuts and idle timeout logic, not by ambient light readings

### Requirement: README Documentation
The project `README.md` SHALL be updated with a "Dongle Setup" section documenting: the new build targets, the flashing procedure (settings_reset on all devices first, then dongle, then peripherals), the peripheral pairing sequence (left half first, then right), and the rollback procedure for returning to non-dongle operation.

#### Scenario: User follows README to set up dongle for the first time
- **WHEN** a new user reads the README and follows the dongle setup instructions in order
- **THEN** they can successfully flash all devices, pair the keyboard halves to the dongle, and see status information on the display

#### Scenario: User follows README to roll back to non-dongle mode
- **WHEN** an existing dongle user follows the rollback instructions
- **THEN** they can revert to standalone split keyboard operation without the dongle
