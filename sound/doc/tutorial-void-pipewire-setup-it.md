# PipeWire + WirePlumber senza Void Linux (2025)
Guida funzionale e semplice, basata solo su pacchetti Void Linux reali.

Documentazione ufficiale:
https://docs.voidlinux.org/config/media/pipewire.html

---

# 1. Installazione dei pacchetti corretti

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

Questi sono gli **unici pacchetti richiesti** per PipeWire su Void.

---

# 2. Configurazione

## 2.1 Integrazione ALSA → PipeWire
```
sudo mkdir -p /etc/alsa/conf.d
sudo ln -sf /usr/share/alsa/alsa.conf.d/50-pipewire.conf /etc/alsa/conf.d
sudo ln -sf /usr/share/alsa/alsa.conf.d/99-pipewire-default.conf /etc/alsa/conf.d
mkdir -p ~/.config/pipewire/pipewire.conf.d
mkdir -p ~/.config/pipewire/pipewire-pulse.conf.d
```

## 2.2 Abilitare il server pipewire-pulse (compatibile con PulseAudio)
```
sudo mkdir -p /etc/pipewire/pipewire.conf.d
sudo ln -sf /usr/share/examples/pipewire/20-pipewire-pulse.conf /etc/pipewire/pipewire.conf.d/
```

## 2.3 Abilita l'avvio automatico di PipeWire nella sessione
```
mkdir -p ~/.config/autostart
ln -sf /usr/share/applications/pipewire.desktop ~/.config/autostart/
ln -sf /usr/share/applications/pipewire-pulse.desktop ~/.config/autostart/
ln -sf /usr/share/applications/wireplumber.desktop ~/.config/autostart/
```
---

# 3. Avvio automatico da DE/WM

## 3.1 Ambienti desktop (funziona da solo)
Questi DE avviano PipeWire automaticamente.
**Niente da fare.**

- XFCE
- LXQt
- LXDE
- KDE Plasma (X11 e Wayland)
- GNOME (X11 e Wayland)
- Cannella
- MORTE
- Illuminismo

PipeWire sale tramite DBus senza intervento.

---

## 3.2 Window Manager (è necessario avviarlo manualmente)

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

## 3.3 WM Wayland

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

# 4. Gruppi richiesti (Vuoto senza elogind)

```
sudo usermod -aG audio,video $USER
```

Esci/accedi.

---

# 5. Test generali

Filotubo:
```
ps aux | grep pipewire
```

WireIdraulico:
```
ps aux | grep wireplumber
```

Compatibilità PulseAudio:
```
pactl info | grep "Server Name"
```

Previsto:
```
Server Name: PulseAudio (on PipeWire 0.3.x)
```

Cucitura:
```
speaker-test -t wav -c 2
```

---

# 6. Risoluzione dei problemi

Rimuovi le vecchie configurazioni PulseAudio:
```
rm -rf ~/.config/pulse ~/.pulse
```

Controlla DBus:
```
ps aux | grep dbus
```

Avvia PipeWire manualmente:
```
dbus-run-session -- pipewire
```

Applicazioni che lamentano PulseAudio:
```
pipewire-pulse &
```

---

# 7. Breve riepilogo

Installare:
```
sudo xbps-install pipewire wireplumber alsa-pipewire alsa-utils alsa-plugins alsa-lib alsa-firmware
```

DE supportati automaticamente:
- XFCE, LXQt, LXDE, GNOME, KDE, Cannella, MATE, Illuminismo

WM che necessitano di configurazione:
- i3, bspwm, dwm, Openbox, Fluxbox, ondeggiamento, wayfire, hyprland

PipeWire funziona per sessione (non utilizza runit).

---

# FINE

