# 📖 Shortcuts Cheatsheet — Caelestia & Fish Shell

> **Window Navigation & Controls**:
> - **Quit**: press `q` or press `SUPER + H` / `SUPER + F1` again
> - **Scroll**: `↑` / `↓` arrow keys, `Page Up` / `Page Down`, or mouse wheel
> - **Search**: press `/`, type your query and press `Enter` (`n` for next match)

---

## 1. 🪟 Caelestia Desktop Shell & Hyprland

### 🚀 Application Launchers & Tools
| Shortcut | Application / Action |
| :--- | :--- |
| `SUPER + T` | Terminal emulator (`foot`) |
| `SUPER + W` | Web browser (`firefox`) |
| `SUPER + C` | Code editor (`codium`) |
| `SUPER + E` | File manager (`thunar`) |
| `CTRL + ALT + V` | Audio mixer settings (`pwvucontrol`) |
| `SUPER + B` | Toggle hardware battery charge limit (**80% ↔ 100%**) |
| `SUPER + H` / `SUPER + F1` | **This floating cheatsheet** (open / close toggle) |

### 🎛️ Caelestia UI Controls
| Shortcut | Action |
| :--- | :--- |
| `SUPER` *(alone)* | Caelestia application launcher |
| `SUPER + N` | Sidebar & notification center |
| `SUPER + K` | Toggle all panels view (`showall`) |
| `CTRL + ALT + Delete` | Session menu (Power off, reboot, logout) |
| `SUPER + L` | Lock screen |
| `SUPER + SHIFT + L` | System sleep (`suspend-then-hibernate`) |
| `CTRL + ALT + C` | Clear all notifications |
| `CTRL + SUPER + ALT + R` | Restart Caelestia shell in-place |

### 📋 Clipboard, Screenshots & Utilities
| Shortcut | Action |
| :--- | :--- |
| `SUPER + V` | Clipboard history (`cliphist` via Fuzzel picker) |
| `SUPER + ALT + V` | Delete selected item from clipboard history |
| `SUPER + .` *(period)* | Emoji and glyph picker |
| `SUPER + SHIFT + C` | Screen color picker (`hyprpicker`) |
| `Print` | Full-screen screenshot |
| `SUPER + SHIFT + S` | Freeze-screen screenshot |
| `SUPER + SHIFT + ALT + S` | Region screenshot selection |
| `CTRL + ALT + R` | Start screen recording |
| `SUPER + ALT + R` | Start screen recording with audio |

### 🖥️ Workspaces (French AZERTY Layout)
| Shortcut | Action |
| :--- | :--- |
| `SUPER + &` to `à` *(1 to 10)* | Switch to workspace 1 through 10 |
| `SUPER + SHIFT + &` to `à` | Move active window to workspace 1 through 10 |
| `SUPER + ALT + &` to `à` | Move active window to workspace 1 through 10 |
| `SUPER + Scroll Up/Down` | Switch to previous / next workspace |
| `SUPER + Page_Up / Page_Down` | Switch to previous / next workspace |

### 📌 Special Workspaces (Scratchpads)
*Press to show the scratchpad, press again to hide it:*

| Shortcut | Dedicated Application / Purpose |
| :--- | :--- |
| `SUPER + D` | **Communication** (`vesktop`, `zapfast`) |
| `SUPER + M` | **Music** (`spotifast`, `fastpotify`, `spotify`) |
| `SUPER + R` | **Notes & Tasks** (`obsidian`) |
| `CTRL + SHIFT + Escape` | **System Monitor** (`btop`) |
| `SUPER + S` | General scratchpad |

### 🪟 Window Management
| Shortcut | Action |
| :--- | :--- |
| `SUPER + Q` | Close active window |
| `SUPER + F` | Toggle full screen |
| `SUPER + ALT + F` | Maximized full screen (with outer margins) |
| `SUPER + ALT + Space` | Toggle **floating / tiled** mode |
| `SUPER + P` | Pin window (remain visible across all workspaces) |
| `SUPER + ,` *(comma)* | Group windows into tabbed layout |
| `SUPER + U` | Ungroup active window |
| `ALT + Tab` / `SHIFT + ALT + Tab` | Cycle through windows |
| `SUPER + Arrow keys` | Move focus in specified direction |
| `SUPER + SHIFT + Arrow keys` | Move active window position |
| `SUPER + Left Click` *(drag)* | Move floating window |
| `SUPER + Right Click` *(drag)* | Resize floating window |

---

## 2. 🐚 Interactive Shell (Fish)

### 🌿 Git Shortcuts (`abbr`)
*Type the abbreviation and hit Space to expand it automatically:*

| Shortcut | Expanded Command | Description |
| :--- | :--- | :--- |
| `lg` | `lazygit` | Open LazyGit terminal UI |
| `gs` | `git status` | Show working tree status |
| `gd` | `git diff` | Show changes between commits, commit and working tree |
| `ga` | `git add .` | Stage all current changes |
| `gc` | `git commit -am` | Commit staged and tracked changes with message |
| `gl` | `git log` | View commit log |
| `gsh` | `git show` | Show details of the latest commit |
| `gp` | `git push` | Push commits to upstream repository |
| `gpl` | `git pull` | Fetch and merge changes from upstream repository |
| `gb` | `git branch` | List local branches |
| `gbd` | `git branch -d` | Delete a local branch safely |
| `gsw` | `git switch` | Switch branches |
| `gsm` | `git switch main` | Switch directly to `main` branch |
| `gco` | `git checkout` | Checkout a branch or paths to working tree |
| `gst` | `git stash` | Stash changes in working directory |
| `gsp` | `git stash pop` | Apply and drop stashed changes |

### 📁 Files & Directory Navigation
| Shortcut | Command / Tool | Description |
| :--- | :--- | :--- |
| `cd <dir>` | `zoxide` | Smart navigation to frequently used directories |
| `l` | `ls` (`eza --icons -1`) | Single column list with file icons |
| `ll` | `ls -l` | Detailed list (permissions, size, modification date) |
| `la` | `ls -a` | List including hidden files |
| `lla` | `ls -la` | Complete detailed list including hidden files |

### ⌨️ Fish Line Editing & Terminal Keys
| Key(s) | Action |
| :--- | :--- |
| `→` or `Ctrl + F` | Accept entire gray autosuggestion |
| `Alt + →` | Accept next word of autosuggestion |
| `↑` / `↓` | Search history filtered by current prefix |
| `Ctrl + R` | Interactive command history search |
| `Tab` | Interactive completion menu / pager (navigate with arrows) |
| `Ctrl + A` / `Ctrl + E` | Jump to beginning / end of line |
| `Ctrl + U` | Clear from cursor to beginning of line |
| `Ctrl + K` | Clear from cursor to end of line |
| `Ctrl + W` (or `Alt + Backspace`) | Erase preceding word |
| `Alt + D` | Erase next word |
| `Ctrl + L` | Clear terminal screen |
| `Ctrl + C` | Cancel current command input |