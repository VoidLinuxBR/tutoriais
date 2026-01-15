# PipeWire + WirePlumber no Void Linux (2025)
Funktionelle, unkomplizierte Anleitung, die nur auf echten Void-Linux-Paketen basiert.

Offizielle Dokumentation:
https://docs.voidlinux.org/config/media/pipewire.html

---

# 1. Die richtigen Pakete installieren

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

Dies sind die **einzig erforderlichen Pakete** für PipeWire auf Void.

---

# 2. Konfiguration

## 2.1 Integrieren Sie ALSA → PipeWire
```
sudo mkdir -p /etc/alsa/conf.d
sudo ln -sf /usr/share/alsa/alsa.conf.d/50-pipewire.conf /etc/alsa/conf.d
sudo ln -sf /usr/share/alsa/alsa.conf.d/99-pipewire-default.conf /etc/alsa/conf.d
mkdir -p ~/.config/pipewire/pipewire.conf.d
mkdir -p ~/.config/pipewire/pipewire-pulse.conf.d
```

## 2.2 Pipewire-Pulse-Server aktivieren (PulseAudio-kompatibel)
```
sudo mkdir -p /etc/pipewire/pipewire.conf.d
sudo ln -sf /usr/share/examples/pipewire/20-pipewire-pulse.conf /etc/pipewire/pipewire.conf.d/
```

## 2.3 Aktivieren Sie den PipeWire-Autostart in der Sitzung
```
mkdir -p ~/.config/autostart
ln -sf /usr/share/applications/pipewire.desktop ~/.config/autostart/
ln -sf /usr/share/applications/pipewire-pulse.desktop ~/.config/autostart/
ln -sf /usr/share/applications/wireplumber.desktop ~/.config/autostart/
```
---

# 3. Autostart durch DE/WM

## 3.1 Desktop-Umgebungen (läuft alleine)
Diese DEs starten PipeWire automatisch.
**Nichts zu tun.**

- XFCE
- LXQt
- LXDE
- KDE Plasma (X11 und Wayland)
- GNOME (X11 und Wayland)
- Zimt
- TOD
- Aufklärung

PipeWire wird ohne Eingriff über DBus hochgefahren.

---

## 3.2 Fenstermanager (müssen manuell gestartet werden)

### i3 – „~/.xinitrc“.

```
export XDG_RUNTIME_DIR="/run/user/$(id -u)"

dbus-run-session -- sh -c '
  pipewire &
  wireplumber &
  pipewire-pulse &
  exec i3
'
```

### bspwm – „~/.xinitrc“.

```
export XDG_RUNTIME_DIR="/run/user/$(id -u)"

dbus-run-session -- sh -c '
  pipewire &
  wireplumber &
  pipewire-pulse &
  exec bspwm
'
```

### dwm – „~/.xinitrc“.

```
export XDG_RUNTIME_DIR="/run/user/$(id -u)"

dbus-run-session -- sh -c '
  pipewire &
  wireplumber &
  pipewire-pulse &
  exec dwm
'
```

### Openbox – „~/.xinitrc“.

```
export XDG_RUNTIME_DIR="/run/user/$(id -u)"

dbus-run-session -- sh -c '
  pipewire &
  wireplumber &
  pipewire-pulse &
  exec openbox-session
'
```

### Fluxbox / IceWM / JWM – „~/.xinitrc“.

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

### Sway – `~/.config/sway/config`

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

# 4. Erforderliche Gruppen (Void ohne elogind)

```
sudo usermod -aG audio,video $USER
```

Abmelden/Anmelden.

---

# 5. Allgemeine Tests

Rohrdraht:
```
ps aux | grep pipewire
```

Drahtklempner:
```
ps aux | grep wireplumber
```

PulseAudio-kompatibel:
```
pactl info | grep "Server Name"
```

Erwartet:
```
Server Name: PulseAudio (on PipeWire 0.3.x)
```

Naht:
```
speaker-test -t wav -c 2
```

---

# 6. Fehlerbehebung

Entfernen Sie alte PulseAudio-Konfigurationen:
```
rm -rf ~/.config/pulse ~/.pulse
```

DBus prüfen:
```
ps aux | grep dbus
```

Starten Sie PipeWire manuell:
```
dbus-run-session -- pipewire
```

Anwendungen, die sich über PulseAudio beschweren:
```
pipewire-pulse &
```

---

# 7. Kurze Zusammenfassung

Installieren:
```
sudo xbps-install pipewire wireplumber alsa-pipewire alsa-utils alsa-plugins alsa-lib alsa-firmware
```

Automatisch unterstützte DEs:
- XFCE, LXQt, LXDE, GNOME, KDE, Cinnamon, MATE, Aufklärung

WMs, die konfiguriert werden müssen:
- i3, bspwm, dwm, Openbox, Fluxbox, sway, wayfire, hyprland

PipeWire arbeitet pro Sitzung (verwendet kein Runit).

---

# ENDE

