# 🐧 Void Linux + GNOME — Tutorial

> ⚠️ **IMPORTANT — READ BEFORE STARTING**
>
> This tutorial **should NOT be run as `root`**, except when **explicitly indicated**.
>
> All commands were designed to be executed by **a common user**, using `sudo` when necessary.
>
> Run the entire tutorial logged in as `root`:
> - breaks permissions logic
> - invalidates steps like `sudo` configuration
> - may generate silent errors or unexpected behavior
>
> 👉 **Recommendation**
> If you have just installed the system and are logged in as `root`:
>
> 1. Create a common user
> 2. Login with this user
> 3. Follow the tutorial normally
>
> Classic rule for Unix/Linux systems:
>
> **`root` is an exception. Common user is the rule.**

---

## 0. Configure sudo - wheel group - to avoid asking for root password
```
sudo tee -a /etc/sudoers.d/g_wheel >/dev/null << EOF
%wheel ALL=(ALL:ALL) NOPASSWD: ALL
EOF

#Required permissions
sudo chmod 440 /etc/sudoers.d/g_wheel
```

## 1. Update the system
```
sudo xbps-install -Syu
```

## 2. Install full GNOME (meta-package)
```
sudo xbps-install -y gnome \
gnome-icon-theme \
papers \
network-manager-applet \
extension-manager \
nautilus \
nautilus-papers-extension \
nautilus-gnome-console-extension \
nautilus-gnome-terminal-extension \
gnome-terminal \
arc-theme \
firefox \
firefox-i18n-pt-BR \
xarchiver \
gnome-disk-utility \
gparted \
gvfs\
p7zip \
unzip \
eog \
noto-fonts-emoji \
htop
```

## 3. Install GDM (official display manager)
```
sudo xbps-install -y gdm
```

## 4. Display Drivers

### Intel
```
sudo xbps-install -y mesa-dri linux-firmware-intel
```

### new AMD (amdgpu)
```
sudo xbps-install -y mesa-dri xf86-video-amdgpu
```

### old AMD
```
sudo xbps-install -y mesa-dri xf86-video-ati
```

### Nvidia (open driver)
```
sudo xbps-install -y mesa-nouveau-dri
```

## 5. Install PipeWire (Modern Void Sound)
```
sudo xbps-install -y \
pipewire \
wireplumber \
alsa-plugins-pulseaudio \
alsa-pipewire \
libjack-pipewire \
pulseaudio-utils \
alsa-utils \
pavucontrol
```

## 6. Integrate ALSA → PipeWire
```
sudo mkdir -p /etc/alsa/conf.d
sudo ln -sf /usr/share/alsa/alsa.conf.d/50-pipewire.conf /etc/alsa/conf.d
sudo ln -sf /usr/share/alsa/alsa.conf.d/99-pipewire-default.conf /etc/alsa/conf.d
```

## 7. Enable pipewire-pulse server (PulseAudio compat)
```
sudo mkdir -p /etc/pipewire/pipewire.conf.d
sudo ln -sf /usr/share/examples/pipewire/20-pipewire-pulse.conf /etc/pipewire/pipewire.conf.d/
```

## 8. Enable PipeWire autostart in session
```
mkdir -p ~/.config/autostart
ln -sf /usr/share/applications/pipewire.desktop ~/.config/autostart/
ln -sf /usr/share/applications/pipewire-pulse.desktop ~/.config/autostart/
ln -sf /usr/share/applications/wireplumber.desktop ~/.config/autostart/
```

## 9. (Optional) Create .xinitrc for startx
```
cat <<EOF > ~/.xinitrc
#!/bin/sh
setxkbmap -layout br -variant abnt2 &
exec gnome-session
EOF
```

## 10. configure timezone - defines the time zone
```
sudo ln -sfv /usr/share/zoneinfo/America/Sao_Paulo /etc/localtime
```

## 11. configure locales
```
Sudo -i -i -e 's/s/#\(en_.us.utf-8 utf-8\)/' -e 's/^#\pt_br.br.br.
```

## 12. Customize /etc/rc.conf. Sets the console's default time zone, keyboard layout, and font. Change as needed.
```
sudo tee -a /etc/rc.conf >/dev/null << EOF
TIMEZONE="America/Sao_Paulo"
KEYMAP="br-abnt2"
FONT=Lat2-Terminus16
EOF
```

## 13. Customize /etc/locale.conf. Sets the language. Change as needed.
```
sudo tee /etc/locale.conf >/dev/null << EOF
LANG=pt_BR.UTF-8
LANGUAGE=pt_BR.UTF-8
LC_COLLATE=pt_BR.UTF-8
EOF
```

## 14. Reconfigure
```
sudo xbps-reconfigure -fa
```

## 15. Activate mandatory services (runit)
```
sudo ln -s /etc/sv/dbus /var/service/
sudo ln -s /etc/sv/elogind /var/service/
sudo ln -s /etc/sv/polkitd /var/service/
sudo ln -s /etc/sv/NetworkManager /var/service/
sudo ln -s /etc/sv/gdm /var/service/
```

## Finalization
- Using GDM → the system starts directly in GNOME.
- Without GDM → use `startx` (if `.xinitrc` exists).
