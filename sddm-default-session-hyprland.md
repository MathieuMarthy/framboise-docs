# Sélection par défaut de la session Hyprland dans SDDM (Thème Caelestia)

## Overview

**Problème** : À chaque démarrage ou écran de connexion SDDM, la session GNOME était systématiquement sélectionnée par défaut au lieu de Hyprland.
**Target Hardware / OS** : `framboise` (CachyOS Linux, Intel Core Ultra Lunar Lake).
**Composants affectés** : SDDM (`sddm`), Thème SDDM Caelestia ([`caelestia-sddm`](https://github.com/ItsABigIgloo/caelestia-sddm)), fichiers de sessions Wayland (`/usr/share/wayland-sessions/`).

---

## Context / Root Cause

Dans le thème SDDM Caelestia installé (`/usr/share/sddm/themes/caelestia/`), deux facteurs causaient ce comportement :

1. **Index hardcodé à `0` dans le thème** :
   Dans [`/usr/share/sddm/themes/caelestia/Main.qml`](file:///usr/share/sddm/themes/caelestia/Main.qml) (ligne 478), le composant `SessionPicker` initialisait l'index de sélection avec une valeur figée :
   ```qml
   SessionPicker {
       id: sessionPickerBtn
       ...
       selectedIndex: 0
   ```
2. **Ordre alphabétique des sessions** :
   SDDM peuple le modèle `sessionModel` à partir de `/usr/share/wayland-sessions/`. Dans l'ordre alphabétique :
   - `0`: `gnome.desktop` (GNOME)
   - `1`: `hyprland.desktop` (Hyprland)
   - `2`: `hyprland-uwsm.desktop` (Hyprland UWSM)
   
   En conséquence, `selectedIndex: 0` forçait systématiquement la sélection de GNOME, ignorant la dernière session utilisée par l'utilisateur ainsi que la configuration système.
   
   *Note* : Contrairement aux thèmes SDDM standards (ex: `elarun`, `maldives`) qui utilisent `sessionModel.lastIndex`, le thème Caelestia avait cette valeur hardcodée à 0.

---

## Solution Implemented

### Patch de `Main.qml` pour utiliser `sessionModel.lastIndex`

Remplacement de l'assignation fixe par la propriété standard SDDM `sessionModel.lastIndex` :

```bash
sudo sed -i 's/selectedIndex: 0/selectedIndex: sessionModel.lastIndex/' /usr/share/sddm/themes/caelestia/Main.qml
```

#### Extrait modifié dans [`/usr/share/sddm/themes/caelestia/Main.qml`](file:///usr/share/sddm/themes/caelestia/Main.qml) :
```qml
        SessionPicker {
            id: sessionPickerBtn

            anchors.top: parent.top
            anchors.horizontalCenter: parent.horizontalCenter
            anchors.topMargin: mainCard.height - 100
            currentText: sessionPickerBtn.items.length > 0 ? sessionPickerBtn.items[0] : ""
            selectedIndex: sessionModel.lastIndex
            opacity: root.firstInput ? 0 : root.mainCardComponentsOpacity
            visible: root.sessionPickerEnabled
            onSelectedIndexChanged: {
                root.sessionIndex = sessionPickerBtn.selectedIndex;
            }
```

Grâce à `sessionModel.lastIndex`, SDDM restaure automatiquement la dernière session ouverte enregistrée dans `/var/lib/sddm/state.conf`.

---

## Verification & Status

1. **Vérification du code QML** :
   ```fish
   grep "selectedIndex: sessionModel.lastIndex" /usr/share/sddm/themes/caelestia/Main.qml
   ```
   *Doit retourner la ligne modifiée.*

2. **Validation à la connexion** :
   - Se connecter une fois sous Hyprland.
   - SDDM enregistre cette session dans `/var/lib/sddm/state.conf`.
   - Aux redémarrages et déconnexions suivants, Hyprland reste sélectionné par défaut automatiquement.

---

## Maintenance / Handy Commands

### Forcer explicitement Hyprland au premier boot (optionnel)
Si l'état SDDM est réinitialisé ou pour un nouvel utilisateur, il est possible de forcer Hyprland via la configuration globale SDDM :
```fish
echo -e "[General]\nSession=hyprland" | sudo tee /etc/sddm.conf.d/default-session.conf
```

### Vérifier la dernière session mémorisée par SDDM
```fish
sudo cat /var/lib/sddm/state.conf
```

### Tester l'affichage SDDM sans redémarrer
```fish
sddm-greeter-qt6 --test-mode --theme /usr/share/sddm/themes/caelestia
```
