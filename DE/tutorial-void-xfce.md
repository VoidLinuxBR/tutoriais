# 🐧 Void Linux + XFCE4 — Tutorial


> ⚠️ **IMPORTANTE — LEIA ANTES DE COMEÇAR**
>
> Este tutorial **NÃO deve ser executado como `root`**, exceto quando **explicitamente indicado**.
>
> Todos os comandos foram pensados para serem executados por **um usuário comum**, utilizando `sudo` quando necessário.
>
> Executar todo o tutorial logado como `root`:
> - quebra a lógica de permissões
> - invalida etapas como configuração de `sudo`
> - pode gerar erros silenciosos ou comportamentos inesperados
>
> 👉 **Recomendação**  
> Se você acabou de instalar o sistema e está logado como `root`:
>
> 1. Crie um usuário comum
> 2. Faça login com esse usuário
> 3. Siga o tutorial normalmente
>
> Regra clássica de sistemas Unix/Linux:
>
> **`root` é exceção. Usuário comum é regra.**

---

## 0. Configurar sudo - (grupo wheel) - evita ficar pedindo senha de root
```
sudo usermod -aG wheel "$USER"

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

```
#limpar cache de pacotes
sudo rm -fv /var/cache/xbps/*.xbps /var/cache/xbps/*.sig*
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
   noto-fonts-emoji \
   font-hack-ttf \
   nerd-fonts-symbols-ttf \
   nerd-fonts-ttf \
   htop
```

## 4. Instalar LXDM (display manager leve)
```
sudo xbps-install -y lxdm
```

## 5. Drivers de vídeo

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
⚠️    **IMPORTANTE:**
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
if [[ -f "$XML" ]] && ! grep -q 'pulseaudio' "$XML"; then
   next=$(grep -o 'plugin-[0-9]*' "$XML" 2>/dev/null | sed 's/.*-//' | sort -n | tail -1)
   next=$((next + 1))

   sed -i '$d' "$XML"   # remove o ũltimo </channel>
   sed -i '$d' "$XML"   # remove o último </property>

   cat << EOF >> "$XML"
    <property name="plugin-$next" type="string" value="pulseaudio">
      <property name="enable-keyboard-shortcuts" type="bool" value="true"/>
    </property>
  </property>
</channel>
EOF

   sed -i '/<property name="panel-1" type="empty">/,/<\/property>/ {
      /<property name="plugin-ids" type="array">/,/<\/property>/ {
      /<\/property>/i\
         <value type="int" value="'"$next"'"/>
      }
   }' "$XML"
fi
```
> Se caso no primeiro login ainda não aparecer o ícone do som
abra o terminal e rode esse script, depois logout e login novamente

## 12. configurar timezone - define o fuso horário
```
sudo ln -sfv /usr/share/zoneinfo/America/Sao_Paulo /etc/localtime
```

## 13. configure locales
```
sudo sed -i -e 's/^#\(en_US.UTF-8 UTF-8\)/\1/' -e 's/^#\(pt_BR.UTF-8 UTF-8\)/\1/' /etc/default/libc-locales
```

## 14. Personalizar o /etc/rc.conf. Define o fuso horário, layout do teclado e fonte padrão do console.
⚠️    **Altere conforme necessidade**

```
sudo tee -a /etc/rc.conf >/dev/null << EOF
TIMEZONE="America/Sao_Paulo"
KEYMAP="br-abnt2"
FONT=Lat2-Terminus16
EOF
```

## 15. Personalizar o /etc/locale.conf. Define o idioma. Altere conforme necessidade.
⚠️    **IMPORTANTE:**
```
sudo tee /etc/locale.conf >/dev/null << EOF
LANG=pt_BR.UTF-8
LANGUAGE=pt_BR.UTF-8
LC_COLLATE=pt_BR.UTF-8
EOF
```

## 16. Reconfigure
```
sudo xbps-reconfigure -fa
```

## 17. Criar .xinitrc (para startx)
```
cat <<EOF > ~/.xinitrc
#!/bin/sh
setxkbmap -layout br -variant abnt2 &
xsetroot -cursor_name left_ptr &
exec startxfce4
EOF
```

## 18. Ativar serviços obrigatórios (runit)
```
sudo ln -sf /etc/sv/dbus /var/service/
sudo ln -sf /etc/sv/elogind /var/service/
sudo ln -sf /etc/sv/polkitd /var/service/
sudo ln -sf /etc/sv/NetworkManager /var/service/
sudo ln -sf /etc/sv/lxdm /var/service/
```

## Finalização
- Se LXDM estiver ativo: boot direto em GUI.
- Se quiser modo clássico: startx
