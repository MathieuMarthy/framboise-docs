# Packet (Quick Share) Autostart & Background Tray Persistence

## Overview

Documents the setup for automatic startup and background system tray persistence for **Packet** (`io.github.nozwock.Packet`, a Linux client compatible with Google Quick Share / Nearby Share) on **`framboise`** running CachyOS with Hyprland and the Caelestia desktop shell.

**Objectives:**
1. **Autostart at Login:** Launch Packet automatically in the background at session startup, displaying an icon in the system tray ready to send and receive transfers without opening an intrusive window.
2. **Minimize to Tray on Close:** Allow the user to close the main window (via the `X` button or `SUPER + Q`) without killing the process or removing the tray icon.

**Key Components:**
- **Flatpak Application:** `io.github.nozwock.Packet`
- **Caelestia User Config:** `~/.config/caelestia/hypr-user.lua`
- **XDG Background Portal Backend:** `~/.local/bin/xdg-desktop-portal-hyprland-background`
- **Systemd User Unit:** `~/.config/systemd/user/xdg-desktop-portal-hyprland-background.service`
- **Hyprland Portal Configuration:** `~/.config/xdg-desktop-portal/hyprland-portals.conf`
- **Portal Backend Definition:** `~/.local/share/xdg-desktop-portal/portals/hyprland-background.portal`
- **D-Bus Service Activation:** `~/.local/share/dbus-1/services/org.freedesktop.impl.portal.desktop.hyprland_background.service`

---

## Context / Root Cause

### 1. Lack of Native XDG Autostart in Hyprland
Under Hyprland and Caelestia, files in `~/.config/autostart/*.desktop` are not automatically executed at login (unlike full desktop environments like GNOME or KDE, as neither `dex` nor `systemd-xdg-autostart-generator` is active). Startup tasks must be declared in the `hyprland.start` event within `~/.config/caelestia/hypr-user.lua`.

### 2. System Tray Disappearance on Window Close (Technical Diagnosis)
In Packet's source code (`src/window.rs`, `close_request` method):

```rust
fn close_request(&self) -> glib::Propagation {
    if self.is_background_allowed.get()
        && self.settings.boolean("run-in-background")
        && !self.should_quit.get()
    {
        tracing::info!("Running Packet in background");
        self.obj().set_visible(false);
        return glib::Propagation::Stop;
    }
    ...
    self.parent_close_request()
}
```

For closing the window to simply hide it (`set_visible(false)`) rather than destroying the window and terminating the GTK application, two conditions must be met:
1. `run-in-background` must be set to `true` in GSettings.
2. `self.is_background_allowed` must be `true`.

`is_background_allowed` is resolved at runtime via a D-Bus request to `org.freedesktop.portal.Background.RequestBackground`.
On Hyprland:
- Neither `xdg-desktop-portal-hyprland` nor `xdg-desktop-portal-gtk` implements `org.freedesktop.impl.portal.Background`.
- Because no loaded backend implemented this interface, `xdg-desktop-portal` did not export `org.freedesktop.portal.Background` on the session bus.
- Packet received a denial error:
  `WARN packet::window: Background request denied: A portal frontend implementing org.freedesktop.portal.Background was not found`
- Packet subsequently flipped `is_background_allowed` to `false` and forced `run-in-background` to `false` in GSettings.
- **Consequence:** Closing the window destroyed the window, killed the process, and removed the tray icon.

---

## Solution Implemented

### 1. Custom Background Portal Backend: `~/.local/bin/xdg-desktop-portal-hyprland-background`
A lightweight Python D-Bus service implementing `org.freedesktop.impl.portal.Background`:
- `GetAppState()`: Returns running app states.
- `NotifyBackground(handle, app_id, name)`: Approves background execution (`result = 1`).
- `EnableAutostart(app_id, enable, commandline, flags)`: Acknowledges autostart registration.

```python
#!/usr/bin/env python3
"""
Custom XDG Desktop Portal backend for org.freedesktop.impl.portal.Background on Hyprland.
Allows applications (such as Packet) to run in the background and minimize to tray.
"""
import sys
import dbus
import dbus.service
import dbus.mainloop.glib
from gi.repository import GLib

BUS_NAME = "org.freedesktop.impl.portal.desktop.hyprland_background"
OBJECT_PATH = "/org/freedesktop/portal/desktop"
INTERFACE = "org.freedesktop.impl.portal.Background"

class BackgroundPortal(dbus.service.Object):
    def __init__(self, bus):
        super().__init__(bus, OBJECT_PATH)

    @dbus.service.method(INTERFACE, in_signature="", out_signature="a{sv}")
    def GetAppState(self):
        return {}

    @dbus.service.method(INTERFACE, in_signature="oss", out_signature="ua{sv}")
    def NotifyBackground(self, handle, app_id, name):
        print(f"[hyprland-background] Allowing background activity for: {app_id} ({name})", flush=True)
        return (dbus.UInt32(0), {"result": dbus.UInt32(1)})

    @dbus.service.method(INTERFACE, in_signature="sbasu", out_signature="b")
    def EnableAutostart(self, app_id, enable, commandline, flags):
        print(f"[hyprland-background] EnableAutostart called for: {app_id}, enable={enable}", flush=True)
        return True

def main():
    dbus.mainloop.glib.DBusGMainLoop(set_as_default=True)
    bus = dbus.SessionBus()
    _name = dbus.service.BusName(BUS_NAME, bus)
    _portal = BackgroundPortal(bus)
    print(f"[hyprland-background] Service started on {BUS_NAME}", flush=True)
    loop = GLib.MainLoop()
    try:
        loop.run()
    except KeyboardInterrupt:
        pass

if __name__ == "__main__":
    main()
```

### 2. D-Bus & XDG Desktop Portal Integration
- **Portal definition** in `~/.local/share/xdg-desktop-portal/portals/hyprland-background.portal`:
  ```ini
  [portal]
  DBusName=org.freedesktop.impl.portal.desktop.hyprland_background
  Interfaces=org.freedesktop.impl.portal.Background;
  UseIn=hyprland;Hyprland;
  ```
- **D-Bus service activation** in `~/.local/share/dbus-1/services/org.freedesktop.impl.portal.desktop.hyprland_background.service`:
  ```ini
  [D-BUS Service]
  Name=org.freedesktop.impl.portal.desktop.hyprland_background
  Exec=/home/mathieu/.local/bin/xdg-desktop-portal-hyprland-background
  SystemdService=xdg-desktop-portal-hyprland-background.service
  ```
- **Portal configuration** in `~/.config/xdg-desktop-portal/hyprland-portals.conf`:
  ```ini
  [preferred]
  default=hyprland;gtk
  org.freedesktop.impl.portal.Background=hyprland-background
  ```
- **Persistent Flatpak permission**:
  ```fish
  flatpak permission-set background background io.github.nozwock.Packet yes
  ```

### 3. Systemd User Service
The user unit `~/.config/systemd/user/xdg-desktop-portal-hyprland-background.service` ensures the service starts with the graphical session and automatically restarts on failure:

```ini
[Unit]
Description=Portal backend for Background interface on Hyprland
PartOf=graphical-session.target
After=graphical-session.target

[Service]
Type=dbus
BusName=org.freedesktop.impl.portal.desktop.hyprland_background
ExecStart=/home/mathieu/.local/bin/xdg-desktop-portal-hyprland-background
Restart=on-failure

[Install]
WantedBy=graphical-session.target
```

### 4. Hyprland Startup Binding in `~/.config/caelestia/hypr-user.lua`
```lua
-- ─── Autostart Applications ──────────────────────────────────────────────────
-- Launch Packet (Nearby Share / Quick Share) in background mode
hl.on("hyprland.start", function()
    hl.exec_cmd("flatpak run io.github.nozwock.Packet --background")
end)
```

### 5. GSettings Preferences for Packet
```fish
flatpak run --command=gsettings io.github.nozwock.Packet set io.github.nozwock.Packet run-in-background true
flatpak run --command=gsettings io.github.nozwock.Packet set io.github.nozwock.Packet auto-start true
```

---

## Verification & Status

1. **Verify Background Portal interface is exposed on session D-Bus:**
   ```fish
   busctl --user introspect org.freedesktop.portal.Desktop /org/freedesktop/portal/desktop | grep -i "Background"
   ```
   *Expected output:*
   ```text
   org.freedesktop.portal.Background          interface -                 -            -
   .RequestBackground                         method    sa{sv}            o            -
   ```

2. **Verify Packet receives permission from portal:**
   ```fish
   flatpak run io.github.nozwock.Packet
   ```
   *Expected log output:*
   ```text
   DEBUG packet::window: Background request successful response=Background { background: true, autostart: false }
   ```

3. **Verify Window Close Behavior:**
   - Open the Packet window (via application launcher or clicking the tray icon).
   - Close the window using the `X` button or `SUPER + Q`.
   - **Result:** The window closes immediately, but the `packet` process remains active and the system tray icon stays in the top bar.
   - Clicking the tray icon reopens the window instantly.

4. **Verify Backend Service Status:**
   ```fish
   systemctl --user status xdg-desktop-portal-hyprland-background.service
   ```

---

## Maintenance / Handy Commands

| Action | Command (Fish syntax) |
| :--- | :--- |
| **Check portal backend logs** | `journalctl --user -u xdg-desktop-portal-hyprland-background.service -f` |
| **Restart portal services** | `systemctl --user restart xdg-desktop-portal-hyprland-background.service xdg-desktop-portal.service` |
| **Start Packet in background** | `flatpak run io.github.nozwock.Packet --background &` |
| **Start Packet with main window** | `flatpak run io.github.nozwock.Packet &` |
| **Inspect Packet Flatpak permissions** | `flatpak permission-show io.github.nozwock.Packet` |
| **Kill Packet** | `pkill -f packet` |
| **Reload Hyprland config** | `hyprctl reload` |
