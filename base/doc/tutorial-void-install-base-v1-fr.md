# 🔥 Tutoriel d'installation de Base Void Linux

# Avant de commencer

Ce didacticiel décrit une **installation manuelle de Void Linux**, utilisant le partitionnement direct du disque, le « chroot » et la configuration explicite du système.
Ce **n'est pas un installateur automatique**.

## ⚠️ Lisez attentivement

- Ce guide **suppose une familiarité avec Linux**, les concepts de terminal et de système de base (disques, partitions, démarrage, services).
- Plusieurs commandes **suppriment définitivement les données** (`parted`, `mkfs`, `umount -R`).
- Une erreur lors de la définition du disque (`/dev/sdX`, `/dev/nvmeX`) peut entraîner une **perte totale de données**.
- Lisez **l'intégralité du didacticiel avant d'exécuter une commande**.

## 🖥️Environnement recommandé

- **VM (VirtualBox, QEMU, KVM, etc.)** pour les tests et l'apprentissage.
- Matériel dédié **aucune donnée importante**.
- Environnement de laboratoire ou installation consciente.

❌ **Déconseillé** pour une utilisation directe en production sans adaptations.

## 🔐 À propos de la sécurité

Pendant le processus d'installation, certains paramètres **donnent la priorité à l'aspect pratique** et non à la sécurité :
- La connexion de l'utilisateur `root` via SSH peut être temporairement activée.
- L'authentification par mot de passe peut être active.
- La compatibilité héritée (par exemple `ssh-rsa`) peut être autorisée.

👉 **Ces paramètres doivent être revus après l'installation**, notamment sur les systèmes exposés au réseau.

## 🧠 Important à savoir

- Exécutez les commandes **une par une**, en vérifiant le résultat.
- Ajustez les noms de disques, les interfaces réseau et les utilisateurs en fonction de votre système.
- **Ne copiez pas et ne collez pas aveuglément**.
- En cas de doute, **arrêtez** et revoyez l'étape en cours.

## 🛠️ En cas d'erreur

Si quelque chose ne va pas :
- Ne redémarrez pas aveuglément.
- Remontez les cloisons.
- Reconnectez-vous au système avec `chroot`.
- Vérifiez GRUB, EFI et `initramfs`.

Faire des erreurs en fait partie. Comprendre l'erreur est ce qui sépare l'utilisateur de l'opérateur.

---

> Ce guide s'adresse aux utilisateurs qui préfèrent un **contrôle total** sur l'installation, en suivant l'approche Unix classique :
> **comprendre → configurer → valider → continuer**.

## Démarrer l'installation
Commencez par l'ISO Void Linux (x86_64 glibc ou musl).

1. Connectez-vous en tant que root
```bash
connexion : root
mot de passe : voidlinux
```
2. Basculez le shell de *sh* vers *bash*.
*dash/sh* **N'implémente PAS** plusieurs fonctionnalités utilisées par de nombreux scripts.
```bash
frapper
```
3. Modifiez la disposition du clavier en **ABNT2**, en garantissant un mappage correct des accents et des symboles :
```bash
touches de chargement br-abnt2
```

4. Connectez-vous à Internet
- Pour le **Wi-Fi** *(si vous utilisez le câble, ignorez cette étape)* :
```bash
wpa_passphrase "WIFI_NETWORK_NAME" "NETWORK_PASSWORD" > wifi.conf
wpa_supplicant -B -i wlan0 -c wifi.conf
dhcpcd wlan0
```
> 📌 **Remarque :** `wlan0` peut varier (`wlp2s0`, `wlp0s3`, etc.).
> Utilisez la commande ci-dessous pour identifier la bonne interface :
>
> ```bash
> ip -br a
> ```

5. Testez la connexion :
```bash
ping -c3 8.8.8.8
ping -c3 repo-default.voidlinux.org
```

## Activez la connexion de l'utilisateur **root** via SSH (facultatif).
Cette étape s'applique uniquement lorsque le système s'exécute sur une VM ; en cas de démarrage local (sans VM), l'installation peut se dérouler normalement via le terminal local.
- Ceci est nécessaire pour accéder à la **VM depuis l'hôte** et poursuivre l'installation à distance ; après cela, les commandes peuvent être collées/exécutées directement dans le terminal via SSH.

1. Configurez SSH
```bash
echo 'PermitRootLogin oui' >> /etc/ssh/sshd_config
```
2. Redémarrez le service ssh
```bash
sv redémarrer sshd
```

3. Afficher l'adresse IP de l'interface réseau
```bash
ip -4 route obtenir 1.1.1.1 | awk '{print $7}'
```
>Notez l'adresse IP de l'interface réseau et utilisez-la pour vous connecter à la VM via SSH.

4. Accédez à la VM via SSH depuis l'hôte.
```bash
sudo ssh <ip-da-vm>
```
> Mot de passe par défaut : `voidlinux`

## Configurer une invite colorée dans le terminal (facultatif)
Il affichera l'utilisateur, l'hôte, le chemin actuel et l'état de la dernière commande :
```bash
exporter PROMPT_COMMAND='RET=$?'
export PS1='\[\e[1;33m\]\u\[\e[0m\]@\[\e[1;35m\]\h\[\e[0m\]:\[\e[0;37m\]\w\[\e[0m\] \[\e[1;32m\]$( [ $RET -eq 0 ] && printf ✔ || printf "\e[1;31m✘$RET" )\[\e[0m\] \$ '
```
> 📌 Cette invite n'est valable que pour la session en cours ; pour le rendre permanent, ajoutez-le à `.bashrc`.

## Installer les packages requis
⚠️ **IMPORTANT :**
```bash
xbps-install -Sy xbps parted nano vim zstd xz bash-completion
```

## Partitionner le disque
1. Identifiez le disque
```bash
fdisk -l | grep -E '^(Disque|Disque) '
```
> Nous supposerons pour le tutoriel `/dev/sda`

2. Ajustez les variables ci-dessous en fonction du disque qui sera utilisé (**IMPORTANT**) :
```bash
# disques SATA/SCSI (sdX)
exporter DEVICE=/dev/sda
export DEV_EFI=${DEVICE}2
exporter DEV_ROOT=${DEVICE}3
```

> 📌 **Remarque :**
> Pour les disques **NVMe**, le suffixe de partition change (`p`) :
> ```bash
> exporter DEVICE=/dev/nvme0n1
> exporter DEV_EFI=${DEVICE}p2
> exporter DEV_RAIZ=${DEVICE}p3
> ```

3. Partitionnez le disque en utilisant **parted** (mode automatique).
Ce schéma crée :
-Partition BIOS (bios_grub)
-Partition EFI (ESP)
- Partition racine (RACINE)
```bash
wipefs -a "${DEVICE}"
parted --script "${DEVICE}" -- \
mklabelgpt\
mkpart primaire 1 Mo 2 Mo nom 1 BIOS défini 1 bios_grub sur \
mkpart primaire fat32 2 Mo 514 Mo nom 2 EFI défini 2 esp sur \
mkpart primaire 514 Mo 100 % nom 3 RACINE \
align-check optimal 1 \
aligner-vérifier optimal 2 \
align-check optimal 3
parted --script "${DEVICE}" -- imprimer
```

## Formater les partitions
```bash
# Formater la partition racine (ext4)
mkfs.ext4 -F ${DEV_RAIZ}

# Formater la partition EFI (FAT32)
mkfs.fat -F32 -I ${DEV_EFI}
```

## Montez les volumes dans `/mnt`
```bash
# Montez la partition racine
monter ${DEV_RAIZ} /mnt

# Créez les points de montage nécessaires
/t /mnt/{hame,boot/efi,var/log,var/cache, procéd, proc, Proc,)

# Montez la partition EFI
monter ${DEV_EFI} /mnt/boot/efi
```

## Installer le système de base
Installe le système de base Void Linux dans l'environnement monté `/mnt`, y compris le noyau, le micrologiciel, le chargeur de démarrage, la mise en réseau et les outils essentiels.
```bash
xbps-install -Sy -R https://repo-default.voidlinux.org/current \
-r /mnt\
système de base e2fsprogs grub-x86_64-efi dracut linux \
linux-headers linux-firmware linux-firmware-network glibc-locales \
xtools dhcpcd openssh vim nano grc zstd xz bash-completion vpm vsv \
socklog-void wget net-tools tmate ncurses chrony
```

> 📌 **Remarque :**
> - `grub-x86_64-efi` → chargeur de démarrage UEFI
> - `linux` → noyau
> - `linux-firmware-network` → pilotes réseau
> - `xtools` → requis pour utiliser `xgenfstab` sans faute

## Créer `fstab`
Génère automatiquement le fichier de montage permanent du système.
```bash
xgenfstab -U /mnt > /mnt/etc/fstab
```

## Entrée et système (chroot)
Accédez au système installé dans `/mnt` pour continuer la configuration.
```bash
xchroot /mnt /bin/bash
```

## Générer INITRAMFS
Configuration Dracut pour les environnements de virtualisation (VM-safe)
```bash
cat > /etc/dracut.conf.d/99-vm-safe.conf << 'EOF'
# /etc/dracut.conf.d/99-vm-safe.conf
hôte uniquement = non
compresser="gzip"
add_drivers+=" virtio virtio_pci virtio_blk virtio_net virtio_scsi "
EOF
```

Détecte automatiquement la version du noyau installée et génère le `initramfs` correspondant à l'aide de **dracut**.
```bash
mods=(/usr/lib/modules/*)
KVER=$(nom de base "${mods[0]}")
écho ${KVER}
dracut --force --kver ${KVER}
```

## Configurer GRUB

> 📌 Les deux méthodes (BIOS et UEFI) sont installées volontairement.
> Cela permet au même disque de démarrer sur les systèmes **Legacy BIOS** et **UEFI**, augmentant ainsi la portabilité entre les machines.

1. Créez le répertoire de support GRUB :
```bash
mkdir -p /boot/grub
```

2. Installez GRUB pour **BIOS (héritage)** :
```bash
grub-install --target=i386-pc ${DEVICE}
```

3. Installez GRUB pour **UEFI** :
```bash
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=void
```

4. Créez une solution de secours UEFI (démarrage universel).
Ce fichier garantit le démarrage même si la NVRAM est effacée :
```bash
mkdir -p /boot/efi/EFI/BOOT
cp -f /boot/efi/EFI/void/grubx64.efi /boot/efi/EFI/BOOT/BOOTX64.EFI
```

5. Générez le fichier de configuration GRUB final :
```bash
grub-mkconfig -o /boot/grub/grub.cfg
```

## Création et configuration des utilisateurs

⚠️ **IMPORTANT :** définissez le vrai nom d'utilisateur ci-dessous.
```bash
export NEWUSER=votre_utilisateur_ici
```

Créez l'utilisateur avec le répertoire personnel, les groupes de base et le shell Bash :
```bash
useradd -m -G audio, vidéo, roue, tty -s /bin/bash ${NEWUSER}
```

Définissez votre mot de passe utilisateur (***IMPORTANT***)
```bash
mot de passe ${NEWUSER}
```

Définir le mot de passe de l'utilisateur root (***IMPORTANT***)
```bash
mot de passe racine
```

Changer le shell par défaut de l'utilisateur root en Bash
```bash
chsh -s /bin/bash racine
```

## Paramètres de base
```bash
# Nom d'hôte Setar
echo void > /etc/nom d'hôte

# Définir l'heure locale
ln -sf /usr/share/zoneinfo/America/Sao_Paulo /etc/localtime

# Sétar local
sed -i 's/#en_US.UTF-8/en_US.UTF-8/' /etc/default/libc-locales
sed -i 's/#pt_BR.UTF-8/pt_BR.UTF-8/' /etc/default/libc-locales

# Générer des locales :
xbps-reconfigure -f glibc-locales

# Correction d'une erreur possible dans le lien symbolique /var/service (important) :
rm -f /var/service
ln -sf /etc/runit/runsvdir/default /var/service

# Activez certains services :
ln -sf /etc/sv/dbus /var/service/
ln -sf /etc/sv/dhcpcd /var/service/
ln -sf /etc/sv/sshd /var/service/
ln -sf /etc/sv/nanoklogd /var/service/
ln -sf /etc/sv/socklog-unix /var/service/
ln -sf /etc/sv/chronyd /var/service/

# télécharger un svlogtail personnalisé (facultatif, mais recommandé) :
wget --quiet --no-check-certificate -O /usr/bin/svlogtail \
"https://raw.githubusercontent.com/voidlinux-br/void-install/refs/heads/main/svlogtail" &&\
chmod +x /usr/bin/svlogtail

# Créer un resolv.conf
printf 'serveur de noms 1.1.1.1\nserveur de noms 8.8.8.8\n' > /etc/resolv.conf

#Configurer sudo - groupe de roues (facultatif, mais recommandé)
chat << 'EOF' > /etc/sudoers.d/g_wheel
%wheel ALL=(TOUS:TOUS) NOPASSWD: TOUS
EOF

#Autorisations requises
chmod 440 /etc/sudoers.d/g_wheel
```

## Personnaliser `/etc/xbps.d/00-repository-main.conf`
*(Facultatif, mais recommandé)*

Crée le répertoire de configuration **XBPS** (s'il n'existe pas déjà) et définit une liste de référentiels officiels et alternatifs.
Les référentiels **repo-fastly** ont tendance à avoir une meilleure latence.

```bash
mkdir -pv /etc/xbps.d

cat << 'EOF' > /etc/xbps.d/00-repository-main.conf
# Dépôt officiel (Fastly – meilleure latence)
référentiel=https://repo-fastly.voidlinux.org/current
#dépôt=https://repo-fastly.voidlinux.org/current/nonfree
#dépôt=https://repo-fastly.voidlinux.org/current/multilib
#dépôt=https://repo-fastly.voidlinux.org/current/multilib/nonfree

# Dépôt alternatif (Chili Linux)
référentiel=https://void.chililinux.com/voidlinux/current
#dépôt=https://void.chililinux.com/voidlinux/current/extras
#dépôt=https://void.chililinux.com/voidlinux/current/nonfree
#dépôt=https://void.chililinux.com/voidlinux/current/multilib
#dépôt=https://void.chililinux.com/voidlinux/current/multilib/nonfree
EOF
```

## Personnaliser `/etc/rc.conf`
Définit le fuseau horaire, la disposition du clavier et la police par défaut de la console.
Changez si nécessaire.
```bash
chat << 'EOF' > /etc/rc.conf
TIMEZONE=Amérique/Sao_Paulo
KEYMAP=br-abnt2
POLICE=Lat2-Terminus16
EOF
```

Modules Virtio (machine virtuelle).
```bash
cat > /etc/modules-load.d/virtio.conf << 'EOF'
virtio
virtio_pci
virtio_net
virtio_blk
virtio_scsi
EOF
```

## Personnaliser l'utilisateur `.bashrc`
Crée un « .bash_profile » par défaut et garantit que « .bashrc » est automatiquement chargé lors de la connexion.
> ⚠️ Assurez-vous que l'utilisateur a été créé à l'étape précédente.

```bash
# Téléchargez le .bashrc par défaut dans /etc/skel
wget --quiet --no-check-certificate \
-O /etc/skel/.bashrc\
"https://raw.githubusercontent.com/voidlinux-br/void-install/refs/heads/main/.bashrc"

racine chown :root /etc/skel/.bashrc
chmod 644 /etc/skel/.bashrc

# Créer un .bash_profile par défaut
cat << 'EOF' > /etc/skel/.bash_profile
# ~/.bash_profile — charge .bashrc dans Void

# Si .bashrc existe, chargez
si [ -f ~/.bashrc ]; alors
source ~/.bashrc
être
EOF

# Copier vers la racine et l'utilisateur
pour d dans /root "/home/${NEWUSER}" ; faire
cp -f /etc/skel/.bash_profile "$d/"
cp -f /etc/skel/.bashrc "$d/"
fait

# Ajuster les autorisations des utilisateurs
chown "${NEWUSER}:${NEWUSER}" \
"/home/${NEWUSER}/.bash_profile" \
"/home/${NEWUSER}/.bashrc"

chmod 644 \
"/home/${NEWUSER}/.bash_profile" \
"/home/${NEWUSER}/.bashrc"
```

## Configurer SSH
*(Facultatif, mais recommandé)*

Crée un fichier de configuration supplémentaire pour **sshd**, en laissant le fichier principal intact.
```bash
mkdir -pv /etc/ssh/sshd_config.d

cat << 'EOF' > /etc/ssh/sshd_config.d/10-custom.conf
# Paramètres généraux
PermisTTY oui
PrintMotd oui
PrintLastLog oui
Bannière /etc/issue.net

# Authentification
PermitRootLogin oui
Authentification par mot de passe oui
KbdInteractiveAuthentication oui
ChallengeResponseAuthentication oui
PubkeyAuthentification oui
PubkeyAcceptedKeyTypes=+ssh-rsa
AuthorizedKeysFile .ssh/authorized_keys
Utiliser PAM oui

# Caractéristiques
X11Transfert oui
Sous-système sftp interne-sftp
EOF
```
> ⚠️ Il est recommandé de revoir et de renforcer ces paramètres SSH après le premier démarrage, notamment sur les systèmes exposés à Internet.

## Finaliser l'installation
Sair faire du chroot
```bash
sortie
```
Démontez toutes les partitions montées sur `/mnt` (y compris les sous-volumes et `/boot/efi`) :
```bash
umount -R /mnt
```
Redémarrez la machine physique ou la VM pour tester le démarrage réel :
```bash
redémarrer
```
> 📌 **Remarque : N'oubliez pas de retirer le support d'installation.

# 🎉 Profitez-en !
**Void Linux** est maintenant installé et prêt à être utilisé.

# CLAUSE DE NON-RESPONSABILITÉ

> Ce tutoriel est gratuit : vous pouvez utiliser, copier, modifier et redistribuer à votre guise.
> Le contenu est mis à disposition sous la **licence MIT** et peut inclure des extraits ou des commandes dérivées de logiciels open source, sous réserve de leurs propres licences.
>
> Aucune garantie n'est fournie — tout ici est livré **« tel quel »**.
> Utilisez à vos propres risques. Ni l'auteur, ni les contributeurs, ni Void Linux ne sont responsables des pertes, dommages, pannes du système ou de toute conséquence de l'utilisation de ce matériel.
>
> Vous êtes libre de réviser, d'adapter et de générer votre propre version de ce tutoriel.

