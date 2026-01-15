# PipeWire + WirePlumber no Void Linux (2025)
实用、简单的指南，仅基于真正的 Void Linux 软件包。

官方文档：
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

这些是 PipeWire on Void **唯一需要的软件包**。

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

## 2.3 在会话中启用PipeWire自动启动
```
mkdir -p ~/.config/autostart
ln -sf /usr/share/applications/pipewire.desktop ~/.config/autostart/
ln -sf /usr/share/applications/pipewire-pulse.desktop ~/.config/autostart/
ln -sf /usr/share/applications/wireplumber.desktop ~/.config/autostart/
```
---

# 3. DE/WM自动启动

## 3.1 桌面环境（单独运行）
这些 DE 自动启动 PipeWire。
**没什么可做的。**

- XFCE
- LXQt
- LXDE
- KDE Plasma（X11 和 Wayland）
- GNOME（X11 和 Wayland）
- 肉桂
- 死亡
- 启示

PipeWire 通过 DBus 上升，无需干预。

---

## 3.2 窗口管理器（需要手动启动）

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

### 海普兰

```
exec-once = pipewire
exec-once = wireplumber
exec-once = pipewire-pulse
```

---

# 4. 所需的组（没有 elogind 则无效）

```
sudo usermod -aG audio,video $USER
```

注销/登录。

---

# 5. 一般测试

管道线：
```
ps aux | grep pipewire
```

电线管道工：
```
ps aux | grep wireplumber
```

PulseAudio 兼容：
```
pactl info | grep "Server Name"
```

预期的：
```
Server Name: PulseAudio (on PipeWire 0.3.x)
```

接缝：
```
speaker-test -t wav -c 2
```

---

# 6. 故障排除

删除旧的 PulseAudio 配置：
```
rm -rf ~/.config/pulse ~/.pulse
```

检查DBus：
```
ps aux | grep dbus
```

手动启动 PipeWire：
```
dbus-run-session -- pipewire
```

抱怨 PulseAudio 的应用程序：
```
pipewire-pulse &
```

---

# 7. 快速总结

安装：
```
sudo xbps-install pipewire wireplumber alsa-pipewire alsa-utils alsa-plugins alsa-lib alsa-firmware
```

自动支持的 DE：
- XFCE、LXQt、LXDE、GNOME、KDE、肉桂、MATE、启蒙

需要配置的WM：
- i3、bspwm、dwm、Openbox、Fluxbox、sway、wayfire、hyprland

PipeWire 按会话运行（不使用 runit）。

---

# 结尾

