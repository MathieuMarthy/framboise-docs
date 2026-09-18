# Logout button ne relance pas SDDM

## Overview

**Problème** : Cliquer sur le bouton « logout » dans le menu session de Caelestia ne renvoyait pas à l'écran SDDM. Deux symptômes successifs observés :
1. La session se fermait mais SDDM n'apparaissait pas (écran noir) — comportement original via logind D-Bus.
2. Rien ne se passait du tout — après un premier essai avec `hyprctl dispatch exit`.

**Composants affectés** : Caelestia Shell (Quickshell), Hyprland, SDDM, logind, VT (Virtual Terminal).

---

## Context / Root Cause

### Architecture VT dans cette configuration

| VT  | Contenu                                   |
| :-- | :---------------------------------------- |
| VT1 | Session Wayland Hyprland (utilisateur)    |
| VT2 | Xorg + greeter SDDM (display manager)     |

### Problème 1 — Écran noir (comportement original)

Le bouton logout appelle `SessionManager.exec(command)` depuis Caelestia ([`Content.qml`](file:///etc/xdg/quickshell/caelestia/modules/session/Content.qml)). Par défaut, `SessionManager` passe par logind via D-Bus (`org.freedesktop.login1.Session`) pour terminer la session PAM.

Cela fonctionne pour terminer Hyprland, **mais** logind ne switche pas le VT automatiquement dans cette configuration. Résultat : VT1 devient noir, SDDM reste actif sur VT2 sans reprendre le focus.

### Problème 2 — Rien ne se passe

Tentative de surcharge avec `["hyprctl", "dispatch", "exit"]` dans `shell.json` :
- `SessionManager.exec()` ne reconnaît pas cette commande → retourne `false`.
- Caelestia fait un fallback via `Quickshell.execDetached()`.
- Le subprocess **n'hérite pas** de `HYPRLAND_INSTANCE_SIGNATURE` → `hyprctl` ne trouve pas le socket Hyprland → silently fails.

### Solution retenue

Utiliser `loginctl terminate-session $XDG_SESSION_ID` via un script wrapper. Avantages :
- `XDG_SESSION_ID` est hérité correctement par `Quickshell.execDetached()`.
- `loginctl terminate-session` passe par logind qui **gère le switch VT** et notifie SDDM pour relancer son greeter.

---

## Solution Implemented

### 1. Script wrapper [`~/.local/bin/hyprland-logout`](file:///home/mathieu/.local/bin/hyprland-logout)

```bash
#!/bin/bash
# Quitte proprement la session Hyprland et renvoie à SDDM

exec loginctl terminate-session "${XDG_SESSION_ID}"
```

```bash
chmod +x ~/.local/bin/hyprland-logout
```

### 2. Override dans [`~/.config/caelestia/shell.json`](file:///home/mathieu/.config/caelestia/shell.json)

```json
{
    "session": {
        "commands": {
            "logout": ["/home/mathieu/.local/bin/hyprland-logout"]
        }
    },
    ...
}
```

---

## Verification & Status

Après redémarrage de la shell Caelestia ou à la prochaine session :

1. Cliquer sur le bouton logout → `hyprland-logout` est exécuté.
2. `loginctl terminate-session $XDG_SESSION_ID` envoie SIGTERM au leader de session (Hyprland).
3. Hyprland se ferme, logind switche le VT vers VT2.
4. SDDM détecte la fin de session et affiche son greeter.

Vérification manuelle depuis un terminal dans la session :

```bash
loginctl terminate-session "$XDG_SESSION_ID"
```

---

## Maintenance / Handy Commands

```bash
# Voir les sessions logind actives et l'ID courant
loginctl list-sessions
echo $XDG_SESSION_ID

# Vérifier la config shell
python3 -m json.tool ~/.config/caelestia/shell.json

# Relancer Caelestia shell sans se déconnecter (pour recharger shell.json)
caelestia shell -k && caelestia shell -d

# Voir le statut SDDM
systemctl status sddm.service

# Voir les logs SDDM en temps réel
journalctl -fu sddm.service
```
