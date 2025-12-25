# PipeWire + WirePlumber no Void Linux (2025)
Guia funcional, direto, baseado somente em pacotes reais do Void Linux.

Documentação oficial:  
https://docs.voidlinux.org/config/media/pipewire.html

---

# 1. Instalação dos pacotes corretos

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

Esses são os **únicos pacotes necessários** para PipeWire no Void.

---

# 2. Configuracao

## 2.1 Integrar ALSA → PipeWire
```
sudo mkdir -p /etc/alsa/conf.d
sudo ln -sf /usr/share/alsa/alsa.conf.d/50-pipewire.conf /etc/alsa/conf.d
sudo ln -sf /usr/share/alsa/alsa.conf.d/99-pipewire-default.conf /etc/alsa/conf.d
mkdir -p ~/.config/pipewire/pipewire.conf.d
mkdir -p ~/.config/pipewire/pipewire-pulse.conf.d
```

## 2.2 Habilitar servidor pipewire-pulse (compat PulseAudio)
```
sudo mkdir -p /etc/pipewire/pipewire.conf.d
sudo ln -sf /usr/share/examples/pipewire/20-pipewire-pulse.conf /etc/pipewire/pipewire.conf.d/
```

## 2.3 Ativar autostart PipeWire na sessão
```
mkdir -p ~/.config/autostart
ln -sf /usr/share/applications/pipewire.desktop ~/.config/autostart/
ln -sf /usr/share/applications/pipewire-pulse.desktop ~/.config/autostart/
ln -sf /usr/share/applications/wireplumber.desktop ~/.config/autostart/
```
---

# 3. Inicialização automática por DE/WM

## 3.1 Desktop Environments (funciona sozinho)
Esses DEs iniciam PipeWire automaticamente.  
**Nada a fazer.**

- XFCE  
- LXQt  
- LXDE  
- KDE Plasma (X11 e Wayland)  
- GNOME (X11 e Wayland)  
- Cinnamon  
- MATE  
- Enlightenment  

PipeWire sobe via DBus sem intervenção.

---

## 3.2 Window Managers (precisa iniciar manualmente)

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

# 4. Grupos necessários (Void sem elogind)

```
sudo usermod -aG audio,video $USER
```

Logout/login.

---

# 5. Testes gerais

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

Esperado:
```
Server Name: PulseAudio (on PipeWire 0.3.x)
```

Som:
```
speaker-test -t wav -c 2
```

---

# 6. Troubleshooting

Remover configs antigas do PulseAudio:
```
rm -rf ~/.config/pulse ~/.pulse
```

Verificar DBus:
```
ps aux | grep dbus
```

Iniciar PipeWire manualmente:
```
dbus-run-session -- pipewire
```

Aplicações reclamando de PulseAudio:
```
pipewire-pulse &
```

---

# 7. Resumo rápido

Instalar:
```
sudo xbps-install pipewire wireplumber alsa-pipewire alsa-utils alsa-plugins alsa-lib alsa-firmware
```

DEs suportados automaticamente:
- XFCE, LXQt, LXDE, GNOME, KDE, Cinnamon, MATE, Enlightenment

WMs que precisam de configuração:
- i3, bspwm, dwm, Openbox, Fluxbox, sway, wayfire, hyprland

PipeWire opera por sessão (não usa runit).

---

# FIM

