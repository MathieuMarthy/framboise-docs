# Hyprland Workspace Keybindings for AZERTY (Caelestia)

## Overview

This documentation explains how workspace navigation and window movement keybindings were configured for a French AZERTY keyboard layout in Caelestia's Hyprland environment.

Configuration file: `~/.config/caelestia/hypr-user.lua`

---

## The Problem

1. **AZERTY Number Row Behavior**:
   - On a standard French AZERTY keyboard, the top row produces symbols and accented characters in its base state:
     - `&`, `é`, `"`, `'`, `(`, `-`, `è`, `_`, `ç`, `à`
   - Numbers `1` through `0` require holding the `Shift` key.
   - Default Hyprland / Caelestia binds map `SUPER + [0-9]`, forcing AZERTY users to press `SUPER + Shift + &` just to switch to workspace 1.

2. **Keysym Requirements in Hyprland Lua**:
   - Hyprland's native Lua API (`hl.bind`) parses XKB keysym identifiers (e.g., `ampersand`, `quotedbl`) rather than literal punctuation characters (`&`, `"`).

3. **Key Collision on `-` (Workspace 6)**:
   - Caelestia binds `SUPER + Minus` by default to decrease window width (based on QWERTY layouts where `-` sits next to `0`).
   - On AZERTY, the physical key for workspace 6 is `-` (`minus`), leading to a conflict where pressing `SUPER + -` resized the active window instead of switching workspaces.

---

## Solution Implemented

In `~/.config/caelestia/hypr-user.lua`:

1. **Unbind Conflicting Shortcuts**:
   - `SUPER + Minus` and `SUPER + SHIFT + Minus` are unbound via `pcall(hl.unbind, ...)`.
   - Window resizing is still accessible via `SUPER + ALT + Left` / `SUPER + ALT + Up`.

2. **Map AZERTY Keysyms to Workspaces**:
   - Uses Caelestia's helper `utils.functions.wsaction("focus", "", i)` to ensure full compatibility with workspace groups.
   - Uses `utils.functions.wsaction("move", "", i)` to move windows between workspaces.

3. **Support Both Move Conventions**:
   - `SUPER + ALT + <key>`: Default Caelestia shortcut to move active window.
   - `SUPER + SHIFT + <key>` / `SUPER + SHIFT + [0-9]`: Standard Hyprland shortcut convention (since pressing `Shift` on AZERTY produces numeric keysyms `1` to `0`).

---

## Keybindings Reference Table

| Workspace | AZERTY Key | XKB Keysym | Switch Workspace | Move Active Window |
|:---------:|:----------:|:----------:|:----------------|:-------------------|
| **1** | `&` | `ampersand` | `SUPER + &` | `SUPER + ALT + &` / `SUPER + SHIFT + 1` |
| **2** | `é` | `eacute` | `SUPER + é` | `SUPER + ALT + é` / `SUPER + SHIFT + 2` |
| **3** | `"` | `quotedbl` | `SUPER + "` | `SUPER + ALT + "` / `SUPER + SHIFT + 3` |
| **4** | `'` | `apostrophe` | `SUPER + '` | `SUPER + ALT + '` / `SUPER + SHIFT + 4` |
| **5** | `(` | `parenleft` | `SUPER + (` | `SUPER + ALT + (` / `SUPER + SHIFT + 5` |
| **6** | `-` | `minus` | `SUPER + -` | `SUPER + ALT + -` / `SUPER + SHIFT + 6` |
| **7** | `è` | `egrave` | `SUPER + è` | `SUPER + ALT + è` / `SUPER + SHIFT + 7` |
| **8** | `_` | `underscore` | `SUPER + _` | `SUPER + ALT + _` / `SUPER + SHIFT + 8` |
| **9** | `ç` | `ccedilla` | `SUPER + ç` | `SUPER + ALT + ç` / `SUPER + SHIFT + 9` |
| **10** | `à` | `agrave` | `SUPER + à` | `SUPER + ALT + à` / `SUPER + SHIFT + 0` |

---

## Configuration Code

The code added to `~/.config/caelestia/hypr-user.lua`:

```lua
hl.config({ input = { kb_layout = "fr" }})

local fn = require("utils.functions")

-- AZERTY top-row keysyms (workspaces 1 through 10)
local azerty_keys = {
    "ampersand",   -- 1 (&)
    "eacute",      -- 2 (é)
    "quotedbl",    -- 3 (")
    "apostrophe",  -- 4 (')
    "parenleft",   -- 5 (()
    "minus",       -- 6 (-)
    "egrave",      -- 7 (è)
    "underscore",  -- 8 (_)
    "ccedilla",    -- 9 (ç)
    "agrave",      -- 10 (à)
}

-- Unbind default window resizing shortcuts on "-" to prevent conflicts with workspace 6
pcall(hl.unbind, "SUPER + Minus")
pcall(hl.unbind, "SUPER + SHIFT + Minus")

for i, key in ipairs(azerty_keys) do
    -- Switch workspace (1..10): SUPER + &..à
    hl.bind("SUPER + " .. key, fn.wsaction("focus", "", i))

    -- Move active window (1..10): SUPER + ALT + &..à (Caelestia convention)
    hl.bind("SUPER + ALT + " .. key, fn.wsaction("move", "", i))

    -- Move active window with SHIFT (both keysym and generated number 1..0)
    hl.bind("SUPER + SHIFT + " .. key, fn.wsaction("move", "", i))
    local num = i % 10
    hl.bind("SUPER + SHIFT + " .. num, fn.wsaction("move", "", i))
end
```

---

## Applying Changes

To reload the configuration without restarting Hyprland:

```bash
hyprctl reload
```

To inspect currently active bindings:

```bash
hyprctl binds
```
