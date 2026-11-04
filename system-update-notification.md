# System Update Notification (Click-to-Update, Persistent Terminal)

## Overview

A user systemd timer periodically checks for pending package updates (official repositories + AUR) and shows a desktop notification with an **Update** action. Clicking the action opens a `foot` terminal running `caelestia update`.

**Problem fixed**: the `foot` window closed immediately once `caelestia update` finished, making it impossible to read the output (errors, `.pacnew` warnings, AUR build logs, etc.).

**Affected components**:
- `~/.local/bin/check-updates.sh`
- `~/.config/systemd/user/check-updates.service`
- `~/.config/systemd/user/check-updates.timer`
- `notify-send` (libnotify) → Caelestia notification daemon
- `foot`, `caelestia update` (wraps `pacman` / `paru`)

## Context & Root Cause

The script launched the update with:

```bash
foot caelestia update
```

`foot` runs the given command as its child process and closes the window as soon as that child exits. Since `caelestia update` returns once the upgrade is done, the window disappeared with it.

Note: the service is `Type=oneshot`, so systemd's start timeout defaults to `infinity`. The script (and the open terminal) can therefore stay alive as long as needed without being killed.

## Solution Implemented

`foot` now runs a small `bash` wrapper that executes the update, prints a colored success/failure status with the exit code, and waits for **Enter** before closing.

File: `~/.local/bin/check-updates.sh`

```bash
#!/usr/bin/env bash

# 1. Check official repositories
OFFICIAL=$(checkupdates 2>/dev/null | wc -l)

# 2. Check AUR packages (if yay or paru is installed)
AUR=0
if command -v yay &> /dev/null; then
    AUR=$(yay -Qum 2>/dev/null | wc -l)
elif command -v paru &> /dev/null; then
    AUR=$(paru -Qum 2>/dev/null | wc -l)
fi

TOTAL=$((OFFICIAL + AUR))

# 3. Send desktop notification if updates are available
if [ "$TOTAL" -gt 0 ]; then
    ACTION=$(notify-send \
        --app-name="System Update" \
        --icon="system-software-update" \
        --urgency="normal" \
        --action="update=Update" \
        "Updates Available" \
        "$TOTAL package(s) waiting ($OFFICIAL official,$AUR AUR).")

    # If the user clicks the action button, trigger update command
    if [ "$ACTION" = "update" ]; then
        # Run the update in foot and keep the window open afterwards so the
        # output can be read. The window closes only when Enter is pressed.
        foot --title="System Update" bash -c '
            caelestia update
            status=$?
            echo
            if [ "$status" -eq 0 ]; then
                printf "\e[1;32m✔ Update finished successfully.\e[0m\n"
            else
                printf "\e[1;31m✘ Update exited with code %s.\e[0m\n" "$status"
            fi
            read -rp "Press Enter to close this window..." _
        '
    fi
fi
```

Units (unchanged):

```ini
# ~/.config/systemd/user/check-updates.service
[Unit]
Description=Check if package updates are available

[Service]
Type=oneshot
ExecStart=%h/.local/bin/check-updates.sh
```

```ini
# ~/.config/systemd/user/check-updates.timer
[Unit]
Description=Timer to regularly check for updates

[Timer]
OnStartupSec=5min
OnUnitActiveSec=2h
Persistent=true

[Install]
WantedBy=timers.target
```

Why a wrapper instead of `foot --hold`: `--hold` keeps the window but leaves it unresponsive with no status message and no keyboard way to close it. The wrapper shows whether the update succeeded and closes cleanly with Enter.

## Verification & Status

```fish
# Syntax check of the script
bash -n ~/.local/bin/check-updates.sh; and echo OK

# Timer is active and scheduled
systemctl --user list-timers check-updates.timer

# Trigger a check now (notification appears only if updates are pending)
systemctl --user start --no-block check-updates.service
```

Click **Update** on the notification: after `caelestia update` finishes, the window stays open with `✔ Update finished successfully.` (or the error code) and `Press Enter to close this window...`.

Test the wrapper alone without updating:

```fish
foot --title="System Update" bash -c 'echo test; read -rp "Press Enter to close this window..." _'
```

## Maintenance & Troubleshooting

```fish
# Logs of the check service
journalctl --user -u check-updates.service -n 50

# Service state (stays "activating" while the update terminal is open)
systemctl --user status check-updates.service

# Restart the timer after editing the units
systemctl --user daemon-reload
systemctl --user restart check-updates.timer

# Run the update manually
foot caelestia update
```

- While the terminal is open, `check-updates.service` remains `activating`; the next timer tick is not scheduled until the window is closed (`OnUnitActiveSec` counts from activation). This is expected.
- To skip prompts, change `caelestia update` to `caelestia update --noconfirm` in the wrapper.
