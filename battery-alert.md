# battery-alert & battery-charge-limit

## Overview

**Problem**: Keeping a lithium-ion battery charged at 100% continuously accelerates its degradation. The optimal longevity range is between **20% and 80%**.

**Two-part solution**:
1. **`battery-alert`** — sends a native Caelestia toast when the battery exceeds 80% while the charger is plugged in.
2. **`battery-charge-limit`** — toggles the *hardware* charge threshold (`charge_control_end_threshold`) between 80% and 100%, accessible via `SUPER + B` or the command line.

**Affected components**: `BAT0` (`/sys/class/power_supply/BAT0`), systemd user session, `caelestia shell toaster`, Hyprland (`hypr-user.lua`), udev.

---

## Context / Root Cause

The kernel exposes the current charge level at `/sys/class/power_supply/BAT0/capacity` and the charging state at `status` (`Charging`, `Discharging`, `Full`, `Not charging`). The `charge_control_end_threshold` file allows hardware-level charge limiting — the battery controller stops charging at the configured threshold. By default this file is read-only for regular users; a udev rule makes it writable by the `users` group at every boot.

---

## Solution Implemented

### Files created / modified

| File | Role |
| :--- | :--- |
| `~/.local/bin/battery-alert` | Sends a Caelestia toast when battery > 80% and charger is connected |
| `~/.local/bin/battery-charge-limit` | Toggles hardware charge limit between 80% and 100% |
| `~/.local/bin/setup-battery-charge-limit` | One-time setup script (requires sudo) — installs udev rule + sleep hook |
| `~/.config/systemd/user/battery-alert.service` | Systemd oneshot unit for battery-alert |
| `~/.config/systemd/user/battery-alert.timer` | Timer: runs every 5 minutes |
| `~/.config/caelestia/hypr-user.lua` | `SUPER + B` keybind for the toggle |
| `/etc/udev/rules.d/80-battery-charge-limit.rules` | udev rule: applies permissions at boot and on `change` events |
| `/etc/systemd/system-sleep/battery-charge-limit-perms` | Sleep hook: reapplies permissions after every resume from suspend/hibernate |

### Script: `battery-alert`

Checks the charge level every 5 minutes via systemd timer. Only sends a toast if the status is `Charging` or `Full` (never while discharging).

```bash
# Toast via Caelestia's native toaster
caelestia shell toaster warn "🔋 Battery at ${CAPACITY}%" "..." "battery_saver"
```

### Script: `battery-charge-limit`

```bash
battery-charge-limit on      # enable limit at 80% + toast
battery-charge-limit off     # reset to 100% + toast
battery-charge-limit toggle  # toggle (used by SUPER + B)
battery-charge-limit status  # {"limit": 80, "limited": true}
```

Also adjusts `charge_control_start_threshold` (75% in limited mode, 95% in normal mode) to avoid micro-cycles.

### udev rule

Fires on `ACTION=="add"` (boot) **and** `ACTION=="change"` (emitted by the battery driver when the charger is plugged/unplugged or on some resume paths):

```
# /etc/udev/rules.d/80-battery-charge-limit.rules
ACTION=="add|change", SUBSYSTEM=="power_supply", KERNEL=="BAT0", \
    RUN+="/bin/chgrp users .../charge_control_end_threshold", \
    RUN+="/bin/chmod g+w .../charge_control_end_threshold", ...
```

### systemd-sleep hook

Belt-and-suspenders guarantee: the kernel resets sysfs permissions on every suspend/resume cycle. This hook explicitly reapplies them in the `post` phase regardless of what udev does:

```bash
# /etc/systemd/system-sleep/battery-charge-limit-perms
case "$1" in
    post)
        chgrp users /sys/class/power_supply/BAT0/charge_control_end_threshold
        chmod g+w  /sys/class/power_supply/BAT0/charge_control_end_threshold
        # ... same for start_threshold
        ;;
esac
```

### Hyprland keybind (`~/.config/caelestia/hypr-user.lua`)

```lua
-- SUPER + B: toggle charge limit between 80% and 100%
hl.bind("SUPER + b", hl.dsp.exec_cmd("/home/mathieu/.local/bin/battery-charge-limit toggle"))
```

---

## Verification & Status

```bash
# Check timer status
systemctl --user status battery-alert.timer

# Show next trigger times
systemctl --user list-timers battery-alert.timer

# Test the alert toast manually (force threshold to 0)
/home/mathieu/.local/bin/battery-alert 0

# Check current charge limit state
battery-charge-limit status

# Verify udev rule and file permissions
ls -la /sys/class/power_supply/BAT0/charge_control_end_threshold
cat /sys/class/power_supply/BAT0/charge_control_end_threshold

# View service logs
journalctl --user -u battery-alert.service -n 20
```

---

## First-Time Setup (run once)

Run this command in your Fish terminal to install the udev rule:

```fish
bash ~/.local/bin/setup-battery-charge-limit
```

---

## Maintenance / Handy Commands

```bash
# Change alert threshold to 85%
micro ~/.config/systemd/user/battery-alert.service
# Edit: ExecStart=...battery-alert 85
systemctl --user daemon-reload

# Change check frequency (e.g. every 10 minutes)
micro ~/.config/systemd/user/battery-alert.timer
# Edit: OnUnitActiveSec=10min
systemctl --user daemon-reload && systemctl --user restart battery-alert.timer

# Temporarily stop the timer
systemctl --user stop battery-alert.timer

# Permanently disable
systemctl --user disable --now battery-alert.timer

# Re-enable
systemctl --user enable --now battery-alert.timer
```

> [!WARNING]
> The udev rule lives in `/etc/udev/rules.d/` (system file). The scripts and the keybind in `~/.config/caelestia/hypr-user.lua` are all user files. **None of these are touched by `paru -Syu caelestia-shell`** — they survive Caelestia updates.

> [!TIP]
> The Intel Core Ultra 5 236V has native support for `charge_control_end_threshold` via the `intel_pmc_core` driver. No need for `tlp` or `auto-cpufreq` — we write directly to sysfs.
