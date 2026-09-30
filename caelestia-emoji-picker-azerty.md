# Caelestia Emoji Picker Shortcut for AZERTY Keyboard

## 1. Overview

This document details the resolution for the emoji and glyph picker keyboard shortcut (`SUPER + .`) failing on French AZERTY keyboard layouts within Caelestia's Hyprland desktop environment on `framboise`.

- **Affected Components**: Caelestia Shell (`caelestia emoji -p`, `fuzzel`), Hyprland Lua configuration (`~/.config/caelestia/hypr-user.lua`, `~/.config/hypr/variables.lua`).
- **Target Hardware**: Personal workstation `framboise` (Intel Core Ultra 5 Lunar Lake) with a physical French AZERTY keyboard (`fr`).

---

## 2. Context & Root Cause

### Default Behavior
In upstream Caelestia dotfiles, the emoji picker is bound in `~/.config/hypr/hyprland/keybinds.lua` using the variable `vars.kbEmoji`:
```lua
create_bind(vars.kbEmoji, hl.dsp.exec_cmd("pkill fuzzel || caelestia emoji -p"))
```
Where `kbEmoji` is defined in `~/.config/hypr/variables.lua` as:
```lua
kbEmoji = "SUPER + Period"
```

### AZERTY Keyboard Layout Mismatch
1. **Physical Key vs Keysym on AZERTY**:
   - On a standard French AZERTY keyboard, the key bearing the period character (`.`) is located on row 4 (position AB09, between `,` and `:`).
   - In its base (unshifted) state, this physical key sends the keysym `semicolon` (`;`).
   - The period keysym (`period` / `.`) is only emitted when holding `Shift` (`Shift + ;`).
2. **Modifier Mask Mismatch in Hyprland**:
   - When the user presses `SUPER` + the physical dot key without `Shift`, XKB outputs `semicolon` with modmask `64` (`SUPER`). Hyprland has no rule for `SUPER + semicolon`, so nothing happens.
   - If the user presses `SUPER + SHIFT` + the physical dot key, XKB outputs `period` (or `semicolon`), but with modmask `65` (`SUPER + SHIFT`). Hyprland's binding was strictly registered for modmask `64` (`SUPER` without `SHIFT`).
   - Consequently, the emoji picker could never be opened from an AZERTY laptop keyboard.

---

## 3. Solution Implemented

We configured dedicated keybindings in `~/.config/caelestia/hypr-user.lua` to accommodate how users naturally press the key on AZERTY keyboards:

1. **`SUPER + semicolon`**: Direct physical key press without needing to hold `Shift` (ergonomic thumb on `SUPER` + finger on the dot/semicolon key).
2. **`SUPER + SHIFT + semicolon`** and **`SUPER + SHIFT + period`**: Supports pressing the key while holding `Shift` (for users accustomed to typing a literal dot).
3. **`SUPER + period`**: Maintained for external keyboards, numpads (`KP_Decimal`), or alternative layouts.

### Configuration Snippet (`~/.config/caelestia/hypr-user.lua`)

```lua
-- ─── Emoji Picker (AZERTY) ───────────────────────────────────────────────────
-- On French AZERTY, the physical key for '.' is ';' (unshifted) / '.' (shifted).
-- Default Caelestia binds 'SUPER + Period', which requires Shift but lacks the Shift modifier.
-- Binding 'SUPER + semicolon' allows triggering the picker directly by pressing SUPER + the physical dot key.
-- Binding with SHIFT ensures it also opens if the user holds Shift to type the literal dot.
hl.bind("SUPER + semicolon", hl.dsp.exec_cmd("pkill fuzzel || caelestia emoji -p"))
hl.bind("SUPER + SHIFT + semicolon", hl.dsp.exec_cmd("pkill fuzzel || caelestia emoji -p"))
hl.bind("SUPER + SHIFT + period", hl.dsp.exec_cmd("pkill fuzzel || caelestia emoji -p"))
hl.bind("SUPER + period", hl.dsp.exec_cmd("pkill fuzzel || caelestia emoji -p"))
```

---

## 4. Verification & Status

1. **Reload Hyprland**:
   ```bash
   hyprctl reload
   ```
2. **Inspect Active Bindings**:
   Verify that `semicolon` and `period` are registered under both modmask `64` (`SUPER`) and `65` (`SUPER + SHIFT`):
   ```bash
   hyprctl binds | grep -B 2 -A 5 -i -E "semicolon|period"
   ```
   *Expected Output*:
   - `modmask: 64` / `key: semicolon`
   - `modmask: 65` / `key: semicolon`
   - `modmask: 64` / `key: Period`
   - `modmask: 65` / `key: period`

3. **Runtime Test**:
   - Press `SUPER + ;` (physical `.` key): The Fuzzel emoji picker opens immediately.
   - Press `SUPER + SHIFT + ;`: The Fuzzel emoji picker opens.
   - Select an emoji and press `Enter`: The emoji is copied to the Wayland clipboard (`wl-copy`) ready to paste (`Ctrl + V`).

---

## 5. Maintenance & Handy Commands

- **Test emoji picker directly from terminal**:
  ```fish
  caelestia emoji -p
  ```
- **Update or re-fetch emoji and Nerd Font glyph database**:
  ```fish
  caelestia emoji -f
  ```
- **Close any lingering Fuzzel instance**:
  ```fish
  pkill fuzzel
  ```
