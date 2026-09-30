# battery-alert & battery-charge-limit

## Overview

**Problem**:
1. Keeping a lithium-ion battery charged at 100% continuously accelerates its degradation. The optimal longevity range is between **20% and 80%**.
2. When working in fullscreen applications (games, videos, fullscreen editors), visual battery alert toasts are easily missed or hidden by default.

On this Dell laptop (**Dell Pro 13 Plus PB13250 / Intel Lunar Lake**), three key challenges occurred:
1. **The charge limit was ignored by hardware**: Even with `charge_control_end_threshold` set to 80, the laptop continued charging to 100%, triggering repetitive "Battery at >80%" alert notifications.
2. **The limit disappeared after sleep / suspend**: Waking the PC from sleep or closing the lid caused the Dell Embedded Controller (EC) to reset sysfs attributes to 100%. Pressing `SUPER + B` would then announce "limit enabled at 80%" again instead of toggling to 100%, desynchronizing the toggle.
3. **Inaudible and hidden battery alerts in fullscreen**: Caelestia HUD toasts by default had no audio feedback and were suppressed in fullscreen mode (`utilities.toasts.fullscreen = "off"`).

**Solution implemented**:
1. **`battery-alert`** — checks battery state every 5 minutes: auto-heals hardware threshold drift, triggers native Caelestia toasts, and plays dedicated audio alerts (`dialog-warning.oga` / `suspend-error.oga`) when charging exceeds the threshold (>80%) or discharging falls to low (≤20%) / critical (≤10%) levels.
2. **`battery-charge-limit`** — manages hardware thresholds and Dell `charge_types` (`Custom`), maintains persistent state in `~/.local/state/battery-charge-limit`, and is bound to `SUPER + B`.
3. **Caelestia Fullscreen Toasts Configuration** — configured `"fullscreen": "important"` in `~/.config/caelestia/shell.json` so that warning and error toasts are permitted over fullscreen windows.

**Affected components**: `BAT0` (`/sys/class/power_supply/BAT0`), `dell-laptop` driver, systemd user session, sleep hook (`systemd-sleep`), PipeWire / WirePlumber audio (`pw-play` / `canberra-gtk-play`), `caelestia shell toaster`, `~/.config/caelestia/shell.json`, Hyprland (`hypr-user.lua`), udev.

---

## Context / Root Cause

### 1. Dell Firmware Charging Modes (`charge_types`)
The Dell kernel driver (`dell-laptop`) exposes `/sys/class/power_supply/BAT0/charge_types` with modes: `Trickle`, `Fast`, `Standard`, `[Adaptive]`, `Custom`.
By default, the Dell BIOS/EC operates in `Adaptive` mode. **In `Adaptive` mode, the firmware completely ignores `charge_control_end_threshold`.** Setting 80% in `charge_control_end_threshold` had zero effect on hardware charging until `charge_types` was explicitly switched to `Custom`.

### 2. Reset on Suspend / Resume (Firmware Volatility)
Linux `sysfs` files are volatile in-memory representations. On Dell laptops, the Embedded Controller (EC) re-evaluates power profiles on every suspend/resume cycle and AC replug, resetting `charge_control_end_threshold` to 100% and reverting `charge_types`.
Previously, the sleep hook only adjusted permissions (`chmod g+w`) without restoring the threshold value, causing the limit to "vanish" after sleep.

### 3. Fullscreen Alert Suppression & Lack of Audio
In Caelestia Shell, toasts are muted by default when any application is fullscreen (`utilities.toasts.fullscreen` defaults to `"off"`). Furthermore, Caelestia does not produce audio cues for toasts natively. Adding PipeWire audio playback to `battery-alert` and enabling `"fullscreen": "important"` ensures the user is warned audibly and visually even inside fullscreen applications.

---

## Solution Implemented

### Files created / modified

| File | Role |
| :--- | :--- |
| `~/.local/bin/battery-charge-limit` | Toggles hardware limit (80% / 100%), sets Dell `Custom` mode, persists state |
| `~/.local/bin/battery-alert` | Auto-heals threshold drift, sends Caelestia toasts, and plays audio alerts (>80% charge, ≤20% low, ≤10% critical) |
| `~/.config/caelestia/shell.json` | Configures `utilities.toasts.fullscreen: "important"` for fullscreen toast rendering |
| `~/.local/state/battery-charge-limit` | Persistent state file storing target threshold (`80` or `100`) |
| `~/.local/state/battery-alert-last-discharge-level` | Tracks last notified discharge percentage to avoid redundant sound triggers |
| `~/.config/systemd/user/battery-charge-limit.service` | Oneshot unit applying saved limit on session login |
| `~/.config/systemd/user/battery-alert.service` | Systemd oneshot unit for battery-alert with audio alerts |
| `~/.config/systemd/user/battery-alert.timer` | Timer: runs every 5 minutes |
| `~/.local/bin/setup-battery-charge-limit` | Setup script (requires sudo) — installs udev rule + sleep hook |
| `/etc/udev/rules.d/80-battery-charge-limit.rules` | udev rule: grants `users` group write access to thresholds & `charge_types` |
| `/etc/systemd/system-sleep/battery-charge-limit-perms` | Sleep hook: reapplies permissions AND restores `Custom` 80% on resume |
| `~/.config/caelestia/hypr-user.lua` | `SUPER + B` keybind for toggle |

---

## Detailed Script Behaviors

### 1. `battery-charge-limit`

```bash
battery-charge-limit          # toggle between 80% and 100% (used by SUPER + B)
battery-charge-limit on       # force enable limit at 80%
battery-charge-limit off      # force disable limit (reset to 100%)
battery-charge-limit apply    # reapply saved state to hardware sysfs
battery-charge-limit status   # {"hardware_limit": 80, "configured_limit": 80, "charge_type": "Custom", "limited": true}
```

#### Safe Threshold Ordering
To prevent kernel `-EINVAL` errors:
- **When lowering to 80%**: `charge_types` → `Custom`, `charge_control_start_threshold` → `75`, `charge_control_end_threshold` → `80`.
- **When raising to 100%**: `charge_control_end_threshold` → `100`, `charge_control_start_threshold` → `95`.

---

### 2. `battery-alert`

Runs every 5 minutes via `battery-alert.timer`:
1. **Self-Healing**: Checks if the persistent state is 80%. If the hardware sysfs has drifted (e.g. back to 100 or non-`Custom`), it immediately calls `battery-charge-limit apply --silent`.
2. **Charging Alert (>80%)**: If the battery is actively charging and exceeds the threshold:
   - Displays a Caelestia toast (`warn` or `error` if ≥95%).
   - Plays an audio alert in the background via `pw-play` / `canberra-gtk-play` (`dialog-warning.oga` or `suspend-error.oga`).
3. **Discharging Low Battery Alert (≤20% / ≤10%)**:
   - If capacity ≤ 10%: sends critical toast and plays `suspend-error.oga`.
   - If capacity ≤ 20%: sends warning toast and plays `dialog-warning.oga`.
   - Uses `~/.local/state/battery-alert-last-discharge-level` so the alert sounds once per threshold crossing and does not spam every 5 minutes.
   - Clears discharge state upon reconnection to AC power.

---

### 3. Caelestia Shell Config (`~/.config/caelestia/shell.json`)

```json
{
    "utilities": {
        "toasts": {
            "fullscreen": "important"
        }
    }
}
```
Setting `"fullscreen": "important"` ensures that `Toast.Warning` and `Toast.Error` HUD notifications remain active when a fullscreen game or media application is running.

---

### 4. Sleep Hook (`/etc/systemd/system-sleep/battery-charge-limit-perms`)

Executed as `root` on every resume from suspend:
1. Re-applies `chgrp users` and `chmod g+w` to `charge_control_end_threshold`, `charge_control_start_threshold`, and `charge_types`.
2. Reads `/home/mathieu/.local/state/battery-charge-limit` and immediately restores `Custom` mode and 80% threshold before the user session resumes.

---

## Verification & Status

```fish
# Check status JSON
battery-charge-limit status

# Verify Dell charge type is Custom
cat /sys/class/power_supply/BAT0/charge_types
# Expected output: Trickle Fast Standard Adaptive [Custom]

# Verify thresholds
cat /sys/class/power_supply/BAT0/charge_control_end_threshold    # 80
cat /sys/class/power_supply/BAT0/charge_control_start_threshold  # 75

# Check persistent state file
cat ~/.local/state/battery-charge-limit                          # 80

# Check user services
systemctl --user status battery-charge-limit.service
systemctl --user status battery-alert.timer
```

---

## Maintenance & Test Commands

Commands below are valid for **Fish** shell:

```fish
# Manually test toggle
battery-charge-limit toggle

# Test high-charge notification and audio alert
battery-alert test

# Test low-battery notification and audio alert
battery-alert test-low

# Check logs of 5-minute watchdog
journalctl --user -u battery-alert.service -n 20
```
