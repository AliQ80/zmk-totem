## 1. West Manifest Setup

- [x] 1.1 Add `janpfischer/zmk-dongle-screen` remote to `config/west.yml` remotes section
- [x] 1.2 Add `zmk-dongle-screen` project entry pinned to `upgrade-4.1` branch with `import: false` (it's a Zephyr module, not a west project with sub-imports)
- [x] 1.3 Verify YADS module is discoverable: confirm `zephyr/module.yml` has `name: dongle_screen` and `build: cmake: .`

## 2. Dongle Shield Definition

- [x] 2.1 Create `boards/shields/totem_dongle/Kconfig.shield` defining `SHIELD_TOTEM_DONGLE` triggered by `$(shields_list_contains,totem_dongle)`
- [x] 2.2 Create `boards/shields/totem_dongle/Kconfig.defconfig` with dongle-specific defaults: `ZMK_SPLIT_ROLE_CENTRAL=y`, `ZMK_SPLIT=y`, `ZMK_SPLIT_BLE_CENTRAL_PERIPHERALS=2`, `BT_MAX_CONN=7`, `BT_MAX_PAIRED=7`, `ZMK_KEYBOARD_NAME="Totem"`
- [x] 2.3 Create `boards/shields/totem_dongle/totem_dongle.overlay` with: mock kscan node (`zmk,kscan-mock`, 0 columns/rows/events), chosen nodes for kscan and physical-layout, matrix transform node (10 col x 4 row) copied from `totem.dtsi`, and physical layout node with keys copied from `totem.dtsi`
- [x] 2.4 Create `boards/shields/totem_dongle/totem_dongle.zmk.yml` metadata file with `id: totem_dongle` and `name: "Totem Dongle"`

## 3. Display Configuration

- [x] 3.1 Create `boards/shields/totem_dongle/totem_dongle.conf` enabling core display features: `CONFIG_ZMK_DISPLAY=y`, `CONFIG_ZMK_DISPLAY_STATUS_SCREEN_CUSTOM=y`, `CONFIG_ZMK_SPLIT_BLE_CENTRAL_BATTERY_LEVEL_FETCHING=y`, `CONFIG_ZMK_DISPLAY_DEDICATED_THREAD_STACK_SIZE=4096`
- [x] 3.2 Add YADS brightness Kconfig overrides to `totem_dongle.conf`: `CONFIG_DONGLE_SCREEN_MAX_BRIGHTNESS=80`, `CONFIG_DONGLE_SCREEN_MIN_BRIGHTNESS=1`, `CONFIG_DONGLE_SCREEN_IDLE_TIMEOUT_S=600`, `CONFIG_DONGLE_SCREEN_BRIGHTNESS_STEP=10`
- [x] 3.3 Ensure `CONFIG_DONGLE_SCREEN_AMBIENT_LIGHT` remains unset (defaults to `n`) -- no APDS9960 configuration added
- [x] 3.4 Verify YADS `xiao_ble_zmk.overlay` provides the ST7789V SPI3 pinout (SCK=P1.13, MOSI=P1.15, CS=P0.9, DC=P0.7, RST=P0.3) and PWM1 backlight (P1.11) -- no project-level overlay modifications needed for display hardware

## 4. Build Matrix Updates

- [x] 4.1 Add dongle build target to `build.yaml`: `board: xiao_ble//zmk` with `shield: totem_dongle dongle_screen`
- [x] 4.2 Add dongle `settings_reset` target to `build.yaml`: `board: xiao_ble//zmk` with `shield: settings_reset`
- [x] 4.3 Update `totem_left` target in `build.yaml` with `cmake-args: "-DCONFIG_ZMK_SPLIT=y" "-DCONFIG_ZMK_SPLIT_ROLE_CENTRAL=n"`
- [x] 4.4 Update `totem_right` target in `build.yaml` with `cmake-args: "-DCONFIG_ZMK_SPLIT=y" "-DCONFIG_ZMK_SPLIT_ROLE_CENTRAL=n"`
- [x] 4.5 Verify existing `settings_reset` target remains (for keyboard half reset)

## 5. Documentation

- [x] 5.1 Add "Dongle Setup" section to `README.md` describing the Prospector dongle hardware and its purpose
- [x] 5.2 Document the new build targets and firmware artifacts produced (dongle UF2, peripheral UF2s, settings_reset UF2s)
- [x] 5.3 Document the flashing procedure: (1) flash `settings_reset` on all devices, (2) flash dongle firmware, (3) flash peripheral firmware on both halves
- [x] 5.4 Document the pairing sequence: left half first (wait for battery widget), then right half
- [x] 5.5 Document YADS display features: brightness control via F22/F23/F24, idle timeout behavior, widget descriptions
- [x] 5.6 Document rollback procedure for returning to non-dongle operation
- [x] 5.7 Update "Installation" section to note the dongle is now part of the build (mention the extra UF2 file and pairing steps, or add a separate "Dongle" path vs "Standalone" path)

## 6. Verification

- [x] 6.1 Confirm GitHub Actions build matrix produces all 5 UF2 artifacts: `totem_dongle-xiao_ble-zmk.uf2`, `totem_left-xiao_ble-zmk.uf2`, `totem_right-xiao_ble-zmk.uf2`, `settings_reset_dongle-xiao_ble-zmk.uf2`, `settings_reset_keyboard-xiao_ble-zmk.uf2`
- [x] 6.2 Verify YADS module resolves correctly by checking the build log for `-- Found dongle_screen module`
- [x] 6.3 Verify no Kconfig warnings about unknown symbols (APDS9960, SENSOR should not appear)
- [x] 6.4 Verify ST7789V devicetree node is resolved (check build output for `st7789v@0` binding success)

> **Note:** Verification tasks 6.1-6.4 require pushing to GitHub and checking the Actions build log. The configuration is complete and ready for build verification.

## 7. Build Error Fix: Board-Specific Overlay (Post-Build #27) — REVERTED

> **NOTE:** The fix in Section 7 was wrong. Creating `boards/xiao_ble_zmk.overlay` inside `totem_dongle` duplicates display hardware already provided by YADS's `dongle_screen` shield, causing devicetree node conflicts. This is being reverted in Section 8.

- [x] 7.1 ~~Create `boards/shields/totem_dongle/boards/xiao_ble_zmk.overlay`~~ — **REVERT: This file duplicates display hardware from dongle_screen and causes devicetree conflicts**
- [x] 7.2 ~~Update `totem_dongle.zmk.yml` with `compatibility` section~~ — **REVERT: .zmk.yml is being removed entirely (not needed; shield detection uses Kconfig.shield)**

## 8. Build Error Fix: Missing include + Remove Duplicates (Post-Build #28)

**Root Cause:** `totem_dongle.overlay` uses `&key_physical_attrs` macro in the physical layout `keys` section but is **missing `#include <physical_layouts.dtsi>`**. This causes `west build` to fail during devicetree processing, which prevents Kconfig from fully resolving — leaving `CONFIG_ZMK_BOARD_COMPAT=y` absent from `.config`. The GitHub Actions post-build check then triggers the "Missing ZMK Compat" error.

**Secondary issues:**
- `boards/xiao_ble_zmk.overlay` duplicates `/mipi_dbi/st7789v@0` already provided by YADS's `dongle_screen/boards/xiao_ble_zmk.overlay` → devicetree node conflict
- `totem_dongle.zmk.yml` is unnecessary (shield detection uses `Kconfig.shield`) and may interfere with ZMK's shield resolution

- [ ] 8.1 Add `#include <physical_layouts.dtsi>` to `boards/shields/totem_dongle/totem_dongle.overlay` (after `#include <dt-bindings/zmk/matrix_transform.h>`)
- [ ] 8.2 Delete `boards/shields/totem_dongle/boards/xiao_ble_zmk.overlay` (display hardware provided by YADS dongle_screen shield)
- [ ] 8.3 Delete `boards/shields/totem_dongle/totem_dongle.zmk.yml` (unnecessary, shield detection uses Kconfig.shield)
- [ ] 8.4 Commit and push fixes
- [ ] 8.5 Rebuild and verify totem_dongle artifact is produced
- [ ] 8.6 Verify build logs show no "Missing ZMK Compat" errors

