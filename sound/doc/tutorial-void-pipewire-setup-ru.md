# PipeWire + WirePlumber без Void Linux (2025 г.)
Функциональное и простое руководство, основанное только на реальных пакетах Void Linux.

Официальная документация:
https://docs.voidlinux.org/config/media/pipewire.html

---

# 1. Установка правильных пакетов

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

Это **единственные необходимые пакеты** для PipeWire на Void.

---

# 2. Конфигурация

## 2.1 Интеграция ALSA → PipeWire
```
sudo mkdir -p /etc/alsa/conf.d
sudo ln -sf /usr/share/alsa/alsa.conf.d/50-pipewire.conf /etc/alsa/conf.d
sudo ln -sf /usr/share/alsa/alsa.conf.d/99-pipewire-default.conf /etc/alsa/conf.d
mkdir -p ~/.config/pipewire/pipewire.conf.d
mkdir -p ~/.config/pipewire/pipewire-pulse.conf.d
```

## 2.2 Включить сервер Pipewire-Pulse (совместим с PulseAudio)
```
sudo mkdir -p /etc/pipewire/pipewire.conf.d
sudo ln -sf /usr/share/examples/pipewire/20-pipewire-pulse.conf /etc/pipewire/pipewire.conf.d/
```

## 2.3 Включение автозапуска PipeWire во время сеанса
```
mkdir -p ~/.config/autostart
ln -sf /usr/share/applications/pipewire.desktop ~/.config/autostart/
ln -sf /usr/share/applications/pipewire-pulse.desktop ~/.config/autostart/
ln -sf /usr/share/applications/wireplumber.desktop ~/.config/autostart/
```
---

# 3. Автозапуск по DE/WM

## 3.1 Среды рабочего стола (работает отдельно)
Эти DE автоматически запускают PipeWire.
**Нечего делать.**

- XFCE
- LXQt
- LXDE
- KDE Плазма (X11 и Wayland)
- GNOME (X11 и Wayland)
- Корица
- СМЕРТЬ
- Просвещение

PipeWire подключается через DBus без вмешательства.

---

## 3.2 Оконные менеджеры (нужно запускать вручную)

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

### Fluxbox/IceWM/JWM — `~/.xinitrc`

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

### Sway — `~/.config/sway/config`

```
exec pipewire
exec wireplumber
exec pipewire-pulse
```

### Вэйфайр

```
exec pipewire
exec wireplumber
exec pipewire-pulse
```

### Хайпрланд

```
exec-once = pipewire
exec-once = wireplumber
exec-once = pipewire-pulse
```

---

# 4. Обязательные группы (Void без elogind)

```
sudo usermod -aG audio,video $USER
```

Выход/вход.

---

# 5. Общие тесты

ТрубаПроволока:
```
ps aux | grep pipewire
```

ПроводСантехник:
```
ps aux | grep wireplumber
```

Совместимость с PulseAudio:
```
pactl info | grep "Server Name"
```

Ожидал:
```
Server Name: PulseAudio (on PipeWire 0.3.x)
```

Шов:
```
speaker-test -t wav -c 2
```

---

# 6. Устранение неполадок

Удалите старые конфиги PulseAudio:
```
rm -rf ~/.config/pulse ~/.pulse
```

Проверьте БД:
```
ps aux | grep dbus
```

Запустите PipeWire вручную:
```
dbus-run-session -- pipewire
```

Приложения, жалующиеся на PulseAudio:
```
pipewire-pulse &
```

---

# 7. Краткое описание

Установить:
```
sudo xbps-install pipewire wireplumber alsa-pipewire alsa-utils alsa-plugins alsa-lib alsa-firmware
```

Автоматически поддерживаемые DE:
- XFCE, LXQt, LXDE, GNOME, KDE, Cinnamon, MATE, Просвещение

WM, требующие настройки:
- i3, bspwm, dwm, Openbox, Fluxbox, Sway, Wayfire, Hyprland

PipeWire работает по сеансам (не использует runit).

---

# КОНЕЦ

