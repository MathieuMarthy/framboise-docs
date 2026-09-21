# Floating Shortcuts Cheatsheet Window (Caelestia & Fish)

## 1. Overview
This feature provides an instant floating HUD window (cheatsheet) summarizing all keybindings and shell shortcuts configured on workstation `framboise`:
- Interactive shell shortcuts and abbreviations in **Fish** (`git`, `eza`, `zoxide`, line editing, and history search).
- Desktop shell keybindings in **Caelestia / Hyprland** (launcher, window management, AZERTY workspace switching, scratchpads, media controls, and desktop utilities).

The cheatsheet can be triggered globally via keybindings (`SUPER + H` or `SUPER + F1`), or from any terminal using `cheatsheet` / `raccourcis`.

---

## 2. Architecture & Workflow

```
[ SUPER + H / SUPER + F1 / CLI ]
                │
                ▼
      ~/.local/bin/cheatsheet
                │
         (Process Check)
       ├── Already open? ──► Close window (Toggle off)
       └── Closed? ────────► Launch foot -a cheatsheet
                                    │
                                    ▼
                         Hyprland tag "+float_70_80"
                        (Floating window, 70%x80%, centered)
                                    │
                                    ▼
                         mdcat -p (less pager)
                         /home/mathieu/docs/shortcuts-cheatsheet.md
```

### Key Features:
- **Toggle behavior**: Pressing `SUPER + H` / `SUPER + F1` while the window is active immediately closes it.
- **Fast exit**: Pressing `q` inside the window exits instantly.
- **Smooth navigation**: Scroll through content with arrow keys `↑`/`↓`, `Page Up`/`Page Down`, or the mouse wheel.
- **Text search**: Press `/`, enter search query, hit `Enter` (`n` jumps to next match).
- **Rich formatting**: Rendered via `mdcat` for syntax highlighting, colored borders, and terminal-native styling.

---

## 3. Implementation Details

### 1. Markdown Cheatsheet Source: `/home/mathieu/docs/shortcuts-cheatsheet.md`
Contains the complete reference structured in two clean sections: Interactive Shell (Fish) and Desktop Shell (Caelestia & Hyprland).

### 2. Launcher Script: `~/.local/bin/cheatsheet`
```bash
#!/usr/bin/env bash
# ~/.local/bin/cheatsheet
# Toggle a floating cheatsheet window displaying Caelestia and Fish shortcuts.

DOC_FILE="/home/mathieu/docs/shortcuts-cheatsheet.md"

# Toggle behavior: if already open, close it
if pgrep -f "foot.*-a cheatsheet" > /dev/null 2>&1; then
    pkill -f "foot.*-a cheatsheet"
    exit 0
fi

# Ensure documentation file exists
if [[ ! -f "$DOC_FILE" ]]; then
    notify-send -u critical "Cheatsheet" "Documentation file $DOC_FILE not found."
    exit 1
fi

# Launch in foot floating window with mdcat and less pager
exec foot -a cheatsheet -T "Shortcuts Cheatsheet" -o "window.dimensions=1100x750" sh -c "LESS='-R -~ -K -i' mdcat -p '$DOC_FILE'"
```
*CLI symlink:* `~/.local/bin/raccourcis -> ~/.local/bin/cheatsheet`.

### 3. Hyprland Rules & Binds: `~/.config/caelestia/hypr-user.lua`
```lua
-- ─── Cheatsheet (Shortcuts HUD) ───────────────────────────────────────────────
hl.window_rule({ match = { class = "cheatsheet" }, tag = "+float_70_80" })
hl.bind("SUPER + h", hl.dsp.exec_cmd("/home/mathieu/.local/bin/cheatsheet"))
hl.bind("SUPER + F1", hl.dsp.exec_cmd("/home/mathieu/.local/bin/cheatsheet"))
```

---

## 4. Verification & Testing

1. **Reload Hyprland**:
   ```bash
   hyprctl reload
   ```
2. **Test Keybindings**:
   - Press `SUPER + H` or `SUPER + F1`: floating cheatsheet opens centered.
   - Press `q`: cheatsheet closes immediately.
   - Press `SUPER + H` twice: window opens, then closes (toggle).
3. **Test from Shell**:
   ```fish
   cheatsheet
   # or
   raccourcis
   ```

---

## 5. Maintenance & Useful Commands

- **Update shortcuts content**: Edit `/home/mathieu/docs/shortcuts-cheatsheet.md` directly. Changes take effect on next launch without restarting any service.
- **Inspect registered Hyprland keybinds**:
  ```fish
  hyprctl binds | grep -Ei "key: (h|F1)$"
  ```
