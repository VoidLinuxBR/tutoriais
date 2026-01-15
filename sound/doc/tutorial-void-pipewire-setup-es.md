# PipeWire + WirePlumber sin Void Linux (2025)
Guía funcional y sencilla, basada únicamente en paquetes reales de Void Linux.

Documentación oficial:
CHILE_REF_0_CHILI

---

# 1. Instalar los paquetes correctos

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

Estos son los **únicos paquetes requeridos** para PipeWire on Void.

---

# 2. Configuración

## 2.1 Integrar ALSA → PipeWire
```
sudo mkdir -p /etc/alsa/conf.d
sudo ln -sf /usr/share/alsa/alsa.conf.d/50-pipewire.conf /etc/alsa/conf.d
sudo ln -sf /usr/share/alsa/alsa.conf.d/99-pipewire-default.conf /etc/alsa/conf.d
mkdir -p ~/.config/pipewire/pipewire.conf.d
mkdir -p ~/.config/pipewire/pipewire-pulse.conf.d
```

## 2.2 Habilitar el servidor pipewire-pulse (compatible con PulseAudio)
```
sudo mkdir -p /etc/pipewire/pipewire.conf.d
sudo ln -sf /usr/share/examples/pipewire/20-pipewire-pulse.conf /etc/pipewire/pipewire.conf.d/
```

## 2.3 Habilitar el inicio automático de PipeWire en la sesión
```
mkdir -p ~/.config/autostart
ln -sf /usr/share/applications/pipewire.desktop ~/.config/autostart/
ln -sf /usr/share/applications/pipewire-pulse.desktop ~/.config/autostart/
ln -sf /usr/share/applications/wireplumber.desktop ~/.config/autostart/
```
---

# 3. Inicio automático por DE/WM

## 3.1 Entornos de escritorio (se ejecuta solo)
Estos DE inician PipeWire automáticamente.
**Nada que hacer.**

- XFCE
- LXQt
- LXDE
- KDE Plasma (X11 y Wayland)
- GNOME (X11 y Wayland)
- Canela
- MUERTE
- Ilustración

PipeWire se conecta mediante DBus sin intervención.

---

## 3.2 Administradores de ventanas (es necesario iniciar manualmente)

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

### Caja abierta — `~/.xinitrc`

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

## 3.3 Wayland WM

### Influencia — `~/.config/sway/config`

```
exec pipewire
exec wireplumber
exec pipewire-pulse
```

### fuego

```
exec pipewire
exec wireplumber
exec pipewire-pulse
```

### Hiprlandia

```
exec-once = pipewire
exec-once = wireplumber
exec-once = pipewire-pulse
```

---

# 4. Grupos requeridos (Nulo sin elogindo)

```
sudo usermod -aG audio,video $USER
```

Cerrar sesión/iniciar sesión.

---

# 5. Pruebas generales

Cable de tubería:
```
ps aux | grep pipewire
```

Alambreplomero:
```
ps aux | grep wireplumber
```

Compatibilidad con PulseAudio:
```
pactl info | grep "Server Name"
```

Esperado:
```
Server Name: PulseAudio (on PipeWire 0.3.x)
```

Costura:
```
speaker-test -t wav -c 2
```

---

# 6. Solución de problemas

Elimine las configuraciones antiguas de PulseAudio:
```
rm -rf ~/.config/pulse ~/.pulse
```

Comprobar DBus:
```
ps aux | grep dbus
```

Inicie PipeWire manualmente:
```
dbus-run-session -- pipewire
```

Aplicaciones que se quejan de PulseAudio:
```
pipewire-pulse &
```

---

# 7. Resumen rápido

Instalar:
```
sudo xbps-install pipewire wireplumber alsa-pipewire alsa-utils alsa-plugins alsa-lib alsa-firmware
```

DE admitidos automáticamente:
- XFCE, LXQt, LXDE, GNOME, KDE, Cinnamon, MATE, Enlightenment

WM que necesitan configuración:
- i3, bspwm, dwm, Openbox, Fluxbox, sway, wayfire, hyprland

PipeWire opera por sesión (no usa runit).

---

# FIN

