# Caelestia Clipboard — cliphist Configuration

## Overview

Documents the clipboard history setup for the Caelestia desktop on `framboise`.
Caelestia uses **cliphist** (a Wayland clipboard manager backed by a SQLite database)
combined with **wl-paste** watchers to capture both text and image clipboard events.

Two behaviours are configured here:
1. **Startup wipe** — the history database is cleared on every Hyprland login.
2. **Image support** — PNG/JPEG screenshots and copied images are stored alongside text.

**Affected files:**
- `~/.config/caelestia/hypr-user.lua` — user-managed Hyprland config (never overwritten by `caelestia update`)
- `~/.config/hypr/hyprland/execs.lua` — upstream Caelestia file (read-only reference; do **not** edit)

> [!IMPORTANT]
> `~/.config/hypr/` is entirely managed by `caelestia update` and will be overwritten on
> every update. All user customizations must live in `~/.config/caelestia/hypr-user.lua`.

---

## Context / Root Cause

By default cliphist accumulates entries indefinitely (up to `max-items`, default 750).
Entries survive reboots because the database is stored on disk at
`~/.cache/cliphist/db`.

Image support is opt-in: a second `wl-paste --type image` watcher must be started
explicitly alongside the default text watcher. Caelestia already ships both watchers
in its `execs.lua`.

---

## Solution Implemented

### File: `~/.config/caelestia/hypr-user.lua`

A `hyprland.start` listener was appended to the user config file. It wipes the
cliphist database at every Hyprland startup, **before** the wl-paste daemons (launched
by Caelestia's own `execs.lua`) have a chance to add new entries.

```lua
-- ─── Clipboard ────────────────────────────────────────────────────────────────
-- Wipe cliphist history on every Hyprland startup so the session starts clean.
-- The watchers (text + image) are already launched by Caelestia's execs.lua.
hl.on("hyprland.start", function()
    hl.exec_cmd("cliphist wipe")
end)
```

### Image support

Image capture was already enabled upstream by Caelestia in `execs.lua`:

```lua
hl.exec_cmd("wl-paste --type text --watch cliphist store")
hl.exec_cmd("wl-paste --type image --watch cliphist store")
```

No user-side change is needed for images. This is confirmed by entries like
`[[ binary data 372 KiB png 904x1155 ]]` appearing in `cliphist list`.

---

## Verification & Status

Check the DB size right after login (should be 0 or very few entries):

```bash
cliphist list | wc -l
```

Copy an image (e.g. screenshot with `grimblast copy area`) then confirm it appears:

```bash
cliphist list | grep "binary data"
```

Expected output example:
```
1  [[ binary data 372 KiB png 904x1155 ]]
```

Open the clipboard picker with the configured shortcut (default `SUPER+V`) to
browse both text and image entries visually.

---

## Maintenance / Handy Commands

| Command | Purpose |
| :--- | :--- |
| `cliphist list` | List all stored clipboard entries |
| `cliphist wipe` | Manually wipe the entire history |
| `cliphist list \| wc -l` | Count entries |
| `ls -lh ~/.cache/cliphist/db` | Check database file size |
| `cliphist --help` | Full option reference (config path, max-items, etc.) |

### Customising limits

Create `~/.config/cliphist/config` to override defaults without touching any
managed file:

```text
max-items = 200
max-dedupe-search = 50
```

This caps the in-session history to 200 entries while still wiping on every
new login.

### Update safety

`caelestia update` tracks and overwrites every file under `~/.config/hypr/`.
The wipe logic sits in `~/.config/caelestia/hypr-user.lua` which is **never**
in the `deployed_files` list and is therefore update-safe.
