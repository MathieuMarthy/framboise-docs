# Volume Adjustment Step Configuration in Hyprland / Caelestia

## 1. Overview

This runbook documents how the volume step size for hardware volume keys (`XF86AudioRaiseVolume` and `XF86AudioLowerVolume`) is configured on `framboise`. By default in Caelestia's Hyprland configuration, the volume changes by **10%** per key press. This setup adjusts the step size to **5%** for finer audio level control.

- **Affected Components**: Hyprland Lua configuration, Caelestia Hyprland variable overrides (`~/.config/caelestia/hypr-vars.lua`), WirePlumber / PipeWire (`wpctl`).
- **Target Hardware**: Personal workstation `framboise` (Intel Core Ultra 5 Lunar Lake).

---

## 2. Context & Root Cause

### Upstream Default Configuration
Hyprland keybindings in Caelestia are defined in [`~/.config/hypr/hyprland/keybinds.lua`](file:///home/mathieu/.config/hypr/hyprland/keybinds.lua):

```lua
-- Volume
create_bind({ vars.kbVolumeMute, "XF86AudioMute" }, hl.dsp.exec_cmd("wpctl set-mute @DEFAULT_AUDIO_SINK@ toggle"), locked)
create_bind("XF86AudioMicMute", hl.dsp.exec_cmd("wpctl set-mute @DEFAULT_AUDIO_SOURCE@ toggle"), locked)
create_bind(
    "XF86AudioRaiseVolume",
    hl.dsp.exec_cmd(
        "wpctl set-mute @DEFAULT_AUDIO_SINK@ 0; wpctl set-volume -l " ..
        (vars.volumeMax / 100) .. " @DEFAULT_AUDIO_SINK@ " .. vars.volumeStep .. "%+"
    ),
    locked_repeating
)
create_bind(
    "XF86AudioLowerVolume",
    hl.dsp.exec_cmd(
        "wpctl set-mute @DEFAULT_AUDIO_SINK@ 0; wpctl set-volume @DEFAULT_AUDIO_SINK@ " .. vars.volumeStep .. "%-"
    ),
    locked_repeating
)
```

The volume step parameter is initialized in [`~/.config/hypr/variables.lua`](file:///home/mathieu/.config/hypr/variables.lua):
```lua
volumeStep = 10,
volumeMax  = 100,
```

### Clean Override Mechanism
In [`~/.config/hypr/hyprland.lua`](file:///home/mathieu/.config/hypr/hyprland.lua), Caelestia explicitly imports user variable overrides from [`~/.config/caelestia/hypr-vars.lua`](file:///home/mathieu/.config/caelestia/hypr-vars.lua) before keybindings are registered:

```lua
-- User variables
maybe_create(home .. "/.config/caelestia/hypr-vars.lua", "return {}\n")
local overrides = require("hypr-vars")
if type(overrides) == "table" then
    local vars = require("variables")
    for k, v in pairs(overrides) do
        vars[k] = v
    end
end
```

By defining `volumeStep = 5` inside `~/.config/caelestia/hypr-vars.lua`, we cleanly customize the volume increment without altering upstream Hyprland files (`variables.lua`), making the configuration robust against dotfile updates.

---

## 3. Solution Implemented

We updated [`~/.config/caelestia/hypr-vars.lua`](file:///home/mathieu/.config/caelestia/hypr-vars.lua) to override `volumeStep`:

```lua
return {
    volumeStep = 5,
}
```

### Applying the Changes
Hyprland reloads its Lua configuration dynamically using:
```bash
hyprctl reload
```

---

## 4. Verification & Status

1. **Verify `volumeStep` in the active Hyprland Lua state**:
   ```fish
   hyprctl repl "return require('variables').volumeStep"
   ```
   *Expected output*: `5`

2. **Inspect the registered volume keybindings**:
   ```fish
   hyprctl -j binds | jq '.[] | select(.key | contains("Volume"))'
   ```

3. **Check the actual command executed by the dispatcher**:
   ```fish
   hyprctl repl "local f = debug.getregistry()[204]; for i=1,10 do local k,v = debug.getupvalue(f, i); if not k then break end; print(k, v) end"
   ```
   *Output*:
   ```text
   wpctl set-mute @DEFAULT_AUDIO_SINK@ 0; wpctl set-volume -l 1.0 @DEFAULT_AUDIO_SINK@ 5%+
   ```

4. **Verify live volume adjustment with `wpctl`**:
   - Check current volume: `wpctl get-volume @DEFAULT_AUDIO_SINK@`
   - Press the volume up key on your keyboard.
   - Run `wpctl get-volume @DEFAULT_AUDIO_SINK@` again and verify it increased by `0.05` (+5%).

---

## 5. Maintenance & Useful Commands

| Task | Command (Fish compatible) |
| :--- | :--- |
| **Inspect Current Sink Volume** | `wpctl get-volume @DEFAULT_AUDIO_SINK@` |
| **Set Specific Volume (e.g. 35%)**| `wpctl set-volume @DEFAULT_AUDIO_SINK@ 0.35` |
| **Check Active Volume Step** | `hyprctl repl "return require('variables').volumeStep"` |
| **Reload Hyprland Config** | `hyprctl reload` |
