# PipeWire + WirePlumber no Void Linux (2025)
實用、簡單的指南，僅基於真正的 Void Linux 軟件包。

官方文檔：
辣椒_REF_0_辣椒

---

# 

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

這些是 PipeWire on Void **唯一需要的軟件包**。

---

# 2. 配置

## 2.1 集成ALSA→PipeWire
```
sudo mkdir -p /etc/alsa/conf.d
sudo ln -sf /usr/share/alsa/alsa.conf.d/50-pipewire.conf /etc/alsa/conf.d
sudo ln -sf /usr/share/alsa/alsa.conf.d/99-pipewire-default.conf /etc/alsa/conf.d
mkdir -p ~/.config/pipewire/pipewire.conf.d
mkdir -p ~/.config/pipewire/pipewire-pulse.conf.d
```

## 
```
sudo mkdir -p /etc/pipewire/pipewire.conf.d
sudo ln -sf /usr/share/examples/pipewire/20-pipewire-pulse.conf /etc/pipewire/pipewire.conf.d/
```

## 2.3 在會話中啟用PipeWire自動啟動
```
mkdir -p ~/.config/autostart
ln -sf /usr/share/applications/pipewire.desktop ~/.config/autostart/
ln -sf /usr/share/applications/pipewire-pulse.desktop ~/.config/autostart/
ln -sf /usr/share/applications/wireplumber.desktop ~/.config/autostart/
```
---

# 3. DE/WM自動啟動

## 3.1 桌面環境（單獨運行）
這些 DE 自動啟動 PipeWire。
**沒什麼可做的。 **

- XFCE
- LXQt
- LXDE
- KDE Plasma（X11 和 Wayland）
- GNOME（X11 和 Wayland）
- 肉桂
- 死亡
- 啟示

PipeWire 通過 DBus 上升，無需干預。

---

## 3.2 窗口管理器（需要手動啟動）

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

## 3.3 Wayland WM

### Sway — `~/.config/sway/config`

```
exec pipewire
exec wireplumber
exec pipewire-pulse
```

### 路火

```
exec pipewire
exec wireplumber
exec pipewire-pulse
```

### 海普蘭

```
exec-once = pipewire
exec-once = wireplumber
exec-once = pipewire-pulse
```

---

# 4. 所需的組（沒有 elogind 則無效）

```
sudo usermod -aG audio,video $USER
```

註銷/登錄。

---

# 5. 一般測試

管道線：
```
ps aux | grep pipewire
```

電線管道工：
```
ps aux | grep wireplumber
```

PulseAudio 兼容：
```
pactl info | grep "Server Name"
```

預期的：
```
Server Name: PulseAudio (on PipeWire 0.3.x)
```

接縫：
```
speaker-test -t wav -c 2
```

---

# 6. 故障排除

刪除舊的 PulseAudio 配置：
```
rm -rf ~/.config/pulse ~/.pulse
```

檢查DBus：
```
ps aux | grep dbus
```

手動啟動 PipeWire：
```
dbus-run-session -- pipewire
```

抱怨 PulseAudio 的應用程序：
```
pipewire-pulse &
```

---

# 7. 快速總結

安裝：
```
sudo xbps-install pipewire wireplumber alsa-pipewire alsa-utils alsa-plugins alsa-lib alsa-firmware
```

自動支持的 DE：
- XFCE、LXQt、LXDE、GNOME、KDE、肉桂、MATE、啟蒙

需要配置的WM：
- i3、bspwm、dwm、Openbox、Fluxbox、sway、wayfire、hyprland

PipeWire 按會話運行（不使用 runit）。

---

# 結尾

