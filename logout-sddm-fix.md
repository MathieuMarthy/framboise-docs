# Logout button ne relance pas SDDM

## Overview

**Problème** : Cliquer sur le bouton « logout » dans le menu session de Caelestia ne renvoyait pas à l'écran de connexion SDDM (écran noir nécessitant de basculer manuellement en TTY avec Ctrl+Alt+F3 pour intervenir).

**Composants affectés** : Caelestia Shell (Quickshell), Hyprland (v0.56+), SDDM (v0.21.0), `start-hyprland`, logind (`systemd-logind`).

---

## Context / Root Cause

### Architecture des terminaux virtuels (VT)
Sur cette machine, SDDM et Hyprland s'exécutent sur deux terminaux virtuels distincts :
- **VT 2 (`tty2`)** : Serveur Xorg lancé par SDDM pour afficher le greeter (thème Caelestia).
- **VT 1 (`tty1`)** : Session Wayland utilisateur exécutée via `/usr/bin/start-hyprland`.

### Pourquoi les tentatives précédentes ont échoué

1. **Tentative `hyprctl dispatch exit`** :
   Dans Hyprland 0.56+ avec le runtime Lua, `hyprctl dispatch exit` essaie d'évaluer le dispatcher en Lua (`return hl.dispatch(exit)`). Comme `exit` n'est pas une variable Lua définie, la commande échoue avec l'erreur `expected a dispatcher (e.g. hl.dsp.window.close())` (code retour 7). Hyprland ne recevait jamais l'ordre de quitter, et le clic sur le bouton ne faisait rien.
   *La commande exacte en Lua est `hyprctl dispatch 'hl.dsp.exit()'`.

2. **Tentative `loginctl terminate-session $XDG_SESSION_ID`** :
   Logind envoie un signal `SIGTERM` brutal à tous les processus de la session sans distinction. Le processus watchdog `/usr/bin/start-hyprland` reçoit ce signal et lève une exception interne non gérée, appelant `abort()` (`SIGABRT`, signal 6).
   En conséquence :
   - `sddm-helper` signale une erreur de crash de session : `Authentication error: SDDM::Auth::ERROR_INTERNAL "Process crashed"`.
   - SDDM considère la session comme crashée et **ne relance pas le greeter**.
   - Le noyau reste positionné sur **VT 1**, qui n'a plus aucun affichage actif → **écran noir complet**.

---

## Solution Implemented

Pour que SDDM relance proprement son greeter sans écran noir, il faut :
1. **Basculer activement l'affichage vers le VT de SDDM** (généralement VT 2) avant l'extinction via l'appel D-Bus logind `org.freedesktop.login1.Seat.SwitchTo` (accessible sans privilèges root pour l'utilisateur de la session active).
2. **Quitter Hyprland proprement** via son API IPC avec la syntaxe Lua valide `hl.dsp.exit()`.
3. **Résoudre l'instance signature** (`HYPRLAND_INSTANCE_SIGNATURE`) si Caelestia ne la propage pas au sous-processus.

### 1. Script wrapper [`~/.local/bin/hyprland-logout`](file:///home/mathieu/.local/bin/hyprland-logout)

```bash
#!/bin/bash
# hyprland-logout — Quitte proprement la session Hyprland et renvoie au greeter SDDM

# 1. Récupérer l'instance Hyprland si absente de l'environnement Caelestia
if [ -z "$HYPRLAND_INSTANCE_SIGNATURE" ]; then
    HYPRLAND_INSTANCE_SIGNATURE=$(ls -1 "/run/user/$(id -u)/hypr" 2>/dev/null | head -n1)
    export HYPRLAND_INSTANCE_SIGNATURE
fi

# 2. Détecter le VT utilisé par le serveur Xorg de SDDM (par défaut vt2)
SDDM_VT=$(pgrep -a Xorg 2>/dev/null | grep -o 'vt[0-9]*' | tr -d 'vt' | head -n1)
SDDM_VT=${SDDM_VT:-2}

# 3. Basculer l'affichage vers le VT de SDDM via logind
busctl call org.freedesktop.login1 /org/freedesktop/login1/seat/seat0 org.freedesktop.login1.Seat SwitchTo u "${SDDM_VT}" 2>/dev/null

# 4. Demander à Hyprland de quitter proprement (syntaxe Lua Hyprland 0.56+)
hyprctl dispatch 'hl.dsp.exit()'
```

Permissions :
```fish
chmod +x ~/.local/bin/hyprland-logout
```

### 2. Configuration Caelestia [`~/.config/caelestia/shell.json`](file:///home/mathieu/.config/caelestia/shell.json)

```json
{
    "session": {
        "commands": {
            "logout": [
                "/home/mathieu/.local/bin/hyprland-logout"
            ]
        }
    }
}
```

---

## Verification & Status

- Exécution de test du dispatcher sans fermeture : `hyprctl dispatch 'hl.dsp.no_op()'` renvoie `ok`.
- Test du switch de VT via `busctl call org.freedesktop.login1 /org/freedesktop/login1/seat/seat0 org.freedesktop.login1.Seat SwitchTo u 1` exécuté avec succès sans élévation de privilèges.
- Lors du logout :
  1. `SwitchTo` bascule sur VT 2 (SDDM).
  2. `hl.dsp.exit()` ferme Hyprland avec code 0.
  3. `start-hyprland` et `sddm-helper` se terminent avec succès (`SDDM::Auth::HELPER_SUCCESS`).
  4. SDDM relance le greeter Caelestia sur VT 2.

---

## Maintenance / Handy Commands

```fish
# Recharger la shell Caelestia pour prendre en compte les modifications de shell.json
caelestia shell -k; and caelestia shell -d

# Vérifier le VT actif de SDDM
pgrep -a Xorg | grep -o 'vt[0-9]*'

# Vérifier les logs SDDM en direct
journalctl -fu sddm.service
```
