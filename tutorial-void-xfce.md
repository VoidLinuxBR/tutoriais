# 🐧 Void Linux + XFCE4 — Tutorial Definitivo

## 1. Atualizar o sistema
```
sudo xbps-install -Syu
```

## 2. Instalar Xorg + Xinit + Xterm
```
sudo xbps-install -y xorg xinit xterm
```

## 3. Instalar XFCE4 completo
```
sudo xbps-install -y xfce4 \
   network-manager-applet \
   xfce4-plugins \
   arc-theme \
   xfce4-screenshooter \
   xfce4-whiskermenu-plugin \
   firefox \
   firefox-i18n-pt-BR \
   xarchiver \
   thunar-archive-plugin \
   gnome-disk-utility \
   gparted \
   gvfs \
   p7zip \
   unzip \
   htop
```

## 4. Instalar LXDM (display manager leve)
```
sudo xbps-install -y lxdm
```

## 5. Drivers de vídeo (escolher apenas um)

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

### Nvidia proprietária
```
sudo xbps-install -y mesa-nouveau-dri
```

## 6. Desativar o XFCE-Polkit (bugado — não usar)
```
mkdir -p ~/.config/autostart
cp /etc/xdg/autostart/xfce-polkit.desktop ~/.config/autostart/
sed -i 's/^Hidden=.*/Hidden=true/' ~/.config/autostart/xfce-polkit.desktop || echo 'Hidden=true' >> ~/.config/autostart/xfce-polkit.desktop
```

## 7. Instalar PipeWire (som moderno do Void)
```
sudo xbps-install -y pipewire wireplumber alsa-pipewire pulseaudio-utils pavucontrol
```

## 8. Integrar ALSA → PipeWire
```
sudo mkdir -p /etc/alsa/conf.d
sudo ln -sf /usr/share/alsa/alsa.conf.d/50-pipewire.conf /etc/alsa/conf.d
sudo ln -sf /usr/share/alsa/alsa.conf.d/99-pipewire-default.conf /etc/alsa/conf.d
```

## 9. Habilitar servidor pipewire-pulse (compat PulseAudio)
```
sudo mkdir -p /etc/pipewire/pipewire.conf.d
sudo ln -sf /usr/share/examples/pipewire/20-pipewire-pulse.conf /etc/pipewire/pipewire.conf.d/
```

## 10. Ativar autostart PipeWire na sessão do XFCE
```
mkdir -p ~/.config/autostart
ln -sf /usr/share/applications/pipewire.desktop ~/.config/autostart/
ln -sf /usr/share/applications/pipewire-pulse.desktop ~/.config/autostart/
ln -sf /usr/share/applications/wireplumber.desktop ~/.config/autostart/
```

## 11. Plugins essenciais do XFCE (ícone de som)
```
sudo xbps-install -y xfce4-pulseaudio-plugin xfce4-notifyd

XML="$HOME/.config/xfce4/xfconf/xfce-perchannel-xml/xfce4-panel.xml"
if ! grep -q 'pulseaudio' "$XML"; then
   NEW_ID=$(( $(grep -o 'plugin-[0-9]\+' "$XML" | awk -F- '{print $2}' | sort -n | tail -1) + 1 ))
   grep -q "<value type=\"int\" value=\"$NEW_ID\"/>" "$XML" || \
   sed -i "/<property name=\"plugin-ids\" type=\"array\">/a\  <value type=\"int\" value=\"$NEW_ID\"/>" "$XML"
   sed -i "
   \$s#</property>#    <property name=\"plugin-$NEW_ID\" type=\"string\" value=\"pulseaudio\">\n      <property name=\"enable-keyboard-shortcuts\" type=\"bool\" value=\"true\"/>\n    </property>\n</property>#
   " "$XML"
fi

```

## 12. Criar .xinitrc (opcional para startx)
```
cat <<EOF > ~/.xinitrc
#!/bin/sh
setxkbmap -layout br -variant abnt2 &
xsetroot -cursor_name left_ptr &
exec startxfce4
EOF
```

## 13. configurar timezone - define o fuso horário
```
sudo ln -sfv /usr/share/zoneinfo/America/Sao_Paulo /etc/localtime
```

## 14. configure locales
```
sudo sed -i -e 's/^#\(en_US.UTF-8 UTF-8\)/\1/' -e 's/^#\(pt_BR.UTF-8 UTF-8\)/\1/' /etc/default/libc-locales
```

## 15. Personalizar o /etc/rc.conf. Define o fuso horário, layout do teclado e fonte padrão do console. Altere conforme necessidade.
```
sudo tee -a /etc/rc.conf >/dev/null << EOF
TIMEZONE="America/Sao_Paulo"
KEYMAP="br-abnt2"
FONT=Lat2-Terminus16
EOF
```

## 16. Reconfigure
```
sudo xbps-reconfigure -fa
```

## 17. Ativar serviços obrigatórios (runit)
```
sudo ln -s /etc/sv/dbus /var/service/
sudo ln -s /etc/sv/elogind /var/service/
sudo ln -s /etc/sv/polkitd /var/service/
sudo ln -s /etc/sv/NetworkManager /var/service/
sudo ln -s /etc/sv/lxdm /var/service/
```

## Finalização
- Se LXDM estiver ativo: boot direto em GUI.
- Se quiser modo clássico: startx
