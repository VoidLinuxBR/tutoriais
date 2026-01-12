# 🧩 TUTORIEL VOID LINUX — INSTALLATION HYBRIDE (UEFI + BIOS) AVEC EXT4, XFS, JFS OU BTRFS (SOUS-VOLUMES), LUKS, HIBERNATION ET ZRAM
### VERSION RÉVISÉE ET VALIDÉE — PARTITIONNEMENT CORRECT + BOOT UNIVERSEL

Ce guide installe un Void Linux entièrement **hybride**, capable de démarrer n'importe quel type de machine - ancienne, nouvelle ou problématique :

- 💾 **UEFI moderne** (avec entrée normale et repli)
- 🧮 **BIOS/Legacy** (compatibilité totale)
- 🧰 **GPT avec BIOS Boot (EF02)** — prise en charge maximale des anciens matériels
- 🚀 **Btrfs avec sous-volumes** (facultatif), instantanés prêts à l'emploi
- 🔐 **LUKS1 entièrement compatible avec GRUB**
- 🌙 **Véritable hibernation via swapfile**
- 🧊 **ZRAM configurée pour les performances**
- 🧱 **Support complet pour EXT4, XFS, JFS et BTRFS**
- 💡 **Initramfs/GRUB configuré automatiquement (LUKS + reprise)**

📌 **Aucun compromis, pas de réinstallation de GRUB, pas de perte de temps.**
📌 **Démarrage garanti même sur une machine avec NVRAM effacée (repli BOOTX64.EFI).**

---

# ▶️ 1. Démarrer ou Live ISO

Suggestion : utilisez la version glibc pour une compatibilité supérieure :
- télécharger l'iso depuis :
```
https://repo-default.voidlinux.org/live/current/void-live-x86_64-20250202-base.iso
```
- ou recherchez la dernière version sur :
```
https://voidlinux.org/download/
```

1. Connectez-vous en tant que root.
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

4. Collez dans le terminal (facultatif) — Invite avec les couleurs, user@host:path et l'état de la dernière commande (✔/✘). Utile et beau.
```bash
export PS1='\[\e[1;32m\]\u\[\e[1;33m\]@\[\e[1;36m\]\h\[\e[1;31m\]:\w \
$([[ $? -eq 0 ]] && echo -e "\e[1;32m✔" || echo -e "\e[1;31m✘$?") \
\[\e[0m\]\$ '
```

# ▶️ 2. Connectez-vous à Internet
- Pour le **Wi-Fi** *(si vous utilisez le câble, ignorez cette étape)* :
```bash
wpa_passphrase "SSID" "MOT DE PASSE" > wifi.conf
wpa_supplicant -B -i wlan0 -c wifi.conf
dhcpcd wlan0
```

1. Testez la connexion :
```bash
ping -c3 8.8.8.8
ping -c3 repo-default.voidlinux.org
```

2. Installez les packages requis :
⚠️ **IMPORTANT :**
```bash
xbps-install -Sy xbps séparé jfsutils xfsprogs nano zstd xz bash-completion
```
---

# ▶️ 3. Identifiez le disque
1. Listez les disques disponibles et notez le nom du périphérique (ex : `/dev/sda`, `/dev/vda`, `/dev/nvme0n1`) :
```bash
fdisk -l | grep -E '^(Disque|Disque) '
```

# ▶️ 4. Définissez les variables utilisées dans le tutoriel :
⚠️ **IMPORTANT :**

1. Définir les appareils **AVANT** utilisation :
> 1. **Nous supposerons** pour le tutoriel `/dev/sda` (normal) ou `/dev/nvme0n1` (nvme)
> 2. **Ajustez** en fonction de votre disque (choisissez juste **un** ou **un autre** modèle)

Pour les disques **normaux** (par exemple /dev/sda)
```bash
exporter DEVICE=/dev/sda
exporter DEV_BIOS=${DEVICE}1
export DEV_EFI=${DEVICE}2
exporter DEV_ROOT=${DEVICE}3
exporter DEV_LUKS=/dev/mapper/cryptroot
```
Pour les disques **NVMe** (par exemple /dev/nvme0n1), le suffixe de partition change (`p`)
```bash
exporter DEVICE=/dev/nvme0n1
exporter DEV_BIOS=${DEVICE}p1
export DEV_EFI=${DEVICE}p2
export DEV_RAIZ=${DEVICE}p3
exporter DEV_LUKS=/dev/mapper/cryptroot
```

> 📌 **Remarque :**
> APPAREIL → disque entier
DEV_BIOS → Partition de démarrage du BIOS (1 à 2 Mio, pas de FS, ne se monte pas)
DEV_EFI → Partition EFI (FAT32)
DEV_ROOT → partition racine (normale ou LUKS)
DEV_LUKS → Mappage LUKS (/dev/mapper/cryptroot)

- 👉 Ici vous définissez l'anatomie du disque. Tout le reste du guide suit simplement ces variables.
- 🔎Pourquoi est-ce nécessaire ?
Parce que tout déclarer au début rend le processus suivant à l’épreuve des fautes de frappe.

2. Définissez **KEYMAP** et **TIMEZONE** (modifiez-les si nécessaire) :
```bash
exporter KEYMAP=br-abnt2
```
```bash
export TIMEZONE=Amérique/Sao_Paulo
```

---

# ▶️ 5. Partitionner le disque
> - La partition BIOS **DOIT** être la première. Cela augmente la compatibilité avec les anciennes cartes mères, les chargeurs de démarrage problématiques et les BIOS qui attendent du code de démarrage dans les premières zones du disque.
> - L'ESP peut venir plus tard sans aucun problème — l'UEFI ne se soucie pas de la position.

### Ordre idéal et correct :

- 1️⃣ Démarrage du BIOS (EF02)
- 2️⃣ ESP (Système EFI, FAT32)
- 3️⃣ Btrfs/Ext4/Xfs/Jfs (racine)

### Partitionner en utilisant Parted (automatique)
> Ici le **DEVICE** est déjà défini là-haut, il n'y a donc pas de variable « magique ».
```
wipefs -a "${DEVICE}"
parted --script "${DEVICE}" -- \
mklabelgpt\
mkpart primaire 1 Mo 2 Mo nom 1 BIOS défini 1 bios_grub sur \
mkpart primaire fat32 2 Mo 514 Mo nom 2 EFI défini 2 esp sur \
mkpart primaire 514 Mo 100 % nom 3 RACINE \
align-check optimal 1 \
aligner-vérifier optimal 2 \
align-check optimal 1
parted --script "${DEVICE}" -- imprimer
```
> - Partition 1 → BIOS boot (bios_grub, pas de FS, ne se monte pas)
> - Partition 2 → EFI (FAT32)
> - Partition 3 → ROOT (nous la formaterons plus tard avec EXT4/XFS/JFS/BTRFS, avec ou sans LUKS)
> - J'ai utilisé mkpart Primary 514MiB à 100% sans spécifier précisément FS pour éviter de bloquer le FS. Vous choisissez FS plus tard.
---

# ▶️ 6. Choisissez le mode d'installation (NORMAL ou LUKS)
⚠️ **IMPORTANT :**
> Choisissez UNIQUEMENT UN des deux blocs ci-dessous.
**PAS** pour exécuter les deux étapes.

1. INSTALLATION NORMALE **(sans LUKS)**
```bash
exporter DISK="${DEV_RAIZ}"
```
- Définit DISK sur le périphérique réel /dev/sda3

2. INSTALLATION **AVEC LUKS** (racine cryptée)
```
# Chiffrez UNIQUEMENT la partition racine sur LUKS1 (compatible GRUB) - jamais le disque entier
# Chiffrez la partition en confirmant par OUI :
cryptsetup luksFormat --type luks1 "${DEV_RAIZ}"

# Ouvrez la partition avec votre phrase secrète.
cryptsetup ouvre la racine cryptée "${DEV_RAIZ}"

# Désormais, la vraie racine est le périphérique mappé
exporter DISK="${DEV_LUKS}"
```
- LUKS se trouve au-dessus de /dev/sda3, pas sur tout le disque
- Le système sera installé dans /dev/mapper/cryptroot

👉 A partir de maintenant, TOUT utilise $DISK.

---

# ▶️ 7. Créez le système de fichiers (FS) et montez root
⚠️ **IMPORTANT :**
> Choisissez UNIQUEMENT UN des deux blocs ci-dessous.

1. **EXT4**
```
mkfs.ext4 -F "${DISK}" -L ROOT
mount -v "${DISK}" /mnt
```
2. **XFS**
```
mkfs.xfs -f "${DISK}"
mount -v "${DISK}" /mnt
```
3. **JFS**
```
mkfs.jfs -f "${DISK}"
mount -v "${DISK}" /mnt
```
4. **BTRFS simples**
```
mkfs.btrfs -f "${DISK}" -L RACINE
mount -v "${DISK}" /mnt
```
5. **BTRFS avec sous-volumes**
```
mkfs.btrfs -f "${DISK}" -L RACINE

monter ${DISK} /mnt
création du sous-volume btrfs /mnt/@
création du sous-volume btrfs /mnt/@home
création du sous-volume btrfs /mnt/@log
création du sous-volume btrfs /mnt/@cache
sous-volume btrfs créer /mnt/@snapshots
montant /mnt

mount -o defaults,noatime,ssd,compress=zstd:3,discard=async,space_cache=v2,commit=300,subvol=/@ ${DISK} /mnt
mkdir -p /mnt/{boot/efi,home,var/log,var/cache,.snapshots,swap}

mount -o defaults,noatime,ssd,compress=zstd:3,discard=async,space_cache=v2,commit=300,subvol=/@home ${DISK} /mnt/home
mount -o defaults,noatime,ssd,compress=zstd:3,discard=async,space_cache=v2,commit=300,subvol=/@cache ${DISK} /mnt/var/cache
mount -o defaults,noatime,ssd,compress=zstd:3,discard=async,space_cache=v2,commit=300,subvol=/@log ${DISK} /mnt/var/log
mount -o defaults,noatime,ssd,compress=zstd:3,discard=async,space_cache=v2,commit=300,subvol=/@snapshots ${DISK} /mnt/.snapshots
```
---

# ▶️ 8. Préparer et assembler l'ESP (EFI)
```
mkfs.fat -F32 -I "${DEV_EFI}"
mkdir -p /mnt/boot/efi
mount -v "${DEV_EFI}" /mnt/boot/efi
```
>💡 La partition BIOS (${DEV_BIOS}) n'a pas de système de fichiers, ne se formate pas, ne se monte pas.
---

# ▶️ 9. Installer ou Void Linux sans chroot

1. Copiez les clés du référentiel (clés XBPS) à utiliser dans le chroot (/mnt)
```
mkdir -p /A{tc, vapor/xbps/xbps)
cp -rpaf /var/db/xbps/keys/*.plist /mnt/var/db/xbps/keys/
cp -fpa /etc/resolv.conf /mnt/etc/resolv.conf
```

2. Installez le système de base sur le disque nouvellement monté :
```
xbps-install -Sy -R https://repo-default.voidlinux.org/current \
-r /mnt\
système de base btrfs-progs cryptsetup grub grub-x86_64-efi dracut linux \
linux-headers linux-firmware linux-firmware-network glibc-locales \
xtools dhcpcd openssh vim nano grc zstd xz bash-completion vpm vsv \
socklog-void wget net-tools tmate ncurses jfsutils xfsprogs duf arbre eza chrony
```
---

# ▶️ 10. Utiliser le fstab no /mnt (chroot)
```bash
# Générer fstab dans /mnt/etc/fstab
xgenfstab -U /mnt > /mnt/etc/fstab
```

```bash
# vérifie s'il a été généré correctement
chat /mnt/etc/fstab
```

# ▶️ 11. Accédez au système installé en utilisant chroot

1. Employer ne croit pas :
```
xchroot /mnt /bin/bash
```
---

# ▶️ 12. Paramètres initiaux (dans le chroot)
```
# configure le nom d'hôte - définit le nom d'hôte
echo void > /etc/nom d'hôte

# configurer le fuseau horaire - définit le fuseau horaire
ln -sfv /usr/share/zoneinfo/"${TIMEZONE}" /etc/localtime

# configurer les paramètres régionaux
-i -e 's/^#\(en_.Utf-8 UTF-8\)/)/' \
-E 's/^#\pt_br.br.utf-8 UTF-8\)/' \' \
/etc/default/libc-locales

# générer des paramètres régionaux
xbps-reconfigure -f glibc-locales

# Correction d'une erreur possible dans le lien symbolique /var/service (important) :
rm -f /var/service
ln -sf /etc/runit/runsvdir/default /var/service

# Activer certains services
ln -sf /etc/sv/dbus /var/service/
ln -sf /etc/sv/dhcpcd /var/service/
ln -sf /etc/sv/sshd /var/service/
ln -sf /etc/sv/nanoklogd /var/service/
ln -sf /etc/sv/socklog-unix /var/service/
ln -sf /etc/sv/chronyd /var/service/

# Configurer sudo - wheel group (facultatif, mais recommandé)
chat << 'EOF' > /etc/sudoers.d/g_wheel
%wheel ALL=(TOUS:TOUS) NOPASSWD: TOUS
EOF
#Autorisations requises
chmod 440 /etc/sudoers.d/g_wheel
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
---

# ▶️ 13. Configurer les UUID
⚠️ **IMPORTANT :**
- Récupérez les UUID des partitions :
```
UUID_LUKS=$(blkid -s UUID -o valeur "${DEV_RAIZ}")
UUID_ROOT=$(blkid -s UUID -o valeur "${DISK}")
UUID_EFI=$(blkid -s UUID -o valeur "${DEV_EFI}")
```
---

# ▶️ 14. Créez un fichier d'échange avec prise en charge de l'hibernation (facultatif)

### Remarques importantes
```
- Swapfile dans Btrfs apparaît toujours comme **prealloc**, c'est normal.
- Il n'est pas nécessaire qu'il s'agisse de la taille totale de la RAM.
- 60% suffisent pour l'hibernation dans la plupart des cas.
- Pour les charges lourdes → utiliser 70 % ou 80 %.
```

1. Calculer automatiquement la taille optimale du fichier d'échange
```
# Recommandation moderne pour l'hibernation : 60 % de la RAM totale
SWAP_GB=$(LC_ALL=C awk '/MemTotal/ {print int($2 * 0.60 / 1024 / 1024)}' /proc/meminfo)
echo "Fichier d'échange recommandé : ${SWAP_GB}G"
```
- soit, définissez manuellement la taille souhaitée :
```
SWAP_GB=4
echo "Fichier d'échange défini par l'utilisateur : ${SWAP_GB}G"
```
2. Créez un répertoire pour le fichier d'échange
```
mkdir -p /échange
swapoff -a 2>/dev/null
rm -f /swap/fichier d'échange
```
3. Désactiver COW (obligatoire dans Btrfs)
```
chattr +C /échange
```
4. Créez le fichier d'échange avec la taille précédemment définie
```
fallocate -l ${SWAP_GB}G /swap/swapfile
chmod 600 / échange / fichier d'échange
```
5. Formatez le fichier d'échange et activez le swap
```
mkswap /swap/fichier d'échange
swapon /swap/fichier d'échange
```
6. Obtenez un décalage :
```
# Installez le package pour filefrag
xbps-install -Sy e2fsprogs

# Obtenez le décalage
offset=$(filefrag -v /swap/swapfile | awk '/^ *0:/{print $4}')
```
---

# ▶️ 15. Configurer GRUB
⚠️ **IMPORTANT :**
> Ce bloc est intelligent :
- Detecta automaticamente se você está usando LUKS
- Détecte si vous avez créé un fichier d'échange avec mise en veille prolongée
- Ajuste /etc/default/grub sans rien dupliquer
- Crée les lignes nécessaires uniquement si elles manquent
- Ne changez rien si ce n'est pas nécessaire

Utilisez exactement le bloc ci-dessous :
```
HAS_RESUME=faux
HAS_LUKS=faux

[[ -n "${offset}" ]] && HAS_RESUME=true
[[ "${DISK}" = "${DEV_LUKS}" ]] && HAS_LUKS=true

# Supprimez l'ancienne ligne pour des raisons de sécurité
sed -i '/^[[:space:]]*GRUB_CMDLINE_LINUX_DEFAULT=/d' /etc/default/grub

# GRUB_CMDLINE_LINUX

# Valeur de base
BASE="niveau de journal=4"

# Ajouter un résumé
si $HAS_RESUME ; alors
BASE="$BASE CV=UUID=${UUID_ROOT} CV_offset=${offset}"
être

# Ajouter des LUKS
si $HAS_LUKS ; alors
grep -q '^GRUB_ENABLE_CRYPTODISK=y' /etc/default/grub || echo 'GRUB_ENABLE_CRYPTODISK=y' >> /etc/default/grub
grep -q '^GRUB_PRELOAD_MODULES=' /etc/default/grub || echo 'GRUB_PRELOAD_MODULES="luks cryptodisk gcry_rijndael"' >> /etc/default/grub
BASE="$BASE rd.luks.uuid=${UUID_LUKS} rd.luks.name=${UUID_LUKS}=cryptroot root=/dev/mapper/cryptroot"
être

# Recréez correctement la dernière ligne
echo "GRUB_CMDLINE_LINUX_DEFAULT=\"${BASE}\"" >> /etc/default/grub
```
---

# ▶️ 16. Recréez l'initrd
⚠️ **IMPORTANT :**
```
mods=(/usr/lib/modules/*)
KVER=$(nom de base "${mods[0]}")
écho ${KVER}
dracut --force --kver ${KVER}
```
---

# ▶️ 17. Créez un fichier clé pour éviter de demander deux fois le mot de passe au démarrage (LUKS uniquement)
> Si le système N'utilise PAS LUKS, ignorez cette étape.
```
si [ "${DISK}" = "${DEV_LUKS}" ]; alors
echo "LUKS détecté : création d'un fichier de clés pour le déverrouillage automatique..."

# Créer un fichier de clés sécurisé
dd if=/dev/urandom of=/boot/volume.key bs=64 count=1
chmod 000 /boot/volume.key

# Ajoutez un fichier de clés à LUKS (vous demandera votre mot de passe actuel)
cryptsetup luksAddKey "${DEV_RAIZ}" /boot/volume.key

# Configurer /etc/crypttab
chat << EOF >> /etc/crypttab
cryptroot ${DEV_RAIZ} /boot/volume.key luks
EOF

# Inclure le fichier de clés et le crypttab dans initramfs
mkdir -p /etc/dracut.conf.d
chat << EOF >> /etc/dracut.conf.d/10-crypt.conf
install_items+=" /boot/volume.key /etc/crypttab "
EOF

# Régénérer les initramfs avec la prise en charge des fichiers clés
xbps-reconfigure -fa
autre
echo "Système sans LUKS : ignorer la création du fichier de clés."
être
```

# ▶️ 18. Installez GRUB dans **BIOS** et **UEFI** (véritable hybride)
1. Installez GRUB pour le BIOS (ancien)
```
grub-install --target=i386-pc ${DEVICE}
```
2. Installez GRUB pour UEFI
```
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=void
```
3. Créez une solution de secours UEFI (démarrage universel). Ce fichier garantit le démarrage même lorsque la NVRAM est effacée.
```
mkdir -p /boot/efi/EFI/BOOT
cp -f /boot/efi/EFI/void/grubx64.efi /boot/efi/EFI/BOOT/BOOTX64.EFI
```
4. Générer le fichier GRUB final
```
grub-mkconfig -o /boot/grub/grub.cfg
```

---

# ▶️ 19. Paramètres utilisateur personnalisés :

1. Paramètres d'environnement :

```
# Personnaliser /etc/xbps.d/00-repository-main.conf
mkdir -pv /etc/xbps.d
cat << 'EOF' >> /etc/xbps.d/00-repository-main.conf
référentiel=https://repo-fastly.voidlinux.org/current
#dépôt=https://repo-fastly.voidlinux.org/current/nonfree
#dépôt=https://repo-fastly.voidlinux.org/current/multilib
#dépôt=https://repo-fastly.voidlinux.org/current/multilib/nonfree

référentiel=https://void.chililinux.com/voidlinux/current
#dépôt=https://void.chililinux.com/voidlinux/current/extras
#dépôt=https://void.chililinux.com/voidlinux/current/nonfree
#dépôt=https://void.chililinux.com/voidlinux/current/multilib
#dépôt=https://void.chililinux.com/voidlinux/current/multilib/nonfree
EOF

# Personnalisez /etc/rc.conf. Définit le fuseau horaire, la disposition du clavier et la police par défaut de la console. Changez si nécessaire.
chat << EOF >> /etc/rc.conf
ZONE HORAIRE="${TIMEZONE}"
KEYMAP="${KEYMAP}"
POLICE=Lat2-Terminus16
EOF

# Personnaliser le .bashrc de root
wget --quiet --no-check-certificate \
-O /etc//skel/.bashrc \
"https://raw.githubusercontent.com/voidlinux-br/void-install/refs/heads/main/.bashrc"
racine chown :root /etc/skel/.bashrc
chmod 644 /etc/skel/.bashrc

cat << 'EOF' > /etc/skel/.bash_profile
# ~/.bash_profile — charge .bashrc dans Void

# Si .bashrc existe, chargez
si [ -f ~/.bashrc ]; alors
source ~/.bashrc
être
EOF

# copie à la racine et à l'utilisateur
pour d dans /root "/home/${NEWUSER}" ; faire
cp -f /etc/skel/.bash_profile "$d/"
cp -f /etc/skel/.bashrc "$d/"
fait

chown "${NEWUSER}:${NEWUSER}" "/home/${NEWUSER}/.bash_profile" "/home/${NEWUSER}/.bashrc"
chmod 644 "/home/${NEWUSER}/.bash_profile" "/home/${NEWUSER}/.bashrc"

# télécharger le svlogtail personnalisé
wget --quiet --no-check-certificate \
-O /usr/bin/svlogtail\
"https://raw.githubusercontent.com/voidlinux-br/void-install/refs/heads/main/svlogtail"
chmod +x /usr/bin/svlogtail
```

2. Configurez ssh (facultatif, mais recommandé) :
```
mkdir -pv /etc/ssh/sshd_config.d/
cat << 'EOF' > /etc/ssh/sshd_config.d/10-custom.conf
PermisTTY oui
PrintMotd oui
PrintLastLog oui
Bannière /etc/issue.net

PermitRootLogin oui
KbdInteractiveAuthentication oui
X11Transfert oui
PubkeyAuthentification oui
PubkeyAcceptedKeyTypes=+ssh-rsa
AuthorizedKeysFile .ssh/authorized_keys
Authentification par mot de passe oui
ChallengeResponseAuthentication oui
Utiliser PAM oui

Sous-système sftp interne-sftp
EOF
```
---

# ▶️ 20. Activer ZRAM (facultatif)
Void Linux utilise le service zramen pour activer ZRAM, créant un bloc de mémoire compressée qui réduit l'utilisation du swap SSD et améliore les performances sous charge.
1. Installez Zramen
```
xbps-install -Sy zramen
```
2. Configurez ZRAM (configuration recommandée) :
```
chat << 'EOF' > /etc/zramen.conf
zram_fraction = 0,5
zram_devices=1
zram_algorithm=zstd
EOF
```
3. Activez le service dans runit
```
ln -s
```
> ZRAM sera automatiquement activé à chaque démarrage

---

# ▶️    21. Finalizar instalação
1. Comment faire du chroot :
```
sortie
```
2. Démontez toutes les partitions montées sur /mnt (sous-volumes et /boot/efi) :
```
umount -R /mnt
```
3. Désactivez tout fichier d'échange ou partition d'échange activé dans le chroot :
```
échange -a
```
4. Redémarrez la machine physique ou la VM pour tester le démarrage réel :
```
redémarrer
```
> N'oubliez pas de retirer le support d'installation et de démarrer à partir du disque nouvellement installé.
Apprécier!

---

# 🎉 SYSTÈME COMPLET, HYBRIDE ET À L'ÉPREUVE DU FUTUR
- Démarrez le BIOS + UEFI
- UEFI de secours
- Btrfs avec instantanés (prêts pour Snapper/Timeshift)
- Véritable hibernation avec swapfile
- Zram pour les performances

Ce SSD démarre **n'importe quelle machine de la planète**.

# CLAUSE DE NON-RESPONSABILITÉ

```
Ce tutoriel est gratuit : vous pouvez utiliser, copier, modifier et redistribuer à votre guise.
Le contenu est mis à disposition sous la **licence MIT** et peut inclure des extraits ou des commandes dérivés de logiciels open source soumis à ses propres licences.

Aucune garantie n'est fournie — tout ici est livré « tel quel ».
Utilisez à vos propres risques. Ni l'auteur, ni les contributeurs, ni Void Linux ne sont responsables des pertes, dommages, pannes du système ou de toute conséquence de l'utilisation de ce matériel.

Si vous le souhaitez, vous pouvez obtenir le code source, réviser, adapter et générer votre propre version de ce didacticiel.
```

