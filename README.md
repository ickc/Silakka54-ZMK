# Silakka54 ZMK Configuration

A ZMK firmware configuration for the **Silakka54** split ergonomic keyboard.

## Overview

The Silakka54 is a 54-key split keyboard that uses the Lily58 shield in ZMK. Since the Lily58 has 58 keys, the 4 keys not present on the Silakka54 are mapped as `&none` (dead keys).

### Hardware

- **Controller**: nice!nano
- **Display**: nice!view
- **Layout**: Split ergonomic (54 keys)
- **Features**: Bluetooth, deep sleep, mouse emulation

## Layers

The keymap includes 5 layers:

| Layer | Name | Description |
|-------|------|-------------|
| 0 | `base` | Default QWERTY layout with home row mods |
| 1 | `esab` | Mirrored base layer (for one-handed typing) |
| 2 | `fn` | Function keys, media controls, Bluetooth, and mouse |
| 3 | `nf` | Mirrored function layer |
| 4 | `shift-esab` | Shifted mirrored layer |

## Features

### Home Row Mods

The configuration uses positional hold-tap behaviors for home row modifiers:

- **Bottom row**: Ctrl, Alt, Cmd, Shift (mirrored on both sides)
- **Timing**: 200ms tapping term with 150ms prior idle requirement
- **Flavor**: Balanced for reliable activation

### Advanced Behaviors

- `hml` / `hmr`: Home row mods (left/right) with opposite-hand trigger
- `mml` / `mmr`: Mirror mods for the mirrored layers
- `mll` / `mlr`: Mirror layer switching with hold-tap

### Macros

Pre-defined macros for productivity:

**Auto-pair characters:**
- Quotes, double quotes, braces, parentheses, brackets

**Windows shortcuts:**
- Cut, Copy, Paste, Undo, Select All
- Desktop, File Explorer, Snipping Tool
- Window tiling (Left/Right/Up/Down)
- Lock PC, Close Program, Settings

**Mac shortcuts:**
- Cut, Copy, Paste, Undo, Select All
- Mission Control, Spotlight Search
- Screen capture, Close Program
- Strikethrough text

### Mouse Emulation

The function layer includes mouse controls:
- Cursor movement (HJKL-style positioning)
- Left and right click
- Enabled via `CONFIG_ZMK_POINTING=y`

### Bluetooth

- 5 Bluetooth profiles (BT_SEL 0-4)
- Clear pairing with BT_CLR
- Extended TX power for better range

### Display

- nice!view e-ink display support
- Battery percentage indicator
- Custom status screen

### Power Management

- Deep sleep enabled (`CONFIG_ZMK_SLEEP=y`)
- Optimized for battery life

### ZMK Studio

ZMK Studio is enabled for real-time keymap editing via the `studio_unlock` key on the function layer.

## Building

Firmware is built automatically via GitHub Actions. Push to the repository to trigger a build.

The workflow generates firmware for:
- `lily58_left` with nice!view
- `lily58_right` with nice!view
- `settings_reset` (for clearing bond information)

### Manual Build

To build locally, follow the [ZMK documentation](https://zmk.dev/docs/development/setup) for setting up a development environment.

## Installation

1. Download the firmware artifacts from GitHub Actions
2. Put your nice!nano into bootloader mode (double-tap reset)
3. Copy the `.uf2` file to the mounted drive
4. Repeat for the other half

## File Structure

```
├── .github/
│   └── workflows/
│       └── build.yml       # GitHub Actions workflow
├── config/
│   ├── lily58.conf         # ZMK configuration options
│   ├── lily58.keymap       # Keymap definition
│   ├── macros.dtsi         # Macro definitions
│   └── west.yml            # West manifest
├── build.yaml              # Build matrix configuration
└── README.md
```

## Resources

- [ZMK Documentation](https://zmk.dev/docs)
- [ZMK Keycodes](https://zmk.dev/docs/keymaps/behaviors)
- [nice!nano Documentation](https://nicekeyboards.com/docs/nice-nano/)
- [nice!view Documentation](https://nicekeyboards.com/docs/nice-view/)

## License

This configuration is provided as-is for personal use with the Silakka54 keyboard.