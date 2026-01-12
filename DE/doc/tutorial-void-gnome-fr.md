# 🐧 Void Linux + GNOME — Tutoriel

> ⚠️ **IMPORTANT — À LIRE AVANT DE COMMENCER**
>
> Ce tutoriel **ne doit PAS être exécuté en tant que `root`**, sauf lorsque **explicitement indiqué**.
>
> Toutes les commandes ont été conçues pour être exécutées par **un utilisateur commun**, en utilisant `sudo` si nécessaire.
>
> Exécutez l'intégralité du tutoriel connecté en tant que `root` :
> - brise la logique des autorisations
> - invalida etapas como configuração de `sudo`
> - peut générer des erreurs silencieuses ou un comportement inattendu
>
> 👉 **Recommandation**
> Si vous venez d'installer le système et que vous êtes connecté en tant que « root » :
>
> 1. Créer un utilisateur commun
> 2. Connectez-vous avec cet utilisateur
> 3. Suivez le tutoriel normalement
>
> Règle classique pour les systèmes Unix/Linux :
>
> **`root` est une exception. L'utilisateur commun est la règle.**

---

## 0. Configurez sudo - wheel group - pour éviter de demander le mot de passe root
```
sudo tee -a /etc/sudoers.d/g_wheel >/dev/null << EOF
%wheel ALL=(TOUS:TOUS) NOPASSWD: TOUS
EOF

#Autorisations requises
sudo chmod 440 /etc/sudoers.d/g_wheel
```

## 1. Mettre à jour le système
```
sudo xbps-install -Syu
```

## 2. Installer GNOME complet (méta-paquet)
```
sudo xbps-install -y gnome \
thème-icône-gnome \
papiers \
applet-gestionnaire-réseau \
gestionnaire d'extensions \
nautile \
extension-papers-nautile \
nautilus-gnome-console-extension \
nautilus-gnome-terminal-extension \
gnome-terminal \
thème d'arc \
Firefox \
Firefox-i18n-pt-BR \
xarchiver \
utilitaire-disque-gnome \
séparé \
gvfs\
p7zip\
décompresser \
eog \
noto-fonts-emoji \
htop
```

## 3. Installez GDM (gestionnaire d'affichage officiel)
```
sudo xbps-install -y gdm
```

## 4. Pilotes d'affichage

### Intel
```
sudo xbps-install -y mesa-dri linux-firmware-intel
```

### nouveau AMD (amdgpu)
```
sudo xbps-install -y mesa-dri xf86-video-amdgpu
```

### ancienne DMLA
```
sudo xbps-install -y mesa-dri xf86-video-ati
```

### Nvidia (pilote ouvert)
```
sudo xbps-install -y mesa-nouveau-dri
```

## 5. Installez PipeWire (Modern Void Sound)
```
sudo xbps-install -y \
fil de tuyauterie \
plombier \
alsa-plugins-pulseaudio \
alsa-pipewire \
libjack-pipewire\
pulseaudio-utils\
alsa-utils\
pavucontrol
```

## 6. Intégrer ALSA → PipeWire
```
sudo mkdir -p /etc/alsa/conf.d
sudo ln -sf /usr/share/alsa/alsa.conf.d/50-pipewire.conf /etc/alsa/conf.d
sudo ln -sf /usr/share/alsa/alsa.conf.d/99-pipewire-default.conf /etc/alsa/conf.d
```

## 7. Activer le serveur pipewire-pulse (compatibilité PulseAudio)
```
sudo mkdir -p /etc/pipewire/pipewire.conf.d
sudo ln -sf /usr/share/examples/pipewire/20-pipewire-pulse.conf /etc/pipewire/pipewire.conf.d/
```

## 8. Activer le démarrage automatique de PipeWire en session
```
mkdir -p ~/.config/autostart
ln -sf /usr/share/applications/pipewire.desktop ~/.config/autostart/
ln -sf /usr/share/applications/pipewire-pulse.desktop ~/.config/autostart/
ln -sf /usr/share/applications/wireplumber.desktop ~/.config/autostart/
```

## 9. (Facultatif) Créez .xinitrc pour startx
```
chat <<EOF > ~/.xinitrc
#!/bin/sh
setxkbmap -layout br -variante abnt2 &
exécution de la session gnome
EOF
```

## 10. configurer le fuseau horaire - définit le fuseau horaire
```
sudo ln -sfv /usr/share/zoneinfo/America/Sao_Paulo /etc/localtime
```

## 11. configurer les paramètres régionaux
```
Sudo -i -i -e 's/s/#\(en_.us.utf-8 utf-8\)/' -e 's/^#\pt_br.br.br.
```

## 12. Personnalisez /etc/rc.conf. Définit le fuseau horaire, la disposition du clavier et la police par défaut de la console. Changez si nécessaire.
```
sudo tee -a /etc/rc.conf >/dev/null << EOF
TIMEZONE="Amérique/Sao_Paulo"
KEYMAP="br-abnt2"
POLICE=Lat2-Terminus16
EOF
```

## 13. Personnalisez /etc/locale.conf. Définit la langue. Changez si nécessaire.
```
sudo tee /etc/locale.conf >/dev/null << EOF
LANG=pt_BR.UTF-8
LANGUE=pt_BR.UTF-8
LC_COLLATE=pt_BR.UTF-8
EOF
```

## 14. Reconfigurer
```
sudo xbps-reconfigure -fa
```

## 15. Activer les services obligatoires (runit)
```
sudo ln -s /etc/sv/dbus /var/service/
sudo ln -s /etc/sv/elogind /var/service/
sudo ln -s /etc/sv/polkitd /var/service/
sudo ln -s /etc/sv/NetworkManager /var/service/
sudo ln -s /etc/sv/gdm /var/service/
```

## Finalisation
- Utilisation de GDM → le système démarre directement dans GNOME.
- Sans GDM → utiliser `startx` (si `.xinitrc` existe).
