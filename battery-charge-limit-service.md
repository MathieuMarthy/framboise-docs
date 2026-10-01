# battery-charge-limit.service — Session Startup Battery Threshold Enforcement

## Overview

**Problem**:
On Linux, kernel `sysfs` power supply attributes (`/sys/class/power_supply/BAT0/`) are ephemeral and reset to hardware defaults upon every system reboot. On this Dell laptop (**Dell Pro 13 Plus PB13250 / Intel Lunar Lake**), the Dell Embedded Controller (EC) boots with:
1. `charge_types` reset to `Adaptive`.
2. `charge_control_end_threshold` reset to `100%`.

Without an automated boot/login mechanism, any user preference to protect battery health (e.g., maintaining an 80% threshold) is lost across reboots, causing the laptop to charge up to 100% and accelerate battery wear.

**Solution Implemented**:
A dedicated systemd user service: `~/.config/systemd/user/battery-charge-limit.service`.
Enabled as part of `default.target`, this oneshot unit runs immediately upon user session initialization, reading the saved state from `~/.local/state/battery-charge-limit` and executing `~/.local/bin/battery-charge-limit apply --silent` to program the Dell EC without displaying intrusive desktop toasts during login.

**Affected Components**:
- Systemd user session (`~/.config/systemd/user/battery-charge-limit.service`)
- State file (`~/.local/state/battery-charge-limit`)
- Threshold manager (`~/.local/bin/battery-charge-limit`)
- Hardware sysfs (`/sys/class/power_supply/BAT0/charge_control_end_threshold`, `charge_control_start_threshold`, `charge_types`)
- Accompanying runbooks: [`battery-alert.md`](file:///home/mathieu/docs/battery-alert.md)

---

## Context & Root Cause

### 1. Ephemeral Nature of Linux `sysfs`
The `/sys` virtual filesystem is generated dynamically by the Linux kernel in memory (`sysfs`). Values written to files under `/sys/class/power_supply/BAT0/` do not persist to disk. Upon shutdown or reboot:
- The Linux kernel unloads drivers and the system powers down.
- Upon booting, the BIOS/UEFI and Dell Embedded Controller initialize hardware power profiles with their default factory state (`Adaptive`, 100% stop threshold).
- Even though the user set 80% during their last session, the hardware resumes charging to 100% unless instructed otherwise.

### 2. Dell Firmware Specifics (`Custom` Mode Requirement)
The Dell platform driver (`dell-laptop`) exposes charging profiles via `/sys/class/power_supply/BAT0/charge_types`. The Dell EC completely ignores `charge_control_end_threshold` while in `Adaptive`, `Standard`, or `ExpressCharge` modes. 
The threshold is **only honored** when `charge_types` is explicitly set to `Custom`.

### 3. Startup Timing & Privilege Separation
- **Privilege handling**: The udev rule `/etc/udev/rules.d/80-battery-charge-limit.rules` assigns group ownership of `charge_control_*` and `charge_types` to the `users` group with write permissions (`chmod g+w`).
- **User session execution**: Running as a user-level service (`systemd --user`) allows the unit to run under the user's UID (`mathieu`), access `$HOME` and `$XDG_STATE_HOME`, read the user's preferred limit, and execute cleanly within the graphical session without requiring `root` or `sudo`.

---

## Solution Implemented

### 1. Systemd User Service File

File path: [`~/.config/systemd/user/battery-charge-limit.service`](file:///home/mathieu/.config/systemd/user/battery-charge-limit.service)

```ini
[Unit]
Description=Apply saved battery charge limit on session startup
Documentation=file:///home/mathieu/docs/battery-charge-limit-service.md file:///home/mathieu/docs/battery-alert.md
After=default.target

[Service]
Type=oneshot
ExecStart=/home/mathieu/.local/bin/battery-charge-limit apply --silent
RemainAfterExit=yes

[Install]
WantedBy=default.target
```

### 2. Breakdown of Service Directives

| Directive | Purpose & Rationale |
| :--- | :--- |
| `After=default.target` | Ensures the user session environment, filesystems, and basic target dependencies are reached before executing. |
| `Documentation=...` | Provides direct file URIs to this runbook and the broader battery management documentation, viewable via `systemctl --user status`. |
| `Type=oneshot` | The command executes once at startup to apply hardware registers and immediately exits. It does not spawn a long-running daemon. |
| `ExecStart=... apply --silent` | Invokes the script with `apply` to parse `~/.local/state/battery-charge-limit` and write sysfs values. The `--silent` flag suppresses Caelestia HUD notification toasts so session startup remains quiet. |
| `RemainAfterExit=yes` | Marks the unit as `active (exited)` after successful completion rather than reverting to `inactive`. This prevents systemd from re-triggering the unit unnecessarily. |
| `WantedBy=default.target` | Enables automatic activation when the user logs in and the systemd user instance reaches `default.target`. |

---

## Complete Persistence Architecture

`battery-charge-limit.service` is one pillar of a 4-part battery management architecture on `framboise`:

```
┌────────────────────────────────────────────────────────────────────────┐
│                        User Preference State                           │
│                ~/.local/state/battery-charge-limit                     │
│                             (80 or 100)                                │
└──────────────────┬─────────────────┬─────────────────┬─────────────────┘
                   │                 │                 │
     1. Login      │    2. Resume    │   3. Runtime    │   4. Watchdog
                   ▼                 ▼                 ▼                 ▼
 ┌───────────────────┐ ┌───────────────┐ ┌───────────┐ ┌─────────────────┐
 │battery-charge-    │ │systemd-sleep  │ │Hyprland   │ │battery-alert    │
 │limit.service      │ │hook (root)    │ │SUPER + B  │ │.timer (5 min)   │
 ├───────────────────┤ ├───────────────┤ ├───────────┤ ├─────────────────┤
 │Runs once on login │ │Runs on wake   │ │Toggles    │ │Heals drift and  │
 │Applies saved limit│ │Re-applies     │ │80% / 100% │ │sounds alerts if │
 │--silent (no toast)│ │perms & limit  │ │with toast │ │exceeding target │
 └─────────┬─────────┘ └───────┬───────┘ └─────┬─────┘ └────────┬────────┘
           │                   │               │                │
           └───────────────────┴───────┬───────┴────────────────┘
                                       ▼
                     ┌───────────────────────────────────┐
                     │ Hardware sysfs (/dev/BAT0)        │
                     │  • charge_types -> [Custom]       │
                     │  • start_threshold -> 75          │
                     │  • end_threshold   -> 80          │
                     └───────────────────────────────────┘
```

1. **Session Login (Reboot persistence)**:
   Handled by `battery-charge-limit.service`. Reads `~/.local/state/battery-charge-limit` and configures sysfs.
2. **Suspend / Resume (Sleep persistence)**:
   Handled by `/etc/systemd/system-sleep/battery-charge-limit-perms` (root hook). Re-applies group permissions and restores `Custom` 80% if stored in the state file.
3. **Manual User Toggle**:
   Handled by `SUPER + B` in Hyprland (`~/.config/caelestia/hypr-user.lua`), calling `battery-charge-limit toggle`. Updates state file and shows Caelestia toast.
4. **Drift Self-Healing**:
   Handled by `battery-alert.timer` (running `battery-alert.service` every 5 minutes). If hardware registers drift back to 100 or non-`Custom`, it re-executes `apply --silent`.

---

## Verification & Status

Commands below are formatted for the **Fish** shell:

### 1. Check Service Status
```fish
systemctl --user status battery-charge-limit.service
```
Expected output:
```text
● battery-charge-limit.service - Apply saved battery charge limit on session startup
     Loaded: loaded (~/.config/systemd/user/battery-charge-limit.service; enabled; preset: enabled)
     Active: active (exited) since ...
    Process: ... ExecStart=/home/mathieu/.local/bin/battery-charge-limit apply --silent (code=exited, status=0/SUCCESS)
```

### 2. Inspect Service Journal Logs
```fish
journalctl --user -u battery-charge-limit.service -b 0
```
Expected output:
```text
systemd[...]: Starting Apply saved battery charge limit on session startup...
battery-charge-limit[...]: Applied limit: 80%
systemd[...]: Finished Apply saved battery charge limit on session startup.
```

### 3. Verify Hardware sysfs Attributes
```fish
# Verify charge type is Custom
cat /sys/class/power_supply/BAT0/charge_types
# Output should contain: [Custom]

# Verify thresholds
cat /sys/class/power_supply/BAT0/charge_control_end_threshold
# Output: 80

cat /sys/class/power_supply/BAT0/charge_control_start_threshold
# Output: 75

# Verify state file consistency
cat ~/.local/state/battery-charge-limit
# Output: 80
```

---

## Maintenance & Troubleshooting

### Re-triggering the Service Manually
To test the service execution without logging out:
```fish
systemctl --user restart battery-charge-limit.service
```

### Checking Unit Symlink in systemd targets
Ensure the service is enabled to start with the user session:
```fish
# Check if enabled
systemctl --user is-enabled battery-charge-limit.service

# If disabled, re-enable it
systemctl --user enable battery-charge-limit.service
```
This manages the symlink:
`~/.config/systemd/user/default.target.wants/battery-charge-limit.service` → `~/.config/systemd/user/battery-charge-limit.service`.

### Reloading After Edits
If modifying `~/.config/systemd/user/battery-charge-limit.service`:
```fish
systemctl --user daemon-reload
systemctl --user restart battery-charge-limit.service
```

### Troubleshooting Permission Denied on `BAT0`
If the service logs `Error: The following file(s) are not writable`:
1. Check group membership (user must belong to `users`):
   ```fish
   id -Gn
   ```
2. Re-run setup script to refresh udev rules:
   ```fish
   bash ~/.local/bin/setup-battery-charge-limit
   ```
