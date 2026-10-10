# Fix for Missing Notification Icons (Black/Pink Checkerboard) after Qt 6.12 Upgrade

## 1. Overview

On **`framboise`** (CachyOS Linux on Intel Core Ultra 5 Lunar Lake, running Hyprland with Caelestia / Quickshell), desktop notifications using standard system icons—such as system update notifications (`system-software-update` from `~/.local/bin/check-updates.sh`) and system reboot warnings (`system-reboot` from `/usr/share/libalpm/scripts/cachyos-reboot-required`)—suddenly lost their icons and displayed a black-and-pink mosaic checkerboard pattern ("missing texture" box).

Affected components:
- **Quickshell** (`quickshell-git`) and Caelestia shell notifications (`Notification.qml`, `ColouredIcon.qml`).
- **`qtengine`** (`aur/qtengine`): Qt platform theme plugin (`libqt6engine-plugin.so`).
- **Qt6 base libraries** (`qt6-base 6.12.0-2.1`).
- Notification scripts: `~/.local/bin/check-updates.sh` and `/usr/share/libalpm/scripts/cachyos-reboot-required`.

---

## 2. Context & Root Cause Analysis

### Investigation & Discovery

1. **Log Analysis**:
   Inspecting Quickshell's runtime logs (`/run/user/1000/quickshell/by-id/<id>/log.log`) revealed repeated warnings:
   ```text
   WARN: Could not load icon "system-software-update" at size QSize(48, 48) from request
   WARN: Could not load icon "system-reboot" at size QSize(27, 27) from request
   ```

2. **The "Missing Texture" Checkerboard**:
   Examining the Quickshell source code (`src/core/iconimageprovider.cpp`) identified the exact source of the black and purple/pink checkerboard:
   ```cpp
   QPixmap IconImageProvider::missingPixmap(const QSize& size) {
       ...
       auto pixmap = QPixmap(width, height);
       pixmap.fill(QColorConstants::Black);
       auto painter = QPainter(&pixmap);
       auto halfWidth = width / 2;
       auto halfHeight = height / 2;
       auto purple = QColor(0xd900d8);
       painter.fillRect(halfWidth, 0, halfWidth, halfHeight, purple);
       painter.fillRect(0, halfHeight, halfWidth, halfHeight, purple);
       return pixmap;
   }
   ```
   When `QIcon::fromTheme(iconName)` returns a null pixmap, Quickshell draws this fallback checkerboard pattern.

3. **Qt Platform Theme Rejection**:
   Hyprland configures Qt's platform theme via `hl.env("QT_QPA_PLATFORMTHEME", "qtengine")` in `~/.config/hypr/hyprland/env.lua`.
   Running a Qt6 diagnostic with `QT_DEBUG_PLUGINS=1` exposed why the theme was not working:
   ```text
   qt.core.plugin.loader: Found metadata in lib /usr/lib/qt6/plugins/platformthemes/libqt6engine-plugin.so
   qt.core.plugin.factoryloader: Ignoring QPA plugin due to mismatching Qt versions 396288 396032
   ```
   - `396032` = `0x60B00` = **Qt 6.11.0** (version when `qtengine` was originally built on Sep 26).
   - `396288` = `0x60C00` = **Qt 6.12.0** (installed during the major system upgrade on Oct 7).

   Qt strictly enforces minor-version matching for QPA platform plugins. Because `libqt6engine-plugin.so` was built against Qt 6.11, Qt 6.12 silently rejected the plugin.

4. **Fallback to `hicolor`**:
   Because `qtengine` was rejected and no other platform theme was active, Qt6 defaulted its icon theme to **`hicolor`** instead of **`Papirus-Dark`**.
   Neither `system-software-update` nor `system-reboot` exist within the minimal `hicolor` theme, causing `QIcon::fromTheme()` to fail and return null pixmaps for both icons.

---

## 3. Solution Implemented

### Step 1: Explicit Quickshell Icon Theme Configuration (Resilience Fix)

Quickshell natively supports reading the `QS_ICON_THEME` environment variable at startup and applying it via `QIcon::setThemeName()`. This makes Quickshell resilient against any future Qt platform theme plugin ABI mismatches or breakage.

In `~/.config/hypr/hyprland/env.lua`, exported `QS_ICON_THEME`:

```lua
-- Themes
hl.env("QT_QPA_PLATFORMTHEME", "qtengine")
hl.env("QS_ICON_THEME", "Papirus-Dark")
hl.env("QT_WAYLAND_DISABLE_WINDOWDECORATION", "1")
hl.env("QT_AUTO_SCREEN_SCALE_FACTOR", "1")
hl.env("XCURSOR_THEME", vars.cursorTheme)
hl.env("XCURSOR_SIZE", vars.cursorSize)
```

### Step 2: Rebuilding `qtengine` for Qt 6.12.0 (System-wide Fix)

The AUR package `qtengine` was recompiled against the newly installed Qt 6.12.0 libraries:
- Compiled package created: `~/.cache/packages/qtengine-0.2.2-1-x86_64.pkg.tar.zst`
- Verification with diagnostic test showed `Theme name: "Papirus-Dark"`, `system-software-update isNull: false`, and `system-reboot isNull: false`.

### Step 3: Reloading Caelestia Shell

Caelestia shell was reloaded with the new environment:
```fish
caelestia shell -k; and sleep 0.5; and caelestia shell -d
```

---

## 4. Verification & Status

1. **Icon Resolution in Caelestia**:
   Triggered test notifications using `notify-send`:
   ```fish
   notify-send --app-name="System Update" --icon="system-software-update" "Updates Available" "Testing update icon"
   notify-send --app-name="CachyOS Update" --icon="system-reboot" "Reboot recommended!" "Testing reboot icon"
   ```

2. **Quickshell Logs**:
   Inspected runtime log at `/run/user/1000/quickshell/by-id/<id>/log.log`:
   - No `WARN: Could not load icon` messages appeared.
   - Notification toasts showed full-color crisp icons (Papirus package icon and reboot icon).
   - Zero missing texture checkerboards.

---

## 5. Maintenance & Handy Commands

### Install the Recompiled `qtengine` Package System-Wide

To update `/usr/lib/qt6/plugins/platformthemes/libqt6engine-plugin.so` so that *all* Qt6 apps (outside of Quickshell) also have proper theme integration:

```fish
sudo pacman -U ~/.cache/packages/qtengine-0.2.2-1-x86_64.pkg.tar.zst
```

Or rebuild directly via `paru`:

```fish
paru -S --rebuild qtengine
```

### Diagnose Qt Plugin & Theme Loading

To verify whether Qt is loading the platform theme plugin or rejecting it:

```fish
QT_DEBUG_PLUGINS=1 QT_QPA_PLATFORM=wayland quickshell --version 2>&1 | grep -iE 'qtengine|version|error'
```

### Restart Caelestia Shell

If Quickshell ever needs to be reloaded:

```fish
caelestia shell -k; and sleep 0.5; and caelestia shell -d
```
