# AGENTS.md — Environment & Documentation Guide for `framboise`

> **Note for AI Agents & Automated Tools**: This document provides essential operational context regarding the host system, desktop stack, shell conventions, and the contents/purpose of this documentation directory (`/home/mathieu/docs`). Read this file before executing system commands, suggesting scripts, or adding documentation.

---

## ⚠️ MANDATORY AGENT RULE: ALWAYS DOCUMENT SYSTEM CHANGES

> [!IMPORTANT]
> **Strict Operational Requirement**:
> Whenever an agent creates, modifies, installs, or troubleshoots any system service, package, configuration, script, or hardware workaround on this system (`framboise`), the agent **MUST ALWAYS**:
> 1. **Create or update a dedicated runbook file** in `/home/mathieu/docs/<topic-name>.md` explaining the problem, root cause, implementation, verification steps, and maintenance commands.
> 2. **Update the index table below** in this `AGENTS.md` file to register the new or modified documentation file.
>
> *Rule definition*: See [`.agents/rules/system-documentation.md`](file:///home/mathieu/docs/.agents/rules/system-documentation.md).

---

## 1. Directory Purpose & Contents

This folder (`/home/mathieu/docs`) is the dedicated knowledge base and runbook repository for the personal workstation **`framboise`**. It tracks hardware workarounds, desktop environment customizations, and system-level fixes applied to maintain a functional, bleeding-edge Linux desktop.

### Existing Documentation Index

| File | Purpose / Scope | Key Components |
| :--- | :--- | :--- |
| [`bluetooth-caelestia-fix.md`](file:///home/mathieu/docs/bluetooth-caelestia-fix.md) | Resolves Intel BE201 PCIe probe race conditions and BlueZ agent pairing failures in Caelestia. | `btintel-fix.service`, `bluetooth-agent.service`, BlueZ `bluetoothctl`, PipeWire / WirePlumber |
| [`hyprland-azerty-workspaces.md`](file:///home/mathieu/docs/hyprland-azerty-workspaces.md) | Maps top-row AZERTY keys (`&` through `à`) to Hyprland workspaces and resolves key collisions. | `~/.config/caelestia/hypr-user.lua`, XKB keysyms, Caelestia `wsaction` helpers |
| [`battery-alert.md`](file:///home/mathieu/docs/battery-alert.md) | Limite de charge matérielle à 80%/100% via `SUPER+B` (mode Dell `Custom`, persistance veille/reboot) + notifications Caelestia. | `~/.local/bin/battery-alert`, `~/.local/bin/battery-charge-limit`, `battery-charge-limit.service`, `battery-alert.timer`, `charge_types`, udev, systemd-sleep |
| [`logout-sddm-fix.md`](file:///home/mathieu/docs/logout-sddm-fix.md) | Fix pour le bouton logout Caelestia qui ne relançait pas SDDM (écran noir). Sortie propre Hyprland via Lua `hl.dsp.exit()` et basculement VT vers SDDM via logind. | `~/.local/bin/hyprland-logout`, `~/.config/caelestia/shell.json`, `hl.dsp.exit()`, logind `SwitchTo`, SDDM VT2 |
| [`sddm-default-session-hyprland.md`](file:///home/mathieu/docs/sddm-default-session-hyprland.md) | Correction de la session par défaut dans SDDM avec le thème Caelestia en utilisant `sessionModel.lastIndex` au lieu de `selectedIndex: 0`. | `/usr/share/sddm/themes/caelestia/Main.qml`, `sessionModel.lastIndex`, SDDM, Hyprland |
| [`caelestia-clipboard.md`](file:///home/mathieu/docs/caelestia-clipboard.md) | Configuration de l'historique du presse-papier cliphist (nettoyage au démarrage et support images). | `~/.config/caelestia/hypr-user.lua`, `cliphist wipe`, wl-paste daemons |
| [`caelestia-special-workspaces.md`](file:///home/mathieu/docs/caelestia-special-workspaces.md) | Configuration des workspaces spéciaux (scratchpads) pour la communication (Vesktop, ZapFast via `SUPER+D`) et la musique (Spotifast via `SUPER+M`). | `~/.config/caelestia/hypr-user.lua`, `~/.config/caelestia/cli.json`, `special:communication`, `special:music` |
| [`shortcuts-cheatsheet-window.md`](file:///home/mathieu/docs/shortcuts-cheatsheet-window.md) | Instant floating cheatsheet window summarizing all shortcuts (Fish & Caelestia/Hyprland) via `SUPER+H` or `SUPER+F1`. | `~/.local/bin/cheatsheet`, `~/.config/caelestia/hypr-user.lua`, `mdcat`, `foot` |


---

## 2. Host System & Architecture

| Component | Detail |
| :--- | :--- |
| **Hostname** | `framboise` |
| **Operating System** | **CachyOS Linux** (Arch Linux-based, rolling release) |
| **Kernel** | `7.2.x-cachyos` (Dynamic Preemption, performance-tuned for x86_64) |
| **CPU Architecture** | `x86_64` — Intel(R) Core(TM) Ultra 5 236V (Lunar Lake) |
| **Audio Subsystem** | PipeWire + WirePlumber (`wpctl`) |
| **Init & Services** | `systemd` (system units in `/etc/systemd/system/`, user units in `~/.config/systemd/user/`) |
| **Package Managers** | `pacman` (official repositories), **`paru`** (AUR helper). *Note: `yay` is not installed.* |
| **Installed Editors** | `micro`, `nano`, `vim`. *Note: `nvim` (Neovim) is not installed.* |
| **Python Runtime** | Python 3.14 (`/usr/bin/python3`) |

---

## 3. Desktop Environment & Input Stack

- **Display Server / Compositor**: **Hyprland** (Wayland session, version `0.56+`)
- **Desktop Shell UI**: **Caelestia** (custom shell built with Quickshell)
- **Terminal Emulator**: `foot`
- **Physical Keyboard Layout**: **French AZERTY (`fr`)**

### Crucial AZERTY Considerations for Agents:
1. **Top Row**: On standard AZERTY keyboards, the top row defaults to symbols and accented letters (`&`, `é`, `"`, `'`, `(`, `-`, `è`, `_`, `ç`, `à`). Numbers `1-0` require `Shift`.
2. **Hyprland Lua Configuration**: Keysym identifiers must be used in `hl.bind` rather than literal characters (e.g., `ampersand`, `eacute`, `minus`).
3. **Shortcuts & Binds**: When configuring keybindings or debugging shortcuts, beware that `SUPER + Minus` is physical key 6 on AZERTY, conflicting with default QWERTY window shrink binds.

---

## 4. Shell Environment & Command Rules

The user's default interactive shell is **fish** (`/bin/fish`, version `4.9+`).

```text
SHELL = /bin/fish
User Home = /home/mathieu
Fish Config = ~/.config/fish/config.fish
Caelestia User Fish Config = ~/.config/caelestia/user-config.fish
```

### Critical Rules for AI Agents Running Commands

1. **Subshell Execution vs Interactive Shell**:
   - Automated tool calls, scripts, and subprocesses run under standard POSIX `/bin/sh` or `/bin/bash`.
   - **However**, any command, alias, or script recommendation written for the **user** to paste into their terminal MUST be compatible with **Fish**.

2. **Fish Syntax Caveats**:
   - **Setting Environment Variables**:
     - *Bash*: `export FOO="bar"`
     - *Fish*: `set -gx FOO "bar"`
   - **Prepending to `$PATH`**:
     - *Fish*: `set -gx PATH "/path/to/bin" $PATH` or `fish_add_path /path/to/bin`
   - **Conditionals & Loops**:
     - *Fish*: `if test -f file; ...; end` (no `then`, `fi`, `do`, `done`).
   - **Command Substitution**:
     - *Fish*: `$(cmd)` or `(cmd)` are both supported in fish 3.4+, but `(cmd)` is traditional.

3. **Active Fish Integrations & Aliases**:
   - Prompt: `starship`
   - Directory navigation: `zoxide` (aliased to `cd`), `direnv`
   - File listing: `eza` (aliased to `ls`, `l`, `ll`, `la`, `lla`)
   - Git shortcuts: `lazygit` (`lg`), `gs` (`git status`), `gd` (`git diff`), `ga` (`git add .`), `gc` (`git commit -am`), etc.

---

## 5. Configuration & Dotfile Map

| Component | Path | Notes |
| :--- | :--- | :--- |
| **Caelestia Hyprland Lua** | `~/.config/caelestia/hypr-user.lua` | Custom user keybinds and input configuration |
| **Caelestia Hyprland Vars** | `~/.config/caelestia/hypr-vars.lua` | Variable overrides |
| **Caelestia Fish Custom** | `~/.config/caelestia/user-config.fish` | Extra shell configuration sourced by fish |
| **Caelestia CLI Config** | `~/.config/caelestia/cli.json` | Caelestia CLI settings |
| **Fish Config** | `~/.config/fish/config.fish` | Main fish interactive configuration |
| **Bluetooth Agent Service** | `~/.config/systemd/user/bluetooth-agent.service` | User systemd unit for BlueZ auto-pairing |
| **Bluetooth Agent Script** | `~/.local/bin/bluetooth-agent` | Python wrapper around `bluetoothctl --agent NoInputNoOutput` |
| **Intel BT Fix Service** | `/etc/systemd/system/btintel-fix.service` | Root systemd service for BE201 boot race recovery |

---

## 6. Guidelines & Workflow for Modifying the System

Whenever an agent performs any change, tweak, installation, or fix on `framboise`:

1. **Perform the Change**:
   - Apply configuration or script changes in the appropriate target directory (e.g., `~/.config/`, `/etc/`, `~/.local/bin/`).
   - Validate service or syntax integrity before reporting completion.

2. **Author or Update the Documentation File**:
   - Create or update the relevant Markdown file under `/home/mathieu/docs/<topic>.md`.
   - Ensure the document adheres to the standard 5-part structure:
     - **Overview**: Problem statement, target hardware, affected components.
     - **Context / Root Cause**: Technical diagnosis (driver errors, logs, protocol failures).
     - **Solution Implemented**: Full configuration snippets or scripts with exact file paths.
     - **Verification & Status**: Commands to verify that the fix works.
     - **Maintenance / Handy Commands**: Diagnostic and recovery commands.

3. **Update the Index in `AGENTS.md`**:
   - Add a row to the **Existing Documentation Index** in Section 1 with a clickable link, brief description, and key components.

4. **Code Blocks & Syntax Highlighting**:
   - Always declare syntax languages: `lua`, `python`, `ini`, `bash`, `fish`, `text`, `qml`, `json`.
   - Distinguish between commands requiring `sudo` (system-level) and rootless user commands (`systemctl --user`).
   - Commands recommended to the user must be valid in **Fish** shell.
