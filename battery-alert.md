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
1. **`battery-alert`** — dynamically linked with `battery-charge-limit`: auto-heals Dell hardware drift, checks battery state every 10 minutes, and sends a visual Caelestia toast notifying that the battery is full when reaching the active limit (80% when limit is on, or 100% when off).
2. **Caelestia Native Battery Monitoring** — low-battery warnings (≤20%, ≤10%, ≤5%) and automatic critical hibernation (≤3%) are natively handled in real-time by Caelestia Shell (`modules/BatteryMonitor.qml` via UPower), avoiding duplicate toast notifications.
3. **`battery-charge-limit`** — manages hardware thresholds and Dell `charge_types` (`Custom`), maintains persistent state in `~/.local/state/battery-charge-limit`, and is bound to `SUPER + B`.
4. **Caelestia Fullscreen Toasts Configuration** — configured `"fullscreen": "important"` in `~/.config/caelestia/shell.json` so that warning and error toasts are permitted over fullscreen windows.

> [!NOTE]
> **Optimization Pass**:
> - **Timer polling relaxed to 10 minutes**: Polling interval was increased from 5 to 10 minutes because the systemd-sleep hook handles immediate drift recovery on wake, rendering 5-minute polling unnecessarily frequent (especially while on battery where the script exits immediately).
> - **Centralized sysfs handling**: Both the sleep hook (`/etc/systemd/system-sleep/battery-charge-limit-perms`) and setup script (`~/.local/bin/setup-battery-charge-limit`) delegate threshold writes directly to `battery-charge-limit apply --silent`, eliminating duplicate sysfs ordering logic.
> - **Post-write sysfs verification**: `battery-charge-limit` includes a `verify_sysfs()` check to catch silent Dell EC rejections and log warnings.
> - **Pre-suspend diagnostic logging**: The sleep hook logs sysfs state prior to suspend via `logger` for simpler troubleshooting.

**Affected components**: `BAT0` (`/sys/class/power_supply/BAT0`), `dell-laptop` driver, systemd user session, sleep hook (`systemd-sleep`), `caelestia shell toaster`, `modules/BatteryMonitor.qml`, `~/.config/caelestia/shell.json`, Hyprland (`hypr-user.lua`), udev.

---

## Context / Root Cause

### 1. Dell Firmware Charging Modes (`charge_types`)
The Dell kernel driver (`dell-laptop`) exposes `/sys/class/power_supply/BAT0/charge_types` with modes: `Trickle`, `Fast`, `Standard`, `[Adaptive]`, `Custom`.
By default, the Dell BIOS/EC operates in `Adaptive` mode. **In `Adaptive` mode, the firmware completely ignores `charge_control_end_threshold`.** Setting 80% in `charge_control_end_threshold` had zero effect on hardware charging until `charge_types` was explicitly switched to `Custom`.

### 2. Reset on Suspend / Resume (Firmware Volatility)
Linux `sysfs` files are volatile in-memory representations. On Dell laptops, the Embedded Controller (EC) re-evaluates power profiles on every suspend/resume cycle and AC replug, resetting `charge_control_end_threshold` to 100% and reverting `charge_types`.
Previously, the sleep hook only adjusted permissions (`chmod g+w`) without restoring the threshold value, causing the limit to "vanish" after sleep.

### 3. Fullscreen Alert Suppression & Silent Notifications
In Caelestia Shell, toasts are muted by default when any application is fullscreen (`utilities.toasts.fullscreen` defaults to `"off"`). Configuring `"fullscreen": "important"` ensures that warning and error HUD popups remain visible over games and fullscreen media. All audio playback was intentionally removed to keep toasts completely silent and unobtrusive.

### 4. Deduplication with Caelestia Native BatteryMonitor
Caelestia Shell natively includes `/etc/xdg/quickshell/caelestia/modules/BatteryMonitor.qml`, which listens to UPower device percentage signals and displays native toasts at 20% (Low battery), 10% (Battery Warning), 5% (Battery Critical), and initiates system hibernation at 3%.
Previously, `battery-alert` also dispatched its own toasts at ≤20% and ≤10%, resulting in duplicate overlapping toasts during battery discharge. All discharge handling was removed from `battery-alert`, delegating low-battery visual notifications entirely to Caelestia.

### 5. Synchronization Between `battery-alert` and `battery-charge-limit`
Previously, `battery-alert` had a hardcoded 80% threshold in its service file and warned to "unplug charger to preserve battery longevity". When the hardware limit is active, reaching 80% is the intended end state (the battery stops charging on its own). `battery-alert` now reads the configured state from `~/.local/state/battery-charge-limit`. When 80% is reached with the limit active, the notification announces `🔋 Battery full (80%)` with message `Battery limit is active (80%). Battery is fully charged.`. Redundant alerts every 10 minutes are suppressed via `~/.local/state/battery-alert-full-notified`, which resets on unplug or limit toggle.

---

## Solution Implemented

### Files created / modified

| File | Role |
| :--- | :--- |
| `~/.local/bin/battery-charge-limit` | Toggles hardware limit (80% / 100%), sets Dell `Custom` mode, persists state, and validates writes via `verify_sysfs()` |
| `~/.local/bin/battery-alert` | Auto-heals threshold drift and sends visual Caelestia toasts when target charge limit (80% or 100%) is reached |
| `~/.config/caelestia/shell.json` | Configures `utilities.toasts.fullscreen: "important"` for fullscreen toast rendering |
| `~/.local/state/battery-charge-limit` | Persistent state file storing target threshold (`80` or `100`) |
| `~/.local/state/battery-alert-full-notified` | Tracks whether the full-battery toast was sent for the current charge session |
| [`~/.config/systemd/user/battery-charge-limit.service`](file:///home/mathieu/docs/battery-charge-limit-service.md) | Oneshot unit applying saved limit on session login (see [dedicated runbook](file:///home/mathieu/docs/battery-charge-limit-service.md)) |
| `~/.config/systemd/user/battery-alert.service` | Systemd oneshot unit for battery-alert charge limit watchdog |
| `~/.config/systemd/user/battery-alert.timer` | Timer: runs every 10 minutes |
| `~/.local/bin/setup-battery-charge-limit` | Setup script (requires sudo) — installs udev rule + sleep hook, delegates threshold application to `battery-charge-limit` |
| `/etc/udev/rules.d/80-battery-charge-limit.rules` | udev rule: grants `users` group write access to thresholds & `charge_types` |
| `/etc/systemd/system-sleep/battery-charge-limit-perms` | Sleep hook: logs pre-suspend state, reapplies permissions, and delegates limit restore to `battery-charge-limit apply --silent` on resume |
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

#### Post-Write sysfs Verification (`verify_sysfs`)
The Dell Embedded Controller (EC) can occasionally fail or drop sysfs updates silently. After writing values, `battery-charge-limit` invokes `verify_sysfs("$limit")` to re-read the hardware sysfs attributes:
- Confirms that `charge_control_end_threshold` matches the requested value.
- When limiting to 80%, checks that `charge_types` reports `[Custom]`.
- If a discrepancy is detected, diagnostic warnings are printed to `stderr` (e.g., `WARNING: sysfs end_threshold mismatch (expected=..., actual=...)`), preventing silent failures from going unnoticed in systemd journals.

---

### 2. `battery-alert`

Runs every 10 minutes via `battery-alert.timer`:
1. **Self-Healing**: Checks if the persistent state is 80%. If the hardware sysfs has drifted (e.g. back to 100 or non-`Custom`), it immediately calls `battery-charge-limit apply --silent`.
2. **Linked Target Notification (80% or 100%)**:
   - Reads the active limit from `~/.local/state/battery-charge-limit`.
   - **Limit Active (80%)**: When reaching 80% (or `Not charging` / `Full`), displays a toast: `🔋 Battery full (80%)` with message `Battery limit is active (80%). Battery is fully charged.`.
   - **Limit Disabled (100%)**: When reaching 100%, displays: `🔋 Battery full (100%)` with message `Battery is fully charged.`.
   - **Spam Prevention**: Writes to `~/.local/state/battery-alert-full-notified` so the toast fires once per charge cycle and does not repeat every 10 minutes. The marker is automatically wiped when unplugging or toggling the limit with `SUPER + B`.
3. **Discharging State**: If discharging, the script exits immediately (`exit 0`). All low-battery notifications (20%, 10%, 5%) and hibernation (3%) are left to Caelestia's native `BatteryMonitor.qml`.

---

### 3. Caelestia Shell Config (`~/.config/caelestia/shell.json`)

```json
{
    "general": {
        "showOverFullscreen": true
    },
    "utilities": {
        "toasts": {
            "fullscreen": "important"
        }
    }
}
```
1. Setting `"utilities.toasts.fullscreen": "important"` ensures that `Toast.Warning` and `Toast.Error` HUD notifications remain active when a fullscreen app is running.
2. Setting `"general.showOverFullscreen": true` promotes Caelestia's Wayland surface layer from `Top` to `Overlay`, allowing UI overlays to render above Hyprland fullscreen windows.
3. In `battery-alert`, notifications are sent via both `caelestia shell toaster` and `notify-send` to ensure top-banner popups appear over games and media players even if bottom drawers are masked.

---

### 4. Sleep Hook (`/etc/systemd/system-sleep/battery-charge-limit-perms`)

Executed as `root` on suspend and resume:
1. **Pre-suspend (`pre`)**: Logs current sysfs state (`charge_control_end_threshold` and `charge_types`) via `logger -t battery-charge-limit` for suspend drift diagnostics.
2. **Post-resume (`post`)**:
   - Re-applies `chgrp users` and `chmod g+w` to `charge_control_end_threshold`, `charge_control_start_threshold`, and `charge_types`.
   - Delegates threshold restoration directly to `HOME=/home/mathieu /home/mathieu/.local/bin/battery-charge-limit apply --silent`, reusing centralized threshold ordering and verification logic instead of duplicating sysfs writes.

---

### 5. Setup Script (`~/.local/bin/setup-battery-charge-limit`)

Centralized setup utility (requires `sudo`):
- Installs the udev rule (`/etc/udev/rules.d/80-battery-charge-limit.rules`).
- Installs the sleep hook (`/etc/systemd/system-sleep/battery-charge-limit-perms`).
- Reloads udev rules and applies immediate group write permissions to sysfs attributes.
- Calls `battery-charge-limit apply --silent` and `battery-charge-limit status` to apply and inspect thresholds directly rather than duplicating sysfs ordering logic.

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

# Test high-charge visual notification toast
battery-alert test

# Check logs of 10-minute watchdog
journalctl --user -u battery-alert.service -n 20
```
