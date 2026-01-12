# 🐧 Void Linux + GNOME – Tutorial

> ⚠️ **WICHTIG – LESEN SIE, BEVOR SIE BEGINNEN**
>
> Dieses Tutorial **sollte NICHT als „root“** ausgeführt werden, außer wenn **ausdrücklich angegeben**.
>
> Alle Befehle wurden so konzipiert, dass sie von **einem normalen Benutzer** ausgeführt werden können, wobei bei Bedarf „sudo“ verwendet wird.
>
> Führen Sie das gesamte Tutorial aus, angemeldet als „root“:
> – bricht die Berechtigungslogik
> – macht Schritte wie die „sudo“-Konfiguration ungültig
> – kann stille Fehler oder unerwartetes Verhalten erzeugen
>
> 👉 **Empfehlung**
> Se você acabou de instalar o sistema e está logado como `root`:
>
> 1. Erstellen Sie einen gemeinsamen Benutzer
> 2. Melden Sie sich mit diesem Benutzer an
> 3. Befolgen Sie die Anleitung wie gewohnt
>
> Klassische Regel für Unix/Linux-Systeme:
>
> **`root` ist eine Ausnahme. Normaler Benutzer ist die Regel.**

---

## 0. Konfigurieren Sie die Sudo-Radgruppe, um die Nachfrage nach dem Root-Passwort zu vermeiden
```
sudo tee -a /etc/sudoers.d/g_wheel >/dev/null << EOF
%wheel ALL=(ALL:ALL) NOPASSWD: ALLE
EOF

#Erforderliche Berechtigungen
sudo chmod 440 /etc/sudoers.d/g_wheel
```

## 1. Aktualisieren Sie das System
```
sudo xbps-install -Syu
```

## 2. Vollständiges GNOME installieren (Metapaket)
```
sudo xbps-install -y gnome \
Gnome-Icon-Theme \
Papiere \
Netzwerkmanager-Applet \
Erweiterungsmanager \
Nautilus \
nautilus-papers-extension \
nautilus-gnome-console-extension \
nautilus-gnome-terminal-extension \
Gnome-Terminal \
Arc-Theme \
Firefox \
firefox-i18n-pt-BR \
xarchiver \
gnome-disk-utility \
gparted \
gvfs\
p7zip \
entpacken \
eog \
noto-fonts-emoji \
htop
```

## 3. GDM (offizieller Display Manager) installieren
```
sudo xbps-install -y gdm
```

## 4. Treiber anzeigen

### Intel
```
sudo xbps-install -y mesa-dri linux-firmware-intel
```

### neues AMD (amdgpu)
```
sudo xbps-install -y mesa-dri xf86-video-amdgpu
```

### alter AMD
```
sudo xbps-install -y mesa-dri xf86-video-ati
```

### Nvidia (offener Treiber)
```
sudo xbps-install -y mesa-nouveau-dri
```

## 5. Installieren Sie PipeWire (Modern Void Sound)
```
sudo xbps-install -y \
Rohrdraht \
Kabelklempner \
alsa-plugins-pulseaudio \
alsa-pipewire \
libjack-pipewire \
Pulsaudio-Utils \
alsa-utils \
pavucontrol
```

## 6. Integrieren Sie ALSA → PipeWire
```
sudo mkdir -p /etc/alsa/conf.d
sudo ln -sf /usr/share/alsa/alsa.conf.d/50-pipewire.conf /etc/alsa/conf.d
sudo ln -sf /usr/share/alsa/alsa.conf.d/99-pipewire-default.conf /etc/alsa/conf.d
```

## 7. Pipewire-Pulse-Server aktivieren (PulseAudio-kompatibel)
```
sudo mkdir -p /etc/pipewire/pipewire.conf.d
sudo ln -sf /usr/share/examples/pipewire/20-pipewire-pulse.conf /etc/pipewire/pipewire.conf.d/
```

## 8. Aktivieren Sie den PipeWire-Autostart in der Sitzung
```
mkdir -p ~/.config/autostart
ln -sf /usr/share/applications/pipewire.desktop ~/.config/autostart/
ln -sf /usr/share/applications/pipewire-pulse.desktop ~/.config/autostart/
ln -sf /usr/share/applications/wireplumber.desktop ~/.config/autostart/
```

## 9. (Optional) Erstellen Sie .xinitrc für startx
```
cat <<EOF > ~/.xinitrc
#!/bin/sh
setxkbmap -layout br -variant abnt2 &
exec gnome-session
EOF
```

## 10. Zeitzone konfigurieren – definiert die Zeitzone
```
sudo ln -sfv /usr/share/zoneinfo/America/Sao_Paulo /etc/localtime
```

## 11. Gebietsschemas konfigurieren
```
Sudo -i -i -e 's/s/#\(en_.us.utf-8 utf-8\)/' -e 's/^#\pt_br.br.br.
```

## 12. Passen Sie /etc/rc.conf an. Legt die Standardzeitzone, das Tastaturlayout und die Schriftart der Konsole fest. Ändern Sie es nach Bedarf.
```
sudo tee -a /etc/rc.conf >/dev/null << EOF
TIMEZONE="Amerika/Sao_Paulo"
KEYMAP="br-abnt2"
FONT=Lat2-Terminus16
EOF
```

## 13. Passen Sie /etc/locale.conf an. Legt die Sprache fest. Ändern Sie es nach Bedarf.
```
sudo tee /etc/locale.conf >/dev/null << EOF
LANG=pt_BR.UTF-8
LANGUAGE=pt_BR.UTF-8
LC_COLLATE=pt_BR.UTF-8
EOF
```

## 14. Neu konfigurieren
```
sudo xbps-reconfigure -fa
```

## 15. Pflichtdienste aktivieren (runit)
```
sudo ln -s /etc/sv/dbus /var/service/
sudo ln -s /etc/sv/elogind /var/service/
sudo ln -s /etc/sv/polkitd /var/service/
sudo ln -s /etc/sv/NetworkManager /var/service/
sudo ln -s /etc/sv/gdm /var/service/
```

## Finalisierung
- Mit GDM → startet das System direkt in GNOME.
- Ohne GDM → „startx“ verwenden (falls „.xinitrc“ vorhanden ist).
