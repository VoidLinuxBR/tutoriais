# 🐧 Void Linux + GNOME — 教程


>
> 本教程**不应以“root”身份运行**，除非**明确指出**。
>
> 所有命令均设计为由 **普通用户** 执行，必要时使用“sudo”。
>
> 以“root”身份登录运行整个教程：
> - 破坏权限逻辑
> - 使“sudo”配置等步骤无效
> - pode gerar erros silenciosos ou comportamentos inesperados
>
> 👉 **推荐**
> 如果您刚刚安装系统并以“root”身份登录：
>
> 1.创建普通用户
> 2. 使用该用户登录
> 3.正常按照教程进行操作
>
> Unix/Linux 系统的经典规则：
>
> **`root` 是一个例外。普通用户是规则。**

---

## 0. 配置 sudo - 轮组 - 以避免询问 root 密码
```
sudo tee -a /etc/sudoers.d/g_wheel >/dev/null << EOF
%wheel ALL=(全部:全部) NOPASSWD: 全部
EOF

#所需权限
须藤 chmod 440 /etc/sudoers.d/g_wheel
```

## 1.更新系统
```
sudo xbps-install -Syu
```

## 2. 安装完整的 GNOME（元包）
```
sudo xbps-install -y gnome \
侏儒图标主题 \
论文\
网络管理器小程序 \
扩展管理器 \
鹦鹉螺\
鹦鹉螺论文扩展 \

nautilus-gnome-terminal-扩展 \
gnome 终端 \
弧形主题 \
火狐浏览器\
firefox-i18n-pt-BR \
xarchiver \
gnome 磁盘实用程序 \
gparted \
gvfs\
p7zip \
解压\
脑电图\
noto-字体-表情符号 \
顶部
```

## 3.安装GDM（官方显示管理器）
```
sudo xbps-install -y gdm
```

## 4. 显示驱动程序

### 英特尔
```
sudo xbps-install -y mesa-dri linux-firmware-intel
```

### 新 AMD (amdgpu)
```
sudo xbps-install -y mesa-dri xf86-video-amdgpu
```

###老AMD
```
sudo xbps-install -y mesa-dri xf86-video-ati
```

### Nvidia（开放驱动程序）
```
sudo xbps-install -y mesa-nouveau-dri
```

## 5.安装PipeWire（现代虚空声音）
```
sudo xbps-install -y \
管材\
电线工\
alsa-插件-pulseaudio \
alsa-pipewire \
libjack-pipewire \
脉冲音频实用程序\
alsa-utils \
帕武控制
```

## 6. 集成 ALSA → PipeWire
```
sudo mkdir -p /etc/alsa/conf.d
sudo ln -sf /usr/share/alsa/alsa.conf.d/50-pipewire.conf /etc/alsa/conf.d
sudo ln -sf /usr/share/alsa/alsa.conf.d/99-pipewire-default.conf /etc/alsa/conf.d
```

## 7. 启用 pipeline-pulse 服务器（PulseAudio 兼容）
```
sudo mkdir -p /etc/pipewire/pipewire.conf.d
sudo ln -sf /usr/share/examples/pipewire/20-pipewire-pulse.conf /etc/pipewire/pipewire.conf.d/
```

## 8. 在会话中启用 PipeWire 自动启动
```
mkdir -p ~/.config/自动启动
ln -sf /usr/share/applications/pipewire.desktop ~/.config/autostart/
ln -sf /usr/share/applications/pipewire-pulse.desktop ~/.config/autostart/
ln -sf /usr/share/applications/wireplumber.desktop ~/.config/autostart/
```

## 9.（可选）为startx创建.xinitrc
```
猫 <<EOF > ~/.xinitrc
#!/bin/sh
setxkbmap -layout br -variant abnt2 &
执行 gnome 会话
EOF
```

## 10.configure timezone - 定义时区
```
sudo ln -sfv /usr/share/zoneinfo/America/Sao_Paulo /etc/localtime
```

## 11. 配置区域设置
```
须藤 -i -i -e 's/s/#\(en_.us.utf-8 utf-8\)/' -e 's/^#\pt_br.br.br.
```

## 12. Customize /etc/rc.conf.设置控制台的默认时区、键盘布局和字体。根据需要进行更改。
```
sudo tee -a /etc/rc.conf >/dev/null << EOF
TIMEZONE="美国/圣保罗"
KEYMAP="br-abnt2"
FONT=Lat2-Terminus16
EOF
```

## 13. 自定义/etc/locale.conf。设置语言。根据需要进行更改。
```
sudo tee /etc/locale.conf >/dev/null << EOF
LANG=pt_BR.UTF-8
语言=pt_BR.UTF-8
LC_COLLATE=pt_BR.UTF-8
EOF
```

## 14. 重新配置
```
sudo xbps-重新配置-fa
```

## 15.激活强制服务（runit）
```
sudo ln -s /etc/sv/dbus /var/service/
sudo ln -s /etc/sv/elogind /var/service/
sudo ln -s /etc/sv/polkitd /var/service/
sudo ln -s /etc/sv/NetworkManager /var/service/
sudo ln -s /etc/sv/gdm /var/service/
```

## 最终确定
- 使用GDM → 系统直接在GNOME 中启动。
- 没有 GDM → 使用 `startx` （如果 `.xinitrc` 存在）。
