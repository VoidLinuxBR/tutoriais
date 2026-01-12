# 🧩 VOID LINUX TUTORIAL – HYBRID-INSTALLATION (UEFI + BIOS) MIT EXT4, XFS, JFS ODER BTRFS (SUBVOLUMES), LUKS, HIBERNATION UND ZRAM
### ÜBERARBEITETE UND VALIDIERTE VERSION – KORREKTE PARTIONIERUNG + UNIVERSAL-BOOT

Diese Anleitung installiert ein vollständig **hybrides** Void Linux, das in der Lage ist, jeden Maschinentyp zu booten – ob alt, neu oder problematisch:

- 💾 **Modernes UEFI** (mit normaler Eingabe und Fallback)
- 🧮 **BIOS/Legacy** (vollständige Kompatibilität)
- 🧰 **GPT mit BIOS-Boot (EF02)** – maximale Unterstützung für alte Hardware
- 🚀 **Btrfs mit Subvolumes** (optional), vorgefertigte Snapshots
- 🔐 **LUKS1 voll kompatibel mit GRUB**
- 🌙 **Echter Ruhezustand per Auslagerungsdatei**
- 🧊 **ZRAM für Leistung konfiguriert**
- 🧱 **Volle Unterstützung für EXT4, XFS, JFS und BTRFS**
- 💡 **Initramfs/GRUB automatisch konfiguriert (LUKS + Lebenslauf)**

📌 **Keine Kompromisse, keine Neuinstallation von GRUB, keine Zeitverschwendung.**
📌 **Garantierter Start auch auf einer Maschine mit gelöschtem NVRAM (BOOTX64.EFI-Fallback).**

---

# ▶️ 1. Booten Sie die Live-ISO

Vorschlag: Verwenden Sie die glibc-Version für bessere Kompatibilität:
- Laden Sie die ISO herunter von:
```
https://repo-default.voidlinux.org/live/current/void-live-x86_64-20250202-base.iso
```
- oder suchen Sie nach der neuesten Version unter:
```
https://voidlinux.org/download/
```

1. Melden Sie sich als Root an.
```bash
Login: root
Passwort: voidlinux
```

2. Wechseln Sie die Shell von *sh* zu *bash*.
*dash/sh* **Implementiert NICHT** mehrere Funktionen, die viele Skripte verwenden.
```bash
bash
```

3. Ändern Sie das Tastaturlayout in **ABNT2** und achten Sie dabei auf die korrekte Zuordnung von Akzenten und Symbolen:
```bash
Ladeschlüssel br-abnt2
```

4. In das Terminal einfügen (optional) – Eingabeaufforderung mit Farben, Benutzer@Host:Pfad und Status des letzten Befehls ( ✔/✘). Nützlich und schön.
```bash
export PS1='\[\e[1;32m\]\u\[\e[1;33m\]@\[\e[1;36m\]\h\[\e[1;31m\]:\w \
$([[ $? -eq 0 ]] && echo -e "\e[1;32m✔" || echo -e "\e[1;31m✘$?") \
\[\e[0m\]\$ '
```

# ▶️ 2. Stellen Sie eine Verbindung zum Internet her
- Für **WLAN** *(bei Kabelverbindung diesen Schritt überspringen)*:
```bash
wpa_passphrase „SSID“ „PASSWORD“ > wifi.conf
wpa_supplicant -B -i wlan0 -c wifi.conf
dhcpcd wlan0
```

1. Testen Sie die Verbindung:
```bash
ping -c3 8.8.8.8
ping -c3 repo-default.voidlinux.org
```

2. Installieren Sie die erforderlichen Pakete:
⚠️ **WICHTIG:**
```bash
xbps-install -Sy xbps parted jfsutils xfsprogs nano zstd xz bash-completion
```
---

# ▶️ 3. Identifizieren Sie die Festplatte
1. Listen Sie die verfügbaren Festplatten auf und notieren Sie sich den Gerätenamen (z. B. „/dev/sda“, „/dev/vda“, „/dev/nvme0n1“):
```bash
fdisk -l | grep -E '^(Disk|Disk) '
```

# ▶️ 4. Definieren Sie die im Tutorial verwendeten Variablen:
⚠️ **WICHTIG:**

1. Definieren Sie Geräte **VOR** der Verwendung:
> 1. **Wir gehen davon aus** für das Tutorial „/dev/sda“ (normal) oder „/dev/nvme0n1“ (nvme)
> 2. **Anpassen** entsprechend Ihrer Disc (wählen Sie einfach **ein** oder **ein anderes** Modell)

Für **normale** Festplatten (z. B. /dev/sda)
```bash
export DEVICE=/dev/sda
exportieren Sie DEV_BIOS=${DEVICE}1
exportieren DEV_EFI=${DEVICE}2
export DEV_ROOT=${DEVICE}3
export DEV_LUKS=/dev/mapper/cryptroot
```
Bei **NVMe**-Festplatten (z. B. /dev/nvme0n1) ändert sich das Partitionssuffix („p“).
```bash
export DEVICE=/dev/nvme0n1
export DEV_BIOS=${DEVICE}p1
export DEV_EFI=${DEVICE}p2
export DEV_RAIZ=${DEVICE}p3
export DEV_LUKS=/dev/mapper/cryptroot
```

> 📌 **Hinweis:**
> GERÄT → gesamte Festplatte
DEV_BIOS → BIOS-Boot-Partition (1–2 MiB, kein FS, wird nicht gemountet)
DEV_EFI → EFI-Partition (FAT32)
DEV_ROOT → Root-Partition (normal oder LUKS)
DEV_LUKS → LUKS-Zuordnung (/dev/mapper/cryptroot)

- 👉 Hier definieren Sie die Anatomie der Bandscheibe. Alles andere im Leitfaden folgt einfach diesen Variablen.
- 🔎 Warum ist das notwendig?
Denn alles am Anfang zu deklarieren, macht den nächsten Prozess fehlersicher.

2. Definieren Sie **KEYMAP** und **TIMEZONE** (ändern Sie diese nach Bedarf):
```bash
export KEYMAP=br-abnt2
```
```bash
export TIMEZONE=Amerika/Sao_Paulo
```

---

# ▶️ 5. Partitionieren Sie die Festplatte
> - Die BIOS-Partition **MUSS** die erste sein. Dies erhöht die Kompatibilität mit älteren Motherboards, problematischen Bootloadern und BIOSen, die Bootcode in den ersten Bereichen der Festplatte erwarten.
> - ESP kann problemlos später kommen – UEFI ist die Position egal.

### Ideale und richtige Reihenfolge:

- 1️⃣ BIOS-Boot (EF02)
- 2️⃣ ESP (EFI-System, FAT32)
- 3️⃣ Btrfs/Ext4/Xfs/Jfs (root)

### Partitionierung mit parted (automatisch)
> Hier ist das **GERÄT** bereits oben definiert, es gibt also keine „magische“ Variable.
```
wipefs -a "${DEVICE}"
parted --script "${DEVICE}" -- \
mklabel gpt \
mkpart primär 1MiB 2MiB Name 1 BIOS-Set 1 bios_grub auf \
mkpart Primary Fat32 2MiB 514MiB Name 2 EFI Set 2 esp on \
mkpart primär 514MiB 100 % Name 3 ROOT \
Align-Check optimal 1 \
Align-Check optimal 2 \
Align-Check optimal 1
parted --script "${DEVICE}" -- drucken
```
> - Partition 1 → BIOS-Boot (bios_grub, kein FS, wird nicht gemountet)
> - Partition 2 → EFI (FAT32)
> - Partition 3 → ROOT (wir werden es später mit EXT4/XFS/JFS/BTRFS formatieren, mit oder ohne LUKS)
> - Ich habe mkpart Primary 514MiB 100 % verwendet, ohne FS genau anzugeben, um eine Bindung des FS zu vermeiden. Sie wählen später FS.
---

# ▶️ 6. Wählen Sie den Installationsmodus (NORMAL oder LUKS)
⚠️ **WICHTIG:**
> Wählen Sie NUR EINEN der beiden folgenden Blöcke.
**NICHT**, beide Schritte auszuführen.

1. NORMALINSTALLATION **(ohne LUKS)**
```bash
export DISK="${DEV_RAIZ}"
```
– Setzt DISK auf das tatsächliche Gerät /dev/sda3

2. INSTALLATION **MIT LUKS** (verschlüsseltes Root)
```
# Verschlüsseln Sie NUR die Root-Partition auf LUKS1 (GRUB-kompatibel) – niemals die gesamte Festplatte
# Verschlüsseln Sie die Partition, indem Sie mit JA bestätigen:
cryptsetup luksFormat --type luks1 "${DEV_RAIZ}"

# Öffnen Sie die Partition mit Ihrer Passphrase.
cryptsetup öffnet die Kryptowurzel „${DEV_RAIZ}“.

# Von nun an ist das zugeordnete Gerät der eigentliche Root
export DISK="${DEV_LUKS}"
```
- LUKS befindet sich auf /dev/sda3, nicht auf der gesamten Festplatte
- Das System wird in /dev/mapper/cryptroot installiert

👉 Von hier an verwendet ALLES $DISK.

---

# ▶️ 7. Erstellen Sie das Dateisystem (FS) und mounten Sie Root
⚠️ **WICHTIG:**
> Wählen Sie NUR EINEN der beiden folgenden Blöcke.

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
4. **Einfaches BTRFS**
```
mkfs.btrfs -f "${DISK}" -L ROOT
mount -v "${DISK}" /mnt
```
5. **BTRFS mit Subvolumes**
```
mkfs.btrfs -f "${DISK}" -L ROOT

Mount ${DISK} /mnt
Btrfs-Subvolume erstellen /mnt/@
Btrfs-Subvolume erstellen /mnt/@home
btrfs subvolume create /mnt/@log
Btrfs-Subvolume erstellen /mnt/@cache
Btrfs-Subvolume erstellen /mnt/@snapshots
umount /mnt

mount -o defaults,noatime,ssd,compress=zstd:3,discard=async,space_cache=v2,commit=300,subvol=/@ ${DISK} /mnt
mkdir -p /mnt/{boot/efi,home,var/log,var/cache,.snapshots,swap}

mount -o defaults,noatime,ssd,compress=zstd:3,discard=async,space_cache=v2,commit=300,subvol=/@home ${DISK} /mnt/home
mount -o defaults,noatime,ssd,compress=zstd:3,discard=async,space_cache=v2,commit=300,subvol=/@cache ${DISK} /mnt/var/cache
mount -o defaults,noatime,ssd,compress=zstd:3,discard=async,space_cache=v2,commit=300,subvol=/@log ${DISK} /mnt/var/log
mount -o defaults,noatime,ssd,compress=zstd:3,discard=async,space_cache=v2,commit=300,subvol=/@snapshots ${DISK} /mnt/.snapshots
```
---

# ▶️ 8. ESP (EFI) vorbereiten und zusammenbauen
```
mkfs.fat -F32 -I "${DEV_EFI}"
mkdir -p /mnt/boot/efi
mount -v "${DEV_EFI}" /mnt/boot/efi
```
>💡 Die BIOS-Partition (${DEV_BIOS}) hat kein Dateisystem, formatiert nicht und wird nicht gemountet.
---

# ▶️ 9. Instalar o Void Linux no chroot

1. Kopieren Sie die Repository-Schlüssel (XBPS-Schlüssel), die in der Chroot verwendet werden sollen (/mnt).
```
mkdir -p /A{tc, Vapor/xbps/xbps)
cp -rpaf /var/db/xbps/keys/*.plist /mnt/var/db/xbps/keys/
cp -fpa /etc/resolv.conf /mnt/etc/resolv.conf
```

2. Installieren Sie das Basissystem auf der neu gemounteten Festplatte:
```
xbps-install -Sy -R https://repo-default.voidlinux.org/current \
-r /mnt \
Basissystem Btrfs-Progs Cryptsetup Grub Grub-x86_64-EFI Dracut Linux \
linux-headers linux-firmware linux-firmware-network glibc-locales \
xtools dhcpcd openssh vim nano grc zstd xz bash-completion vpm vsv \
socklog-void wget net-tools tmate ncurses jfsutils xfsprogs duf tree eza chrony
```
---

# ▶️ 10. Gerar fstab no /mnt (chroot)
```bash
# Fstab in /mnt/etc/fstab generieren
xgenfstab -U /mnt > /mnt/etc/fstab
```

```bash
# Überprüfen Sie, ob es korrekt generiert wurde
cat /mnt/etc/fstab
```

# ▶️ 11. Greifen Sie mit Chroot auf das installierte System zu

1. Beschäftigung statt Croit:
```
xchroot /mnt /bin/bash
```
---

# ▶️ 12. Anfangseinstellungen (in Chroot)
```
# hostname konfigurieren – definiert den Hostnamen
echo void > /etc/hostname

# Zeitzone konfigurieren – definiert die Zeitzone
ln -sfv /usr/share/zoneinfo/"${TIMEZONE}" /etc/localtime

# Gebietsschemas konfigurieren
-i -e 's/^#\(en_.Utf-8 UTF-8\)/)/' \
-E 's/^#\pt_br.br.utf-8 UTF-8\)/' \' \
/etc/default/libc-locales

# Gebietsschemas generieren
xbps-reconfigure -f glibc-locales

# Möglichen Fehler im /var/service-Symlink beheben (wichtig):
rm -f /var/service
ln -sf /etc/runit/runsvdir/default /var/service

# Aktivieren Sie einige Dienste
ln -sf /etc/sv/dbus /var/service/
ln -sf /etc/sv/dhcpcd /var/service/
ln -sf /etc/sv/sshd /var/service/
ln -sf /etc/sv/nanoklogd /var/service/
ln -sf /etc/sv/socklog-unix /var/service/
ln -sf /etc/sv/chronyd /var/service/

# Sudo-Radgruppe konfigurieren (optional, aber empfohlen)
cat << 'EOF' > /etc/sudoers.d/g_wheel
%wheel ALL=(ALL:ALL) NOPASSWD: ALLE
EOF
#Erforderliche Berechtigungen
chmod 440 /etc/sudoers.d/g_wheel
```

## Benutzer erstellen und konfigurieren

⚠️ **WICHTIG:** Definieren Sie unten den tatsächlichen Benutzernamen.
```bash
export NEWUSER=your_user_here
```

Erstellen Sie den Benutzer mit Home-Verzeichnis, Basisgruppen und Bash-Shell:
```bash
useradd -m -G audio,video,wheel,tty -s /bin/bash ${NEWUSER}
```

Legen Sie Ihr Benutzerpasswort fest (***WICHTIG***)
```bash
Passwort ${NEWUSER}
```

Root-Benutzerpasswort festlegen (***WICHTIG***)
```bash
Passwort root
```

Ändern Sie die Standard-Shell des Root-Benutzers in Bash
```bash
chsh -s /bin/bash root
```
---

# ▶️ 13. UUIDs konfigurieren
⚠️ **WICHTIG:**
- Holen Sie sich die UUIDs der Partitionen:
```
UUID_LUKS=$(blkid -s UUID -o value "${DEV_RAIZ}")
UUID_ROOT=$(blkid -s UUID -o value "${DISK}")
UUID_EFI=$(blkid -s UUID -o value "${DEV_EFI}")
```
---

# ▶️ 14. Auslagerungsdatei mit Ruhezustandsunterstützung erstellen (optional)

### Wichtige Hinweise
```
– Swapfile in Btrfs erscheint immer als **prealloc**, das ist normal.
- Es muss nicht die volle RAM-Größe vorhanden sein.
- 60 % reichen in den meisten Fällen für den Winterschlaf aus.
- Für schwere Lasten → 70 % oder 80 % verwenden.
```

1. Berechnen Sie automatisch die optimale Größe der Auslagerungsdatei
```
# Moderne Empfehlung für den Ruhezustand: 60 % des gesamten RAM
SWAP_GB=$(LC_ALL=C awk '/MemTotal/ {print int($2 * 0.60 / 1024 / 1024)}' /proc/meminfo)
echo „Empfohlene Auslagerungsdatei: ${SWAP_GB}G“
```
- oder stellen Sie die gewünschte Größe manuell ein:
```
SWAP_GB=4
echo „Benutzerdefinierte Auslagerungsdatei: ${SWAP_GB}G“
```
2. Erstellen Sie ein Verzeichnis für die Auslagerungsdatei
```
mkdir -p /swap
swapoff -a 2>/dev/null
rm -f /swap/swapfile
```
3. COW deaktivieren (erforderlich in Btrfs)
```
chattr +C /swap
```
4. Erstellen Sie die Auslagerungsdatei mit der zuvor definierten Größe
```
fallocate -l ${SWAP_GB}G /swap/swapfile
chmod 600 /swap/swapfile
```
5. Formatieren Sie die Auslagerungsdatei und aktivieren Sie den Auslagerungsvorgang
```
mkswap /swap/swapfile
swapon /swap/swapfile
```
6. Offset abrufen:
```
# Installieren Sie das Paket für Filefrag
xbps-install -Sy e2fsprogs

# Ermitteln Sie den Offset
offset=$(filefrag -v /swap/swapfile | awk '/^ *0:/{print $4}')
```
---

# ▶️ 15. GRUB konfigurieren
⚠️ **WICHTIG:**
> Dieser Block ist smart:
- Erkennt automatisch, ob Sie LUKS verwenden
– Erkennt, ob Sie eine Auslagerungsdatei im Ruhezustand erstellt haben
- Passt /etc/default/grub an, ohne etwas zu duplizieren
- Erstellt die erforderlichen Zeilen nur, wenn diese fehlen
- Ändern Sie nichts, wenn es nicht notwendig ist

Verwenden Sie genau den folgenden Block:
```
HAS_RESUME=false
HAS_LUKS=false

[[ -n "${offset}" ]] && HAS_RESUME=true
[[ "${DISK}" = "${DEV_LUKS}" ]] && HAS_LUKS=true

# Entfernen Sie aus Sicherheitsgründen die alte Leitung
sed -i '/^[[:space:]]*GRUB_CMDLINE_LINUX_DEFAULT=/d' /etc/default/grub

# GRUB_CMDLINE_LINUX

# Basiswert
BASE="loglevel=4"

# Zusammenfassung hinzufügen
wenn $HAS_RESUME; Dann
BASE="$BASE resume=UUID=${UUID_ROOT} resume_offset=${offset}"
Sei

# LUKS hinzufügen
wenn $HAS_LUKS; Dann
grep -q '^GRUB_ENABLE_CRYPTODISK=y' /etc/default/grub || echo 'GRUB_ENABLE_CRYPTODISK=y' >> /etc/default/grub
grep -q '^GRUB_PRELOAD_MODULES=' /etc/default/grub || echo 'GRUB_PRELOAD_MODULES="luks cryptodisk gcry_rijndael"' >> /etc/default/grub
BASE="$BASE rd.luks.uuid=${UUID_LUKS} rd.luks.name=${UUID_LUKS}=cryptroot root=/dev/mapper/cryptroot"
Sei

# Die letzte Zeile korrekt neu erstellen
echo "GRUB_CMDLINE_LINUX_DEFAULT=\"${BASE}\"" >> /etc/default/grub
```
---

# ▶️ 16. Erstellen Sie die Initrd neu
⚠️ **WICHTIG:**
```
mods=(/usr/lib/modules/*)
KVER=$(basename „${mods[0]}“)
echo ${KVER}
dracut --force --kver ${KVER}
```
---

# ▶️ 17. Erstellen Sie eine Schlüsseldatei, um beim Booten nicht zweimal nach dem Passwort zu fragen (nur LUKS).
> Wenn das System LUKS NICHT verwendet, überspringen Sie diesen Schritt.
```
if [ "${DISK}" = "${DEV_LUKS}" ]; Dann
echo "LUKS detectado: criando keyfile para desbloqueio automático..."

# Erstellen Sie eine sichere Schlüsseldatei
dd if=/dev/urandom of=/boot/volume.key bs=64 count=1
chmod 000 /boot/volume.key

# Schlüsseldatei zu LUKS hinzufügen (fragt nach Ihrem aktuellen Passwort)
cryptsetup luksAddKey „${DEV_RAIZ}“ /boot/volume.key

# /etc/crypttab konfigurieren
cat << EOF >> /etc/crypttab
cryptroot ${DEV_RAIZ} /boot/volume.key luks
EOF

# Schlüsseldatei und Crypttab in initramfs einbinden
mkdir -p /etc/dracut.conf.d
cat << EOF >> /etc/dracut.conf.d/10-crypt.conf
install_items+=" /boot/volume.key /etc/crypttab "
EOF

# Initramfs mit Schlüsseldateiunterstützung neu generieren
xbps-reconfigure -fa
anders
echo „System ohne LUKS: Schlüsseldateierstellung überspringen.“
Sei
```

# ▶️ 18. GRUB im **BIOS** und **UEFI** installieren (echter Hybrid)
1. Installieren Sie GRUB für BIOS (Legacy)
```
grub-install --target=i386-pc ${DEVICE}
```
2. Installieren Sie GRUB für UEFI
```
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=void
```
3. Erstellen Sie einen UEFI-Fallback (Universal Boot). Diese Datei garantiert das Booten auch bei gelöschtem NVRAM.
```
mkdir -p /boot/efi/EFI/BOOT
cp -f /boot/efi/EFI/void/grubx64.efi /boot/efi/EFI/BOOT/BOOTX64.EFI
```
4. Generieren Sie die endgültige GRUB-Datei
```
grub-mkconfig -o /boot/grub/grub.cfg
```

---

# ▶️ 19. Benutzerdefinierte Benutzereinstellungen:

1. Umgebungseinstellungen:

```
# Passen Sie /etc/xbps.d/00-repository-main.conf an
mkdir -pv /etc/xbps.d
cat << 'EOF' >> /etc/xbps.d/00-repository-main.conf
Repository=https://repo-fastly.voidlinux.org/current
#repository=https://repo-fastly.voidlinux.org/current/nonfree
#repository=https://repo-fastly.voidlinux.org/current/multilib
#repository=https://repo-fastly.voidlinux.org/current/multilib/nonfree

Repository=https://void.chililinux.com/voidlinux/current
#repository=https://void.chililinux.com/voidlinux/current/extras
#repository=https://void.chililinux.com/voidlinux/current/nonfree
#repository=https://void.chililinux.com/voidlinux/current/multilib
#repository=https://void.chililinux.com/voidlinux/current/multilib/nonfree
EOF

# Passen Sie /etc/rc.conf an. Legt die Standardzeitzone, das Tastaturlayout und die Schriftart der Konsole fest. Ändern Sie es nach Bedarf.
cat << EOF >> /etc/rc.conf
TIMEZONE="${TIMEZONE}"
KEYMAP="${KEYMAP}"
FONT=Lat2-Terminus16
EOF

# Passen Sie die .bashrc-Datei von root an
wget --quiet --no-check-certificate \
-O /etc//skel/.bashrc \
„https://raw.githubusercontent.com/voidlinux-br/void-install/refs/heads/main/.bashrc"
chown root:root /etc/skel/.bashrc
chmod 644 /etc/skel/.bashrc

cat << 'EOF' > /etc/skel/.bash_profile
# ~/.bash_profile – lädt .bashrc in Void

# Wenn .bashrc vorhanden ist, laden
if [ -f ~/.bashrc ]; Dann
Quelle ~/.bashrc
Sei
EOF

# Nach Root und Benutzer kopieren
für d in /root "/home/${NEWUSER}"; Tun
cp -f /etc/skel/.bash_profile "$d/"
cp -f /etc/skel/.bashrc "$d/"
Erledigt

chown „${NEWUSER}:${NEWUSER}“ „/home/${NEWUSER}/.bash_profile“ „/home/${NEWUSER}/.bashrc“
chmod 644 „/home/${NEWUSER}/.bash_profile“ „/home/${NEWUSER}/.bashrc“

# Benutzerdefiniertes SVlogtail herunterladen
wget --quiet --no-check-certificate \
-O /usr/bin/svlogtail\
„https://raw.githubusercontent.com/voidlinux-br/void-install/refs/heads/main/svlogtail"
chmod +x /usr/bin/svlogtail
```

2. SSH konfigurieren (optional, aber empfohlen):
```
mkdir -pv /etc/ssh/sshd_config.d/
cat << 'EOF' > /etc/ssh/sshd_config.d/10-custom.conf
PermitTTY ja
PrintMotd ja
PrintLastLog ja
Banner /etc/issue.net

PermitRootLogin ja
KbdInteractiveAuthentication ja
X11Forwarding ja
PubkeyAuthentication ja
PubkeyAcceptedKeyTypes=+ssh-rsa
AuthorizedKeysFile .ssh/authorized_keys
PasswordAuthentication ja
ChallengeResponseAuthentication ja
Benutze PAM ja

Subsystem SFTP internal-sftp
EOF
```
---

# ▶️ 20. ZRAM aktivieren (optional)
Void Linux nutzt den zramen-Dienst, um ZRAM zu aktivieren und einen Block komprimierten Speichers zu erstellen, der die SSD-Swap-Nutzung reduziert und die Leistung unter Last verbessert.
1. Installieren Sie Zramen
```
xbps-install -Sy zramen
```
2. ZRAM konfigurieren (empfohlene Konfiguration):
```
cat << 'EOF' > /etc/zramen.conf
zram_fraction=0,5
zram_devices=1
zram_algorithm=zstd
EOF
```
3. Aktivieren Sie den Dienst in runit
```
ln -s
```
> O ZRAM será ativado automaticamente em todos os boots

---

# ▶️ 21. Installation abschließen
1. Sair do chroot:
```
Ausfahrt
```
2. Hängen Sie alle auf /mnt gemounteten Partitionen aus (Subvolumes und /boot/efi):
```
umount -R /mnt
```
3. Deaktivieren Sie alle Swap-Dateien oder Swap-Partitionen, die innerhalb der Chroot aktiviert wurden:
```
swapoff -a
```
4. Starten Sie die physische Maschine oder VM neu, um den tatsächlichen Start zu testen:
```
neu starten
```
> Vergessen Sie nicht, das Installationsmedium zu entfernen und von der neu installierten CD zu booten.
Genießen!

---

# 🎉 KOMPLETTES, HYBRIDES, ZUKUNFTSSICHERES SYSTEM
- Boot BIOS + UEFI
- Fallback-UEFI
- Btrfs mit Snapshots (Snapper/Timeshift-fähig)
- Echter Ruhezustand mit Auslagerungsdatei
- Zram für Leistung

Diese SSD bootet **jede Maschine auf dem Planeten**.

# HAFTUNGSAUSSCHLUSS

```
Dieses Tutorial ist kostenlos: Sie können es nach Belieben verwenden, kopieren, ändern und weiterverbreiten.
Inhalte werden unter der **MIT-Lizenz** zur Verfügung gestellt und können Snippets oder Befehle enthalten, die von Open-Source-Software abgeleitet sind, die ihren eigenen Lizenzen unterliegt.

Es werden keine Garantien übernommen – hier wird alles „wie besehen“ geliefert.
Die Nutzung erfolgt auf eigene Gefahr. Weder der Autor noch die Mitwirkenden noch Void Linux sind verantwortlich für Verluste, Schäden, Systemausfälle oder irgendwelche Folgen aus der Verwendung dieses Materials.

Wenn Sie möchten, können Sie den Quellcode erhalten, ihn überprüfen, anpassen und Ihre eigene Version dieses Tutorials erstellen.
```

