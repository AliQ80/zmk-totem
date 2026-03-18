# Totem ZMK Configuration

Custom ZMK firmware configuration for the [GEIGEIGEIST Totem](https://github.com/GEIGEIGEIST/totem) split keyboard with **Dual battery monitor and ZMK Studio support**.

## Features

- **Colemak-DH Matrix layout** optimized for ortholinear keyboards
- **Dual battery monitoring** - Reports battery levels for both keyboard halves
- **ZMK Studio unlock** for live keymap editing (not in original Totem config)
- **5 layers** optimized for Python and JavaScript development
- **Homerow mods** for comfortable modifier access
- **Mouse support** with scroll and movement controls
- **Combos** for quick access to ESC, dictation, and special characters

## Layers

### BASE - Colemak-DH

Main typing layer with homerow mods (GUI/Alt/Shift/Ctrl).

### CODE

Optimized symbol layer for Python/JavaScript:

- Brackets `[]` `()` on homerow for easy access
- Numbers arranged as numpad on right side
- Frequently-used symbols (`_`, `@`, `#`, `"`, `'`) in comfortable positions

### NAV

Navigation and mouse control layer with arrow keys, page navigation, and mouse movements.

### MOD

System controls including media keys, screen lock macros, and brightness/volume controls.

### ADJ

Function keys, Bluetooth device switching, and ZMK Studio unlock.

## Keymap Visualization

![Totem Keymap](./keymap-drawer/totem.svg)

## Installation

1. Fork this repository
2. Enable GitHub Actions in your fork
3. Modify `config/totem.keymap` as needed
4. Push changes to trigger automatic firmware build
5. Download firmware from Actions artifacts
6. Flash `totem_left-seeeduino_xiao_ble-zmk.uf2` to left half
7. Flash `totem_right-seeeduino_xiao_ble-zmk.uf2` to right half

## Hardware

- **Keyboard:** GEIGEIGEIST Totem (38-key split)
- **Controller:** Seeeduino XIAO BLE (nRF52840)
- **Firmware:** ZMK with Studio support

## Special Features

### Combos

- **Q + W:** ESC
- **N + M:** Dictation (Alt+Space)
- **U + Y:** ñ character

### Macros

- **Mac Lock:** Cmd+Ctrl+Q
- **Win Lock:** Win+L

### Battery Monitoring

Configured for split battery level reporting to support peripheral battery monitoring apps.

## Changing the Keyboard Name

To change the Bluetooth device name:

1. Edit `config/totem.conf` and set:

   ```
   CONFIG_ZMK_KEYBOARD_NAME="Your Custom Name"
   ```

2. Build the firmware (GitHub Actions will create 3 files including `settings_reset`)

3. Flash the firmware:
   - Flash `settings_reset-seeeduino_xiao_ble-zmk.uf2` to both halves
   - Flash `totem_left-seeeduino_xiao_ble-zmk.uf2` to left half
   - Flash `totem_right-seeeduino_xiao_ble-zmk.uf2` to right half

4. Clear Bluetooth bonds on the keyboard using `BT_CLR_ALL`

Note: The settings reset is required because the keyboard name is stored in persistent memory.

## Resources

### zmk-config

[Keymap Editor](https://nickcoutsos.github.io/keymap-editor/)

[Official ZMK site](https://zmk.dev/)

[Keyboard Tester](https://en.key-test.ru/)
