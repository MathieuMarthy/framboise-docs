# Caelestia Transparency Toggle → All Windows (Hyprland + foot)

## Overview

- **Problem**: The Caelestia settings transparency toggle (`appearance.transparency.enabled` in `~/.config/caelestia/shell.json`) only affects the shell UI. Application windows stayed transparent.
- **Goal**: One toggle in the Caelestia settings turns transparency on/off for the shell, every Hyprland window, and `foot`.
- **Affected components**: `~/.config/caelestia/hypr-vars.lua`, `~/.local/bin/caelestia-transparency-sync`, `caelestia-transparency-sync.path` / `.service` (systemd user), `~/.config/foot/foot.ini`.

## Context / Root Cause

The Caelestia toggle is read by Quickshell only. Window transparency comes from the Caelestia Hyprland dotfiles:

- `~/.config/hypr/variables.lua`: `windowOpacity = 0.95`
- `~/.config/hypr/hyprland/rules.lua:43`:

```lua
hl.window_rule({ match = { fullscreen = false }, opacity = vars.windowOpacity .. " override" })
```

`override` makes the rule absolute, so it also beats `decoration.active_opacity = 1.0` from HyprMod (`~/.config/caelestia/hyprland-gui.lua`).

`foot` has its own background transparency (`alpha=` in `~/.config/foot/foot.ini`), independent of Hyprland.

## Solution Implemented

### 1. Hyprland reads the toggle at config load

`~/.config/caelestia/hypr-vars.lua` (user overrides, merged into `variables` by `~/.config/hypr/hyprland.lua` before the rules load; survives Caelestia updates):

```lua
local function shell_transparency()
    local f = io.open(os.getenv("HOME") .. "/.config/caelestia/shell.json")
    if not f then return true end
    local enabled = f:read("a"):match('"transparency"%s*:%s*{[^}]-"enabled"%s*:%s*(%a+)')
    f:close()
    return enabled ~= "false"
end

local vars = {
    volumeStep = 5,
}
if not shell_transparency() then
    vars.windowOpacity = 1.0 -- neutralizes the 0.95 "override" window rule (rules.lua)
end
return vars
```

Transparency on → upstream default `0.95`; off → `1.0`.

### 2. Live sync when the toggle changes

`~/.local/bin/caelestia-transparency-sync`: reads the toggle with `jq`, exits early if unchanged (shell.json also changes for every other setting), otherwise sets foot `alpha` (`0.78` on / `1.0` off) and runs `hyprctl reload`. Last applied state is stored in `~/.local/state/caelestia-transparency`.

`~/.config/systemd/user/caelestia-transparency-sync.path`:

```ini
[Path]
PathChanged=%h/.config/caelestia/shell.json

[Install]
WantedBy=default.target
```

`~/.config/systemd/user/caelestia-transparency-sync.service`:

```ini
[Service]
Type=oneshot
ExecStart=%h/.local/bin/caelestia-transparency-sync
```

Note: `jq '.x // true'` returns `true` when `.x` is `false` (`//` treats `false` as missing), hence `.appearance.transparency.enabled != false` in the script.

### Limits

- `foot` only reads its config at startup: already open terminals keep their old transparency, new ones follow the toggle.
- The foot "on" value `0.78` is hardcoded in the script (`FOOT_ALPHA_ON`). Edit it there if you change your preferred alpha.

## Verification & Status

Tested by toggling `shell.json` both ways: the state file and foot `alpha` followed (`true / 0.78`, `false / 1.0`), the service ran on each change, and `hyprctl configerrors` stayed empty.

```fish
systemctl --user status caelestia-transparency-sync.path
cat ~/.local/state/caelestia-transparency
grep '^alpha=' ~/.config/foot/foot.ini
journalctl --user -u caelestia-transparency-sync.service -n 10
hyprctl configerrors   # must print nothing
```

## Maintenance / Handy Commands

```fish
# Force a resync (e.g. after editing foot.ini by hand)
rm ~/.local/state/caelestia-transparency; systemctl --user start caelestia-transparency-sync.service

# Disable the live sync (Hyprland still follows the toggle at next login/reload)
systemctl --user disable --now caelestia-transparency-sync.path

# Check the upstream default value / rule
grep -n windowOpacity ~/.config/hypr/variables.lua ~/.config/hypr/hyprland/rules.lua
```
