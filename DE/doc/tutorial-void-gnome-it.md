# 🐧 Void Linux + GNOME — Tutorial

> ⚠️ **IMPORTANTE — LEGGERE PRIMA DI INIZIARE**
>
> Questo tutorial **NON deve essere eseguito come `root`**, tranne quando **esplicitamente indicato**.
>
> Tutti i comandi sono stati progettati per essere eseguiti da **un utente comune**, utilizzando `sudo` quando necessario.
>
> Esegui l'intero tutorial accedendo come `root`:
> - interrompe la logica delle autorizzazioni
> - invalida passaggi come la configurazione di `sudo`
> - potrebbe generare errori silenziosi o comportamenti imprevisti
>
> 👉 **Raccomandazione**
> Se hai appena installato il sistema e hai effettuato l'accesso come `root`:
>
> 1. Creare un utente comune
> 2. Accedi con questo utente
> 3. Segui normalmente il tutorial
>
> Regola classica per i sistemi Unix/Linux:
>
> **`root` è un'eccezione. L'utente comune è la regola.**

---

## 0. Configura sudo - wheel group - per evitare di richiedere la password di root
```
sudo tee -a /etc/sudoers.d/g_wheel >/dev/null << EOF
%wheel ALL=(ALL:ALL) NOPASSWD: TUTTI
EOF

#Autorizzazioni richieste
sudo chmod 440 /etc/sudoers.d/g_wheel
```

## 1. Aggiorna il sistema
```
sudo xbps-install -Syu
```

## 2. Installa GNOME completo (metapacchetto)
```
sudo xbps-install -y gnome \
tema-icone-gnome \
documenti \
applet-gestore-rete \
gestore estensioni \
Nautilus \
estensione-nautilus-papers \
nautilus-gnome-estensione-console \
estensione-terminale-nautilus-gnome \
terminale gnome \
tema dell'arco \
firefox \
firefox-i18n-pt-BR \
xarchiver \
utilità-disco-gnome \
partizionato \
gvfs\
p7zip \
decomprimere \
eog \
noto-fonts-emoji \
htop
```

## 3. Installa GDM (display manager ufficiale)
```
sudo xbps-install -y gdm
```

## 4. Driver video

### Intel
```
sudo xbps-install -y mesa-dri linux-firmware-intel
```

### nuova AMD (amdgpu)
```
sudo xbps-install -y mesa-dri xf86-video-amdgpu
```

### vecchia AMD
```
sudo xbps-install -y mesa-dri xf86-video-ati
```

### Nvidia (driver aperto)
```
sudo xbps-install -y mesa-nouveau-dri
```

## 5. Installa PipeWire (Modern Void Sound)
```
sudo xbps-install -y \
tubo \
Idraulico \
alsa-plugins-pulseaudio \
alsa-pipewire \
libjack-pipewire \
Pulseaudio-utils \
alsa-utils \
pavucontrol
```

## 6. Integra ALSA → PipeWire
```
sudo mkdir -p /etc/alsa/conf.d
sudo ln -sf /usr/share/alsa/alsa.conf.d/50-pipewire.conf /etc/alsa/conf.d
sudo ln -sf /usr/share/alsa/alsa.conf.d/99-pipewire-default.conf /etc/alsa/conf.d
```

## 7. Abilita il server pipewire-pulse (compatibile con PulseAudio)
```
sudo mkdir -p /etc/pipewire/pipewire.conf.d
sudo ln -sf /usr/share/examples/pipewire/20-pipewire-pulse.conf /etc/pipewire/pipewire.conf.d/
```

## 8. Abilita l'avvio automatico di PipeWire nella sessione
```
mkdir -p ~/.config/autostart
ln -sf /usr/share/applications/pipewire.desktop ~/.config/autostart/
ln -sf /usr/share/applications/pipewire-pulse.desktop ~/.config/autostart/
ln -sf /usr/share/applications/wireplumber.desktop ~/.config/autostart/
```

## 9. (Facoltativo) Crea .xinitrc per startx
```
cat <<EOF > ~/.xinitrc
#!/bin/sh
setxkbmap -layout br -variant abnt2 &
exec gnome-session
EOF
```

## 10. configura fuso orario: definisce il fuso orario
```
sudo ln -sfv /usr/share/zoneinfo/America/Sao_Paulo /etc/localtime
```

## 11. configurare le impostazioni locali
```
Sudo -i -i -e 's/s/#\(en_.us.utf-8 utf-8\)/' -e 's/^#\pt_br.br.br.
```

## 12. Personalizza /etc/rc.conf. Imposta il fuso orario, il layout della tastiera e il carattere predefiniti della console. Cambia secondo necessità.
```
sudo tee -a /etc/rc.conf >/dev/null << EOF
TIMEZONE="America/Sao_Paulo"
KEYMAP="br-abnt2"
FONT=Lat2-Terminus16
EOF
```

## 13. Personalizza /etc/locale.conf. Imposta la lingua. Cambia secondo necessità.
```
sudo tee /etc/locale.conf >/dev/null << EOF
LANG=pt_BR.UTF-8
LINGUA=pt_BR.UTF-8
LC_COLLATE=pt_BR.UTF-8
EOF
```

## 14. Riconfigurare
```
sudo xbps-reconfigure -fa
```

## 15. Attiva i servizi obbligatori (runit)
```
sudo ln -s /etc/sv/dbus /var/service/
sudo ln -s /etc/sv/elogind /var/service/
sudo ln -s /etc/sv/polkitd /var/service/
sudo ln -s /etc/sv/NetworkManager /var/service/
sudo ln -s /etc/sv/gdm /var/service/
```

## Finalizzazione
- Utilizzando GDM → il sistema si avvia direttamente in GNOME.
- Senza GDM → usa `startx` (se esiste `.xinitrc`).
