# PipeWire + WirePlumber sans Void Linux (2025)
Guide fonctionnel et simple, basé uniquement sur de vrais packages Void Linux.

Documentation officielle :
https://docs.voidlinux.org/config/media/pipewire.html

---

# 1. Installer les packages appropriés

```
sudo xbps-install -S \
  pipewire \
  wireplumber \
  alsa-pipewire \
  alsa-lib \
  alsa-utils \
  alsa-plugins \
  alsa-firmware \
  pavucontrol \
  pulsemixer
```

Ce sont les **seuls packages requis** pour PipeWire on Void.

---

# 2.Configuration

## 2.1 Intégrer ALSA → PipeWire
```
sudo mkdir -p /etc/alsa/conf.d
sudo ln -sf /usr/share/alsa/alsa.conf.d/50-pipewire.conf /etc/alsa/conf.d
sudo ln -sf /usr/share/alsa/alsa.conf.d/99-pipewire-default.conf /etc/alsa/conf.d
mkdir -p ~/.config/pipewire/pipewire.conf.d
mkdir -p ~/.config/pipewire/pipewire-pulse.conf.d
```

## 2.2 Activer le serveur pipewire-pulse (compatibilité PulseAudio)
```
sudo mkdir -p /etc/pipewire/pipewire.conf.d
sudo ln -sf /usr/share/examples/pipewire/20-pipewire-pulse.conf /etc/pipewire/pipewire.conf.d/
```

## 2.3 Activer le démarrage automatique de PipeWire en session
```
mkdir -p ~/.config/autostart
ln -sf /usr/share/applications/pipewire.desktop ~/.config/autostart/
ln -sf /usr/share/applications/pipewire-pulse.desktop ~/.config/autostart/
ln -sf /usr/share/applications/wireplumber.desktop ~/.config/autostart/
```
---

# 3. Démarrage automatique par DE/WM

## 3.1 Environnements de bureau (fonctionne seul)
Ces DE démarrent automatiquement PipeWire.
**Rien à faire.**

- XFCE
- LXQt
- LXDE
- Plasma KDE (X11 et Wayland)
- GNOME (X11 et Wayland)
- Cannelle
- LA MORT
- Éclaircissement

PipeWire monte via DBus sans intervention.

---

## 3.2 Gestionnaires de fenêtres (doivent démarrer manuellement)

### i3 — `~/.xinitrc`

```
export XDG_RUNTIME_DIR="/run/user/$(id -u)"

dbus-run-session -- sh -c '
  pipewire &
  wireplumber &
  pipewire-pulse &
  exec i3
'
```

### bspwm — `~/.xinitrc`

```
export XDG_RUNTIME_DIR="/run/user/$(id -u)"

dbus-run-session -- sh -c '
  pipewire &
  wireplumber &
  pipewire-pulse &
  exec bspwm
'
```

### dwm — `~/.xinitrc`

```
export XDG_RUNTIME_DIR="/run/user/$(id -u)"

dbus-run-session -- sh -c '
  pipewire &
  wireplumber &
  pipewire-pulse &
  exec dwm
'
```

### Boîte ouverte — `~/.xinitrc`

```
export XDG_RUNTIME_DIR="/run/user/$(id -u)"

dbus-run-session -- sh -c '
  pipewire &
  wireplumber &
  pipewire-pulse &
  exec openbox-session
'
```

### Fluxbox / IceWM / JWM — `~/.xinitrc`

```
export XDG_RUNTIME_DIR="/run/user/$(id -u)"

dbus-run-session -- sh -c '
  pipewire &
  wireplumber &
  pipewire-pulse &
  exec fluxbox
'
```

---

## 3.3 WM Wayland

### Balancement — `~/.config/sway/config`

```
exec pipewire
exec wireplumber
exec pipewire-pulse
```

### Feu de chemin

```
exec pipewire
exec wireplumber
exec pipewire-pulse
```

### Hyprlande

```
exec-once = pipewire
exec-once = wireplumber
exec-once = pipewire-pulse
```

---

# 4. Groupes requis (Vide sans elogind)

```
sudo usermod -aG audio,video $USER
```

Déconnexion/connexion.

---

# 5. Tests généraux

Fil de tuyau :
```
ps aux | grep pipewire
```

FilPlombier :
```
ps aux | grep wireplumber
```

Compatibilité PulseAudio :
```
pactl info | grep "Server Name"
```

Attendu:
```
Server Name: PulseAudio (on PipeWire 0.3.x)
```

Couture:
```
speaker-test -t wav -c 2
```

---

# 6. Dépannage

Supprimez les anciennes configurations PulseAudio :
```
rm -rf ~/.config/pulse ~/.pulse
```

Vérifiez DBus :
```
ps aux | grep dbus
```

Démarrez PipeWire manuellement :
```
dbus-run-session -- pipewire
```

Applications se plaignant de PulseAudio :
```
pipewire-pulse &
```

---

# 7. Résumé rapide

Installer:
```
sudo xbps-install pipewire wireplumber alsa-pipewire alsa-utils alsa-plugins alsa-lib alsa-firmware
```

DE automatiquement pris en charge :
- XFCE, LXQt, LXDE, GNOME, KDE, Cinnamon, MATE, Lumières

WM nécessitant une configuration :
- i3, bspwm, dwm, Openbox, Fluxbox, balancement, wayfire, hyprland

PipeWire fonctionne par session (n'utilise pas runit).

---

# FIN

