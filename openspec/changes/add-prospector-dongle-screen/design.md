## Context

The `zmk-totem` repository is a ZMK user-config repo providing firmware for a GEIGEIGEIST Totem split keyboard using Seeed Studio XIAO nRF52840 controllers. Currently, the left half acts as BLE central and the right half as peripheral. The project builds against ZMK `main` (Zephyr 4.1) using the modern `xiao_ble//zmk` board target format.

The goal is to add a dedicated BLE dongle based on the Prospector architecture: a XIAO nRF52840 paired with a Waveshare 1.69" 240x280 round-corner LCD driven by an ST7789V controller over SPI, running the YADS (Yet Another Dongle Screen) status display module with LVGL.

YADS's `main` branch targets ZMK v0.3 / Zephyr 3.5 and is incompatible with ZMK main. However, its `upgrade-4.1` branch (with community PR #32 merged) has been updated for Zephyr 4.1, including renamed board overlay files (`xiao_ble_zmk.overlay` instead of `seeeduino_xiao_ble.overlay`) and updated ZMK HID endpoint selection API. The `upgrade-4.1` branch is the target for this integration.

## Goals / Non-Goals

**Goals:**
- Add a ZMK shield definition (`totem_dongle`) that represents the Prospector dongle hardware
- Integrate YADS `dongle_screen` shield for ST7789V display with LVGL-based status widgets
- Configure ST7789V SPI display using YADS's built-in board overlay for `xiao_ble_zmk` (pins: SPI3 SCK=P1.13, MOSI=P1.15, CS=P0.9, DC=P0.7, RST=P0.3; PWM backlight on P1.11)
- Enable YADS backlight PWM control with brightness management via F-key keycodes (F22 toggle, F23 dim, F24 brighten)
- Configure idle timeout dimming, max/min brightness, and default brightness via Kconfig
- Convert existing Totem keyboard halves to peripheral-only operation
- Provide `settings_reset` build target for the dongle board
- Update README with dongle build/flash procedure and peripheral pairing sequence

**Non-Goals:**
- APDS9960 ambient light sensor integration (explicitly omitted per hardware spec)
- Hardware wiring guide (delegated to Prospector project documentation)
- Modifications to the Totem keymap itself (the existing keymap is shared between dongle and peripherals -- no changes needed)
- ZMK Studio support on the dongle (can be added later but not part of this change)
- One-sided (single-peripheral) dongle mode (the Totem uses two halves and needs `ZMK_SPLIT_BLE_CENTRAL_PERIPHERALS=2`)

## Decisions

### 1. Use YADS `upgrade-4.1` branch (not `main`)
**Why:** YADS `main` targets ZMK v0.3 / Zephyr 3.5 and fails to compile on ZMK main (Zephyr 4.1). The `upgrade-4.1` branch is specifically maintained for Zephyr 4.1 compatibility and includes PR #32 which renamed board overlays from `seeeduino_xiao_ble.overlay` to `xiao_ble_zmk.overlay` and updated the ZMK HID endpoint selection API.

**Alternatives considered:** Forking YADS and porting manually. Rejected -- the `upgrade-4.1` branch is actively maintained and has merge parity with the community's ZMK main branch fixes.

### 2. Board target format: `xiao_ble//zmk` (not `seeeduino_xiao_ble`)
**Why:** ZMK main (post Feb 2026) deprecated the `seeeduino_xiao_ble` board name in favor of `xiao_ble//zmk`. YADS PR #32 confirmed this. The project already uses `xiao_ble//zmk` in its existing `build.yaml`.

### 3. LVGL module via ZMK's Zephyr module system (not manual LVGL import)
**Why:** YADS depends on the LVGL Zephyr module, which ZMK's build system resolves automatically when `CONFIG_ZMK_DISPLAY=y` is set. No explicit `west.yml` entry for LVGL is needed -- the ZMK `app/west.yml` import already includes it. YADS's `zephyr/module.yml` declares `depends: [lvgl]`.

**Alternatives considered:** Adding a separate LVGL west project. Rejected -- ZMK's `app/west.yml` already imports LVGL at the correct version.

### 4. Display configuration: YADS Kconfig defaults with project-level overrides
**Why:** YADS's `Kconfig.defconfig` already provides sensible defaults for the ST7789V (RGB565 pixel format, 16-bit color depth with swap, LVGL config for VDB size, memory pool, DPI). The `totem_dongle.conf` file will only need to set display enablement flags and custom brightness/timeout values. ST7789V init parameters (vcom, gamma, gate control, etc.) are provided by YADS's `xiao_ble_zmk.overlay` devicetree file.

### 5. Omit APDS9960 ambient light sensor
**Why:** The user's hardware omits this component. YADS's `Kconfig.defconfig` defaults `CONFIG_DONGLE_SCREEN_AMBIENT_LIGHT=n`, so no action is needed -- the sensor is opt-in only. The I2C1 bus and APDS9960 devicetree node in YADS's overlay will remain in the devicetree but never be enabled since the Kconfig is off and the sensor is absent.

### 6. Backlight control: PWM via `pwm-leds` (YADS native)
**Why:** YADS's `xiao_ble_zmk.overlay` defines a `pwmleds` node with `disp_bl` label connected to `pwm1` channel 0 on pin P1.11. The YADS brightness module (`src/brightness.c`) uses the `disp_bl` LED label to control backlight brightness. This is the native YADS approach and requires no additional configuration beyond what YADS already provides.

### 7. Dongle Kconfig: Two peripherals (not one)
**Why:** The Totem is a split keyboard with two halves. The dongle must connect to both. Setting `ZMK_SPLIT_BLE_CENTRAL_PERIPHERALS=2` and `BT_MAX_CONN=7` (2 peripherals + 5 BT profiles) ensures both halves can pair.

**Alternatives considered:** Using `ZMK_SPLIT_BLE_CENTRAL_PERIPHERALS=1` and having only one half work. Rejected -- both halves are needed.

### 8. Keymap sharing: No dongle-specific keymap needed
**Why:** The dongle acts purely as a BLE-to-USB bridge and display host. It receives key events from peripherals and has no physical keys of its own. The existing `totem.keymap` applies to all builds. The YADS screen toggle and brightness keys (F22, F23, F24) are already valid keycodes in the keymap system -- the user can add them to a layer if desired, but this is optional and outside the scope of this change.

## Risks / Trade-offs

| Risk | Mitigation |
|------|------------|
| YADS `upgrade-4.1` branch diverges from `main` or becomes stale | Pin to a specific commit SHA instead of branch name in `west.yml` for reproducible builds. The design defaults to branch name for simplicity but recommends SHA pinning for production. |
| ZMK main branch API changes break YADS compatibility again | Community track record (issue #29, PR #32) shows responsive maintenance. Pin west revision to a known-good ZMK commit. |
| LVGL memory pool too small for 240x280 display with double VDB | YADS default `LV_Z_MEM_POOL_SIZE=16384` with 16-bit color and `LV_Z_VDB_SIZE=50` should be sufficient. If rendering issues occur, increase pool size via `totem_dongle.conf`. |
| PWM backlight conflicts with other PWM usage on XIAO | XIAO nRF52840 has PWM1 free (SPI display uses SPI3, I2C uses I2C0/I2C1). No conflict expected. |
| Pairing sequence critical for battery widget ordering | Document the pairing procedure (left half first, then right) in the README. This is a YADS requirement, not a bug. |
| Settings reset required on all devices when transitioning from non-dongle setup | Document that `settings_reset` must be flashed on both halves AND dongle before first use. |

## Migration Plan

1. **Before deploying:** Flash `settings_reset` firmware to both Totem halves to clear existing BLE bonds.
2. **Deploy dongle:** Build and flash the `totem_dongle dongle_screen` UF2 image.
3. **Deploy peripherals:** Build and flash the updated `totem_left` and `totem_right` images (now with `CONFIG_ZMK_SPLIT_ROLE_CENTRAL=n`).
4. **Pair:** Power on left half first, wait for dongle battery indicator, then power on right half.
5. **Rollback:** To revert to non-dongle operation, remove the `cmake-args: -DCONFIG_ZMK_SPLIT=y -DCONFIG_ZMK_SPLIT_ROLE_CENTRAL=n` lines from the keyboard half build targets, flash all devices with `settings_reset`, then flash the reverted firmware.

## Open Questions

1. **YADS branch lifecycle:** Should we pin to a specific YADS commit SHA or the `upgrade-4.1` branch? **Recommendation:** Pin to branch initially for flexibility during development; switch to SHA before merging to main.
2. **ZMK Studio on dongle:** The user's keyboard config has ZMK Studio disabled. If Studio is needed on the dongle in the future, the dongle overlay needs the `zmk,studio` chosen node and `zmk-studio` snippet. Deferred.
3. **Physical key positions for ZMK Studio:** The dongle overlay will include the full `totem_physical_layout` with key positions copied from `totem.dtsi`. If Studio integration is not needed, a simplified layout (without `keys` property) suffices. **Decision:** Include full keys for future Studio readiness.

## Implementation Notes (Post-Build #27, #28)

### Issue #1: Missing `#include <physical_layouts.dtsi>` (Build #27, #28)

**Symptom:** "Missing ZMK Compat: The selected board is not set up for ZMK and there is a ZMK variant available."

**Root Cause:** `totem_dongle.overlay` uses `&key_physical_attrs` in the physical layout `keys` section (copied from `totem.dtsi`) but is missing the necessary include. Compare:

```
totem.dtsi:          #include <dt-bindings/zmk/matrix_transform.h>
                     #include <physical_layouts.dtsi>   ← PRESENT: defines key_physical_attrs
totem_dongle.overlay: #include <dt-bindings/zmk/matrix_transform.h>
                     ← MISSING: key_physical_attrs is undefined!
```

Without this include, `west build` fails during devicetree processing. The Kconfig phase runs first and writes a partial `.config`, but the failure prevents `CONFIG_ZMK_BOARD_COMPAT=y` from being resolved. The GitHub Actions post-build compatibility check then detects the missing symbol and reports the misleading "Missing ZMK Compat" error.

**Resolution:** Add `#include <physical_layouts.dtsi>` to `totem_dongle.overlay`.

### Issue #2: Duplicate Display Hardware in Board Overlay (Build #28)

**Symptom:** Same "Missing ZMK Compat" error persists after adding `boards/xiao_ble_zmk.overlay`.

**Root Cause:** The `boards/xiao_ble_zmk.overlay` inside `totem_dongle` duplicated the ST7789V display hardware configuration (`/mipi_dbi/st7789v@0`, SPI3 pinout, PWM1 backlight) already provided by YADS's `dongle_screen/boards/xiao_ble_zmk.overlay`. When both shields are loaded for the same board, Zephyr encounters a devicetree node conflict (same node path defined twice).

**Resolution:** Delete `boards/shields/totem_dongle/boards/xiao_ble_zmk.overlay`. The `totem_dongle` shield's overlay should ONLY contain keyboard-specific configuration (mock kscan, matrix transform, physical layout). Display hardware configuration comes exclusively from YADS's `dongle_screen` shield.

### Issue #3: Unnecessary `.zmk.yml` (Build #27, #28)

**Root Cause:** `totem_dongle.zmk.yml` with a `compatibility` section was added unnecessarily. ZMK shield detection uses `Kconfig.shield` (`$(shields_list_contains,totem_dongle)`), not `.zmk.yml`. An incorrectly formatted `.zmk.yml` may interfere with ZMK's shield resolution and board variant detection.

**Resolution:** Delete `totem_dongle.zmk.yml`. Other shields in this project (totem_left, totem_right) function correctly without one.

### Architecture Clarification: Shield Layering

When `shield: totem_dongle dongle_screen` is specified, ZMK layers both shields:
- `totem_dongle` provides: keyboard-specific config (Kconfig role defaults, mock kscan, matrix transform, physical layout)
- `dongle_screen` provides: display hardware (SPI3 clock/pins, ST7789V init params, PWM backlight, LVGL integration)

Each shield handles what it's responsible for. The custom dongle shield should NOT duplicate hardware configuration from the display shield.
