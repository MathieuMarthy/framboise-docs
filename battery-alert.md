# battery-alert & battery-charge-limit

## Overview

**Problem**: Keeping a lithium-ion battery charged at 100% continuously accelerates its degradation. The optimal longevity range is between **20% and 80%**.

On this Dell laptop (**Dell Pro 13 Plus PB13250 / Intel Lunar Lake**), two issues occurred:
1. **The charge limit was ignored by hardware**: Even with `charge_control_end_threshold` set to 80, the laptop continued charging to 100%, triggering repetitive "Battery at >80%" alert notifications.
2. **The limit disappeared after sleep / suspend**: Waking the PC from sleep or closing the lid caused the Dell Embedded Controller (EC) to reset sysfs attributes to 100%. Pressing `SUPER + B` would then announce "limit enabled at 80%" again instead of toggling to 100%, desynchronizing the toggle.

**Two-part solution**:
1. **`battery-alert`** — checks battery state every 5 minutes: auto-heals hardware threshold drift and alerts the user via Caelestia toast if charging exceeds the threshold.
2. **`battery-charge-limit`** — manages hardware thresholds and Dell `charge_types` (`Custom`), maintains persistent state in `~/.local/state/battery-charge-limit`, and is bound to `SUPER + B`.

**Affected components**: `BAT0` (`/sys/class/power_supply/BAT0`), `dell-laptop` driver, systemd user session, sleep hook (`systemd-sleep`), `caelestia shell toaster`, Hyprland (`hypr-user.lua`), udev.

---

## Context / Root Cause

### 1. Dell Firmware Charging Modes (`charge_types`)
The Dell kernel driver (`dell-laptop`) exposes `/sys/class/power_supply/BAT0/charge_types` with modes: `Trickle`, `Fast`, `Standard`, `[Adaptive]`, `Custom`.
By default, the Dell BIOS/EC operates in `Adaptive` mode. **In `Adaptive` mode, the firmware completely ignores `charge_control_end_threshold`.** Setting 80% in `charge_control_end_threshold` had zero effect on hardware charging until `charge_types` was explicitly switched to `Custom`.

### 2. Reset on Suspend / Resume (Firmware Volatility)
Linux `sysfs` files are volatile in-memory representations. On Dell laptops, the Embedded Controller (EC) re-evaluates power profiles on every suspend/resume cycle and AC replug, resetting `charge_control_end_threshold` to 100% and reverting `charge_types`.
Previously, the sleep hook only adjusted permissions (`chmod g+w`) without restoring the threshold value, causing the limit to "vanish" after sleep.

---

## Solution Implemented

### Files created / modified

| File | Role |
| :--- | :--- |
| `~/.local/bin/battery-charge-limit` | Toggles hardware limit (80% / 100%), sets Dell `Custom` mode, persists state |
| `~/.local/bin/battery-alert` | Detects threshold drift (auto-heals) & toasts when charging > 80% |
| `~/.local/state/battery-charge-limit` | Persistent state file storing target threshold (`80` or `100`) |
| `~/.config/systemd/user/battery-charge-limit.service` | Oneshot unit applying saved limit on session login |
| `~/.config/systemd/user/battery-alert.service` | Systemd oneshot unit for battery-alert |
| `~/.config/systemd/user/battery-alert.timer` | Timer: runs every 5 minutes |
| `~/.local/bin/setup-battery-charge-limit` | Setup script (requires sudo) — installs udev rule + sleep hook |
| `/etc/udev/rules.d/80-battery-charge-limit.rules` | udev rule: grants `users` group write access to thresholds & `charge_types` |
| `/etc/systemd/system-sleep/battery-charge-limit-perms` | Sleep hook: reapplies permissions AND restores `Custom` 80% on resume |
| `~/.config/caelestia/hypr-user.lua` | `SUPER + B` keybind for toggle |

---

### Script: `battery-charge-limit`

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

### Script: `battery-alert`

Runs every 5 minutes via `battery-alert.timer`.
1. **Self-Healing**: Checks if the persistent state is 80%. If the hardware sysfs has drifted (e.g. back to 100 or non-`Custom`), it immediately calls `battery-charge-limit apply --silent`.
2. **Alert**: If the battery is actively charging and exceeds the threshold, displays a Caelestia toast.

---

### Sleep Hook (`/etc/systemd/system-sleep/battery-charge-limit-perms`)

Executed as `root` on every resume from suspend:
1. Re-applies `chgrp users` and `chmod g+w` to `charge_control_end_threshold`, `charge_control_start_threshold`, and `charge_types`.
2. Reads `/home/mathieu/.local/state/battery-charge-limit` and immediately restores `Custom` mode and 80% threshold before the user session resumes.

---

## Verification & Status

```bash
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

## Applying System Permissions (Run Once with Sudo)

To install the updated udev rule and resume sleep hook:

```fish
bash ~/.local/bin/setup-battery-charge-limit
```

---

## Maintenance / Handy Commands

```bash
# Manually test toggle
battery-charge-limit toggle

# Test alert notification toast manually
/home/mathieu/.local/bin/battery-alert 0

# Check logs of 5-minute watchdog
journalctl --user -u battery-alert.service -n 20
```
