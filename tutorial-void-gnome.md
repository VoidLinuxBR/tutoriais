# 🐧 Void Linux + GNOME — Tutorial Definitivo

## 0. Configurar sudo - grupo wheel - para evitar de ficar pedindo senha de root
```
sudo tee -a /etc/sudoers.d/g_wheel >/dev/null << EOF
%wheel ALL=(ALL:ALL) NOPASSWD: ALL
EOF

#Permissões obrigatórias
sudo chmod 440 /etc/sudoers.d/g_wheel
```

## 1. Atualizar o sistema
```
sudo xbps-install -Syu
```

## 2. Instalar o GNOME completo (meta-pacote)
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
   gvfs \
   p7zip \
   unzip \
   eog \
   htop
```

## 3. Instalar o GDM (display manager oficial)
```
sudo xbps-install -y gdm
```

## 4. Drivers de vídeo

### Intel
```
sudo xbps-install -y mesa-dri linux-firmware-intel
```

### AMD nova (amdgpu)
```
sudo xbps-install -y mesa-dri xf86-video-amdgpu
```

### AMD antiga
```
sudo xbps-install -y mesa-dri xf86-video-ati
```

### Nvidia (driver aberto)
```
sudo xbps-install -y mesa-nouveau-dri
```

## 5. Instalar PipeWire (som moderno do Void)
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

## 6. Integrar ALSA → PipeWire
```
sudo mkdir -p /etc/alsa/conf.d
sudo ln -sf /usr/share/alsa/alsa.conf.d/50-pipewire.conf /etc/alsa/conf.d
sudo ln -sf /usr/share/alsa/alsa.conf.d/99-pipewire-default.conf /etc/alsa/conf.d
```

## 7. Habilitar servidor pipewire-pulse (compat PulseAudio)
```
sudo mkdir -p /etc/pipewire/pipewire.conf.d
sudo ln -sf /usr/share/examples/pipewire/20-pipewire-pulse.conf /etc/pipewire/pipewire.conf.d/
```

## 8. Ativar autostart PipeWire na sessão
```
mkdir -p ~/.config/autostart
ln -sf /usr/share/applications/pipewire.desktop ~/.config/autostart/
ln -sf /usr/share/applications/pipewire-pulse.desktop ~/.config/autostart/
ln -sf /usr/share/applications/wireplumber.desktop ~/.config/autostart/
```

## 9. (Opcional) Criar .xinitrc para startx
```
cat <<EOF > ~/.xinitrc
#!/bin/sh
setxkbmap -layout br -variant abnt2 &
exec gnome-session
EOF
```

## 10. configurar timezone - define o fuso horário
```
sudo ln -sfv /usr/share/zoneinfo/America/Sao_Paulo /etc/localtime
```

## 11. configure locales
```
sudo sed -i -e 's/^#\(en_US.UTF-8 UTF-8\)/\1/' -e 's/^#\(pt_BR.UTF-8 UTF-8\)/\1/' /etc/default/libc-locales
```

## 12. Personalizar o /etc/rc.conf. Define o fuso horário, layout do teclado e fonte padrão do console. Altere conforme necessidade.
```
sudo tee -a /etc/rc.conf >/dev/null << EOF
TIMEZONE="America/Sao_Paulo"
KEYMAP="br-abnt2"
FONT=Lat2-Terminus16
EOF
```

## 13. Reconfigure
```
sudo xbps-reconfigure -fa
```

## 14. Ativar serviços obrigatórios (runit)
```
sudo ln -s /etc/sv/dbus /var/service/
sudo ln -s /etc/sv/elogind /var/service/
sudo ln -s /etc/sv/polkitd /var/service/
sudo ln -s /etc/sv/NetworkManager /var/service/
sudo ln -s /etc/sv/gdm /var/service/
```

## Finalização
- Usando GDM → o sistema inicia direto no GNOME.
- Sem GDM → usar `startx` (se `.xinitrc` existir).
