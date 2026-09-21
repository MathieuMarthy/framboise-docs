# Caelestia Special Workspaces (Scratchpads) — Communication & Musique

## Overview

Ce document détaille la configuration des **workspaces spéciaux** (ou *scratchpads*) dans Caelestia / Hyprland sur `framboise`.
Les workspaces spéciaux sont des tiroirs escamotables accessibles par raccourci clavier (`SUPER + <touche>`), permettant d'afficher ou masquer des applications d'arrière-plan sans encombrer les bureaux numérotés (1 à 10).

**Applications configurées :**
- **Communication (`special:communication`)** : `vesktop` (Discord) et `zapfast` (WhatsApp client) via **`SUPER + D`**
- **Musique (`special:music`)** : `spotifast` / `fastpotify` via **`SUPER + M`**
- **Scratchpad générique (`special:special`)** : Tiroir temporaire standard via **`SUPER + S`**

**Fichiers affectés :**
- `~/.config/caelestia/hypr-user.lua` — Règles de placement de fenêtres Hyprland (persistant, non écrasé par `caelestia update`)
- `~/.config/caelestia/cli.json` — Configuration des bascules (*toggles*) Caelestia CLI / shell
- `~/.config/hypr/*` — Fichiers amont Caelestia (**lecture seule**, gérés par `caelestia update`)

> [!IMPORTANT]
> Ne modifiez jamais directement les fichiers dans `~/.config/hypr/`. Ils sont gérés par les mises à jour amont (`caelestia update`) et seront écrasés. Toute personnalisation utilisateur doit impérativement être placée dans `~/.config/caelestia/` (`hypr-user.lua`, `hypr-vars.lua`, `cli.json`).

---

## Context / Root Cause

1. **Placement automatique dans un workspace spécial :**
   Dans la configuration amont de Caelestia (`~/.config/hypr/hyprland/rules.lua`), les clients Discord (`discord`, `equibop`, `vesktop`) sont balisés `communication_app` et automatiquement dirigés vers `workspace = "special:communication"`.
2. **Confusion entre `SUPER + S` et `SUPER + D` :**
   - Le raccourci **`SUPER + S`** contrôle le scratchpad générique (`special:special`).
   - Le raccourci **`SUPER + D`** contrôle le tiroir de communication (`special:communication`).
   - Si aucun tiroir n'est ouvert, `SUPER + S` ouvre `special:special`, qui est vide (le fameux "deuxième workspace").
   - Si un tiroir est ouvert, `SUPER + S` le masque. Au ré-appui, il ouvre `special:special` et non pas Vesktop.
3. **Nouveaux clients locaux :**
   - **`zapfast`** (client WhatsApp natif) utilise la classe WM `zapfast`.
   - **`spotifast`** (client Spotify natif rapide) utilise l'exécutable `fastpotify` et la classe WM `fastpotify`.

---

## Solution Implemented

### 1. Règles de fenêtres Hyprland dans `~/.config/caelestia/hypr-user.lua`

Assignation automatique des fenêtres aux workspaces spéciaux dès leur création :

```lua
-- ─── Special Workspaces (Scratchpads) ─────────────────────────────────────────
hl.window_rule({ match = { class = "zapfast" }, workspace = "special:communication" })
hl.window_rule({ match = { class = "fastpotify|spotifast" }, workspace = "special:music" })
```

### 2. Configuration des bascules (*toggles*) dans `~/.config/caelestia/cli.json`

Dans la section `"toggles"` de `cli.json`, Caelestia prend en charge le comportement des raccourcis de bascule (`SUPER + D` et `SUPER + M`) :
- Désactivation des entrées par défaut non installées (`discord`, `spotify`).
- Ajout de `vesktop` et `zapfast` dans `communication`.
- Ajout de `spotifast` dans `music` avec commande de lancement automatique si non démarré.

```json
    "toggles": {
        "communication": {
            "discord": {
                "enable": false
            },
            "vesktop": {
                "enable": true,
                "match": [{ "class": "vesktop" }],
                "command": ["vesktop"],
                "move": true
            },
            "zapfast": {
                "enable": true,
                "match": [{ "class": "zapfast" }],
                "move": true
            }
        },
        "music": {
            "spotify": {
                "enable": false
            },
            "spotifast": {
                "enable": true,
                "match": [{ "class": "fastpotify" }, { "class": "spotifast" }, { "initialTitle": "Spotifast" }],
                "command": ["spotifast"],
                "move": true
            }
        },
        "sysmon": {
            "btop": {
                "enable": true,
                "match": [{ "class": "btop", "title": "btop", "workspace": { "name": "special:sysmon" } }],
                "command": ["foot", "-a", "btop", "-T", "btop", "fish", "-C", "exec btop"]
            }
        },
        "todo": {
            "todoist": {
                "enable": true,
                "match": [{ "class": "Todoist" }],
                "command": ["todoist"],
                "move": true
            }
        }
    }
```

*Note sur `command` :*
- `spotifast` et `vesktop` ont un champ `command` : si l'application n'est pas encore lancée, appuyer sur le raccourci (`SUPER+M` ou `SUPER+D`) la démarre automatiquement.
- `zapfast` a `move: true` sans `command` : cela évite de forcer le lancement simultané de Discord ET WhatsApp dès qu'on appuie sur `SUPER+D`. Dès que `zapfast` est lancé, il rejoint le tiroir communication.

---

## Verification & Status

1. **Recharger la configuration Hyprland :**
   ```fish
   hyprctl reload
   ```
2. **Tester le tiroir Musique (`SUPER + M`) :**
   - Appuyer sur `SUPER + M` : `spotifast` apparaît à l'écran.
   - Réappuyer sur `SUPER + M` : `spotifast` disparaît en arrière-plan.
3. **Tester le tiroir Communication (`SUPER + D`) :**
   - Appuyer sur `SUPER + D` : `vesktop` (et `zapfast` s'il est lancé) apparaît.
   - Réappuyer sur `SUPER + D` : le tiroir se referme.
4. **Vérifier les classes des fenêtres actives :**
   ```fish
   hyprctl clients -j | jq '.[] | {class: .class, initialClass: .initialClass, workspace: .workspace.name}'
   ```

---

## Maintenance / Handy Commands

| Raccourci | Fonction |
| :--- | :--- |
| **`SUPER + D`** | Basculer le tiroir **Communication** (`special:communication` : Vesktop, ZapFast) |
| **`SUPER + M`** | Basculer le tiroir **Musique** (`special:music` : Spotifast) |
| **`CTRL + SHIFT + Échap`** | Basculer le moniteur système **btop** (`special:sysmon`) |
| **`SUPER + R`** | Basculer le gestionnaire de tâches **Todoist** (`special:todo`) |
| **`SUPER + S`** | Basculer le **scratchpad générique** (`special:special`) |
| **`SUPER + ALT + S`** | Envoyer la fenêtre active dans le scratchpad générique |
| **`CTRL + SUPER + SHIFT + Bas`** | Sortir la fenêtre active du scratchpad vers le bureau courant |
