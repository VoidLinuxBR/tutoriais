# PipeWire + WirePlumber no Void Linux (2025)
Functional, straightforward guide, based only on real Void Linux packages.

Official documentation:
https://docs.voidlinux.org/config/media/pipewire.html

---

# 1. Installing the correct packages

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

These are the **only required packages** for PipeWire on Void.

---

# 2. Configuration

## 2.1 Integrate ALSA → PipeWire
```
sudo mkdir -p /etc/alsa/conf.d
sudo ln -sf /usr/share/alsa/alsa.conf.d/50-pipewire.conf /etc/alsa/conf.d
sudo ln -sf /usr/share/alsa/alsa.conf.d/99-pipewire-default.conf /etc/alsa/conf.d
mkdir -p ~/.config/pipewire/pipewire.conf.d
mkdir -p ~/.config/pipewire/pipewire-pulse.conf.d
```

## 2.2 Enable pipewire-pulse server (PulseAudio compat)
```
sudo mkdir -p /etc/pipewire/pipewire.conf.d
sudo ln -sf /usr/share/examples/pipewire/20-pipewire-pulse.conf /etc/pipewire/pipewire.conf.d/
```

## 2.3 Enable PipeWire autostart in session
```
mkdir -p ~/.config/autostart
ln -sf /usr/share/applications/pipewire.desktop ~/.config/autostart/
ln -sf /usr/share/applications/pipewire-pulse.desktop ~/.config/autostart/
ln -sf /usr/share/applications/wireplumber.desktop ~/.config/autostart/
```
---

# 3. Autostart by DE/WM

## 3.1 Desktop Environments (runs alone)
These DEs start PipeWire automatically.
**Nothing to do.**

- XFCE
- LXQt
- LXDE
- KDE Plasma (X11 and Wayland)
- GNOME (X11 and Wayland)
- Cinnamon
- DEATH
- Enlightenment

PipeWire goes up via DBus without intervention.

---

## 3.2 Window Managers (need to start manually)

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

### Openbox — `~/.xinitrc`

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

## 3.3 Wayland WMs

### Sway — `~/.config/sway/config`

```
exec pipewire
exec wireplumber
exec pipewire-pulse
```

### Wayfire

```
exec pipewire
exec wireplumber
exec pipewire-pulse
```

### Hyprland

```
exec-once = pipewire
exec-once = wireplumber
exec-once = pipewire-pulse
```

---

# 4. Required groups (Void without elogind)

```
sudo usermod -aG audio,video $USER
```

Logout/login.

---

# 5. General Tests

PipeWire:
```
ps aux | grep pipewire
```

WirePlumber:
```
ps aux | grep wireplumber
```

PulseAudio compat:
```
pactl info | grep "Server Name"
```

Expected:
```
Server Name: PulseAudio (on PipeWire 0.3.x)
```

Seam:
```
speaker-test -t wav -c 2
```

---

# 6. Troubleshooting

Remove old PulseAudio configs:
```
rm -rf ~/.config/pulse ~/.pulse
```

Check DBus:
```
ps aux | grep dbus
```

Start PipeWire manually:
```
dbus-run-session -- pipewire
```

Applications complaining about PulseAudio:
```
pipewire-pulse &
```

---

# 7. Quick Summary

Install:
```
sudo xbps-install pipewire wireplumber alsa-pipewire alsa-utils alsa-plugins alsa-lib alsa-firmware
```

Automatically supported DEs:
- XFCE, LXQt, LXDE, GNOME, KDE, Cinnamon, MATE, Enlightenment

WMs that need configuration:
- i3, bspwm, dwm, Openbox, Fluxbox, sway, wayfire, hyprland

PipeWire operates per session (does not use runit).

---

# END

