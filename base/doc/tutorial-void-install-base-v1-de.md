# 🔥 Base Void Linux-Installations-Tutorial

# Bevor Sie beginnen

Dieses Tutorial beschreibt eine **manuelle Installation von Void Linux** mit direkter Festplattenpartitionierung, „Chroot“ und expliziter Systemkonfiguration.
Es ist **kein automatischer Installer**.

## ⚠️ Bitte sorgfältig lesen

- Dieses Handbuch **setzt Vertrautheit mit Linux**, Terminal und grundlegenden Systemkonzepten (Festplatten, Partitionen, Boot, Dienste) voraus.
- Mehrere Befehle **löschen Daten dauerhaft** („parted“, „mkfs“, „umount -R“).
- Ein Fehler beim Definieren der Festplatte („/dev/sdX“, „/dev/nvmeX“) kann zu **totalem Datenverlust** führen.
- Lesen Sie **das gesamte Tutorial, bevor Sie einen Befehl ausführen**.

## 🖥️ Empfohlene Umgebung

- **VM (VirtualBox, QEMU, KVM usw.)** zum Testen und Lernen.
- Dedizierte Hardware **keine wichtigen Daten**.
- Laborumgebung oder bewusste Einrichtung.

❌ **Nicht empfohlen** für den direkten Einsatz in der Produktion ohne Anpassungen.

## 🔐 Über Sicherheit

Während des Installationsvorgangs haben einige Einstellungen **Vorrang vor der Praktikabilität** und nicht vor der Sicherheit:
- „Root“-Benutzeranmeldung über SSH kann vorübergehend aktiviert werden.
- Möglicherweise ist die Passwortauthentifizierung aktiv.
- Legacy-Kompatibilität (z. B. „ssh-rsa“) kann zulässig sein.

👉 **Diese Einstellungen müssen nach der Installation überprüft werden**, insbesondere auf Systemen, die dem Netzwerk ausgesetzt sind.

## 🧠 Wichtig zu wissen

- Führen Sie die Befehle **einzeln** aus und überprüfen Sie die Ausgabe.
- Passen Sie Festplattennamen, Netzwerkschnittstellen und Benutzer entsprechend Ihrem System an.
- **Nicht blind kopieren und einfügen**.
- Im Zweifelsfall **halten Sie an** und überprüfen Sie den aktuellen Schritt.

## 🛠️ Im Fehlerfall

Wenn etwas schief geht:
- Nicht blind neu starten.
- Bauen Sie die Trennwände wieder zusammen.
- Melden Sie sich mit „chroot“ wieder am System an.
- Überprüfen Sie GRUB, EFI und „initramfs“.

Fehler zu machen gehört dazu. Das Verstehen des Fehlers ist der Unterschied zwischen Benutzer und Bediener.

---

> Dieses Handbuch richtet sich an Benutzer, die die **volle Kontrolle** über die Installation bevorzugen und dabei dem klassischen Unix-Ansatz folgen:
> **verstehen → konfigurieren → validieren → fortfahren**.

## Installation starten
Beginnen Sie mit der Void Linux ISO (x86_64 glibc oder musl).

1. Melden Sie sich als Root an
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

4. Stellen Sie eine Verbindung zum Internet her
- Für **WLAN** *(bei Kabelverbindung diesen Schritt überspringen)*:
```bash
wpa_passphrase „WIFI_NETWORK_NAME“ „NETWORK_PASSWORD“ > wifi.conf
wpa_supplicant -B -i wlan0 -c wifi.conf
dhcpcd wlan0
```
> 📌 **Hinweis:** `wlan0` kann variieren (`wlp2s0`, `wlp0s3` usw.).
> Verwenden Sie den folgenden Befehl, um die richtige Schnittstelle zu identifizieren:
>
> „Bash
> ip -br a
> ```

5. Testen Sie die Verbindung:
```bash
ping -c3 8.8.8.8
ping -c3 repo-default.voidlinux.org
```

## **Root**-Benutzeranmeldung über SSH aktivieren (optional).
Dieser Schritt gilt nur, wenn das System in einer VM ausgeführt wird; Im Falle eines lokalen Starts (ohne VM) kann die Installation normal über das lokale Terminal erfolgen.
– Dies ist erforderlich, um vom Host aus auf die **VM zuzugreifen und die Installation remote fortzusetzen; Danach können Befehle über SSH direkt im Terminal eingefügt/ausgeführt werden.

1. Konfigurieren Sie SSH
```bash
echo 'PermitRootLogin ja' >> /etc/ssh/sshd_config
```
2. Starten Sie den SSH-Dienst neu
```bash
sv sshd neu starten
```

3. Sehen Sie sich die IP der Netzwerkschnittstelle an
```bash
ip -4 Route get 1.1.1.1 | awk '{print $7}'
```
>Notieren Sie sich die IP der Netzwerkschnittstelle und verbinden Sie sich damit über SSH mit der VM.

4. Greifen Sie vom Host aus über SSH auf die VM zu.
```bash
sudo ssh <ip-da-vm>
```
> Standardpasswort: `voidlinux`

## Konfigurieren Sie eine farbige Eingabeaufforderung im Terminal (optional)
Es werden Benutzer, Host, aktueller Pfad und der Status des letzten Befehls angezeigt:
```bash
export PROMPT_COMMAND='RET=$?'
export PS1='\[\e[1;33m\]\u\[\e[0m\]@\[\e[1;35m\]\h\[\e[0m\]:\[\e[0;37m\]\w\[\e[0m\] \[\e[1;32m\]$( [ $RET -eq 0 ] && printf ✔ || printf "\e[1;31m✘$RET" )\[\e[0m\] \$ '
```
> 📌 Diese Eingabeaufforderung gilt nur für die aktuelle Sitzung; Um es dauerhaft zu machen, fügen Sie es zu „.bashrc“ hinzu.

## Erforderliche Pakete installieren
⚠️ **WICHTIG:**
```bash
xbps-install -Sy xbps parted nano vim zstd xz bash-completion
```

## Partitionieren Sie die Festplatte
1. Identifizieren Sie die Festplatte
```bash
fdisk -l | grep -E '^(Disk|Disk) '
```
> Wir gehen für das Tutorial von „/dev/sda“ aus

2. Passen Sie die folgenden Variablen entsprechend der zu verwendenden Festplatte an (**WICHTIG**):
```bash
# SATA/SCSI-Festplatten (sdX)
export DEVICE=/dev/sda
exportieren DEV_EFI=${DEVICE}2
export DEV_ROOT=${DEVICE}3
```

> 📌 **Hinweis:**
> Für **NVMe**-Festplatten ändert sich das Partitionssuffix (`p`):
> „Bash
> export DEVICE=/dev/nvme0n1
> DEV_EFI=${DEVICE}p2 exportieren
> DEV_RAIZ=${DEVICE}p3 exportieren
> ```

3. Partitionieren Sie die Festplatte mit **parted** (automatischer Modus).
Dieses Schema erstellt:
- BIOS-Partition (bios_grub)
- EFI-Partition (ESP)
- Root-Partition (ROOT)
```bash
wipefs -a "${DEVICE}"
parted --script "${DEVICE}" -- \
mklabel gpt \
mkpart primär 1MiB 2MiB Name 1 BIOS-Set 1 bios_grub auf \
mkpart Primary Fat32 2MiB 514MiB Name 2 EFI Set 2 esp on \
mkpart primär 514MiB 100 % Name 3 ROOT \
Align-Check optimal 1 \
Align-Check optimal 2 \
Align-Check optimal 3
parted --script "${DEVICE}" -- drucken
```

## Partitionen formatieren
```bash
# Formatieren Sie die Root-Partition (ext4)
mkfs.ext4 -F ${DEV_RAIZ}

# Formatieren Sie die EFI-Partition (FAT32)
mkfs.fat -F32 -I ${DEV_EFI}
```

## Hängen Sie die Volumes in „/mnt“ ein
```bash
# Mounten Sie die Root-Partition
Mount ${DEV_RAIZ} /mnt

# Erstellen Sie die erforderlichen Mount-Punkte
/t /mnt/{hame,boot/efi,var/log,var/cache, proced, proc, Proc,)

# Mounten Sie die EFI-Partition
mounten Sie ${DEV_EFI} /mnt/boot/efi
```

## Installieren Sie das Basissystem
Installiert das Void Linux-Basissystem in der „/mnt“-gemounteten Umgebung, einschließlich Kernel, Firmware, Bootloader, Netzwerk und wichtigen Tools.
```bash
xbps-install -Sy -R https://repo-default.voidlinux.org/current \
-r /mnt \
Basissystem e2fsprogs grub-x86_64-efi dracut linux \
linux-headers linux-firmware linux-firmware-network glibc-locales \
xtools dhcpcd openssh vim nano grc zstd xz bash-completion vpm vsv \
socklog-void wget net-tools tmate ncurses chrony
```

> 📌 **Hinweis:**
> - `grub-x86_64-efi` → Bootloader UEFI
> - `linux` → Kernel
> - `linux-firmware-network` → Netzwerktreiber
> - „xtools“ → erforderlich, um „xgenfstab“ unbedingt verwenden zu können

## Erstellen Sie „fstab“.
Erzeugt automatisch die permanente Mount-Datei des Systems.
```bash
xgenfstab -U /mnt > /mnt/etc/fstab
```

## Eingang und System (chroot)
Greifen Sie auf das unter „/mnt“ installierte System zu, um mit der Konfiguration fortzufahren.
```bash
xchroot /mnt /bin/bash
```

## INITRAMFS generieren
Dracut-Konfiguration für Virtualisierungsumgebungen (VM-sicher)
```bash
cat > /etc/dracut.conf.d/99-vm-safe.conf << 'EOF'
# /etc/dracut.conf.d/99-vm-safe.conf
hostonly=nein
compress="gzip"
add_drivers+=" virtio virtio_pci virtio_blk virtio_net virtio_scsi "
EOF
```

Erkennt automatisch die installierte Kernel-Version und generiert die entsprechenden „initramfs“ mit **dracut**.
```bash
mods=(/usr/lib/modules/*)
KVER=$(basename „${mods[0]}“)
echo ${KVER}
dracut --force --kver ${KVER}
```

## GRUB konfigurieren

> 📌 Beide Methoden (BIOS und UEFI) werden absichtlich installiert.
> Dadurch kann dieselbe Festplatte sowohl auf **Legacy-BIOS**- als auch auf **UEFI**-Systemen gestartet werden, was die Portabilität zwischen Maschinen erhöht.

1. Erstellen Sie das GRUB-Supportverzeichnis:
```bash
mkdir -p /boot/grub
```

2. Installieren Sie GRUB für **BIOS (Legacy)**:
```bash
grub-install --target=i386-pc ${DEVICE}
```

3. Installieren Sie GRUB für **UEFI**:
```bash
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=void
```

4. Erstellen Sie einen UEFI-Fallback (Universal Boot).
Diese Datei garantiert das Booten, auch wenn NVRAM gelöscht wird:
```bash
mkdir -p /boot/efi/EFI/BOOT
cp -f /boot/efi/EFI/void/grubx64.efi /boot/efi/EFI/BOOT/BOOTX64.EFI
```

5. Generieren Sie die endgültige GRUB-Konfigurationsdatei:
```bash
grub-mkconfig -o /boot/grub/grub.cfg
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

## Grundeinstellungen
```bash
# Hostnamen festlegen
echo void > /etc/hostname

# Lokalzeit einstellen
ln -sf /usr/share/zoneinfo/America/Sao_Paulo /etc/localtime

# Lokal einstellen
sed -i 's/#en_US.UTF-8/en_US.UTF-8/' /etc/default/libc-locales
sed -i 's/#pt_BR.UTF-8/pt_BR.UTF-8/' /etc/default/libc-locales

# Gebietsschemas generieren:
xbps-reconfigure -f glibc-locales

# Möglichen Fehler im /var/service-Symlink beheben (wichtig):
rm -f /var/service
ln -sf /etc/runit/runsvdir/default /var/service

# Einige Dienste aktivieren:
ln -sf /etc/sv/dbus /var/service/
ln -sf /etc/sv/dhcpcd /var/service/
ln -sf /etc/sv/sshd /var/service/
ln -sf /etc/sv/nanoklogd /var/service/
ln -sf /etc/sv/socklog-unix /var/service/
ln -sf /etc/sv/chronyd /var/service/

# Benutzerdefiniertes Svlogtail herunterladen (optional, aber empfohlen):
wget --quiet --no-check-certificate -O /usr/bin/svlogtail \
"https://raw.githubusercontent.com/voidlinux-br/void-install/refs/heads/main/svlogtail" && \
chmod +x /usr/bin/svlogtail

# Erstellen Sie eine resolv.conf
printf 'Nameserver 1.1.1.1\nNameserver 8.8.8.8\n' > /etc/resolv.conf

#Sudo-Radgruppe konfigurieren (optional, aber empfohlen)
cat << 'EOF' > /etc/sudoers.d/g_wheel
%wheel ALL=(ALL:ALL) NOPASSWD: ALLE
EOF

#Erforderliche Berechtigungen
chmod 440 /etc/sudoers.d/g_wheel
```

## Passen Sie „/etc/xbps.d/00-repository-main.conf“ an
*(Optional, aber empfohlen)*

Erstellt das **XBPS**-Konfigurationsverzeichnis (sofern es noch nicht vorhanden ist) und definiert eine Liste offizieller und alternativer Repositorys.
**repo-fastly** Repositories haben tendenziell eine bessere Latenz.

```bash
mkdir -pv /etc/xbps.d

cat << 'EOF' > /etc/xbps.d/00-repository-main.conf
# Offizielles Repository (Fastly – beste Latenz)
Repository=https://repo-fastly.voidlinux.org/current
#repository=https://repo-fastly.voidlinux.org/current/nonfree
#repository=https://repo-fastly.voidlinux.org/current/multilib
#repository=https://repo-fastly.voidlinux.org/current/multilib/nonfree

# Alternatives Repository (Chili Linux)
Repository=https://void.chililinux.com/voidlinux/current
#repository=https://void.chililinux.com/voidlinux/current/extras
#repository=https://void.chililinux.com/voidlinux/current/nonfree
#repository=https://void.chililinux.com/voidlinux/current/multilib
#repository=https://void.chililinux.com/voidlinux/current/multilib/nonfree
EOF
```

## Passen Sie „/etc/rc.conf“ an
Legt die Standardzeitzone, das Tastaturlayout und die Schriftart der Konsole fest.
Ändern Sie es nach Bedarf.
```bash
cat << 'EOF' > /etc/rc.conf
ZEITZONE=Amerika/Sao_Paulo
KEYMAP=br-abnt2
FONT=Lat2-Terminus16
EOF
```

Virtio-Module (virtuelle Maschine).
```bash
cat > /etc/modules-load.d/virtio.conf << 'EOF'
Virtio
virtio_pci
virtio_net
virtio_blk
virtio_scsi
EOF
```

## Passen Sie den Benutzer „.bashrc“ an
Erstellt ein Standard-„.bash_profile“ und stellt sicher, dass „.bashrc“ beim Anmelden automatisch geladen wird.
> ⚠️ Stellen Sie sicher, dass der Benutzer im vorherigen Schritt erstellt wurde.

```bash
# Laden Sie die Standard-.bashrc-Datei nach /etc/skel herunter
wget --quiet --no-check-certificate \
-O /etc/skel/.bashrc \
„https://raw.githubusercontent.com/voidlinux-br/void-install/refs/heads/main/.bashrc"

chown root:root /etc/skel/.bashrc
chmod 644 /etc/skel/.bashrc

# Erstellen Sie ein Standard-.bash_profile
cat << 'EOF' > /etc/skel/.bash_profile
# ~/.bash_profile – lädt .bashrc in Void

# Wenn .bashrc vorhanden ist, laden
if [ -f ~/.bashrc ]; Dann
Quelle ~/.bashrc
Sei
EOF

# In Root und Benutzer kopieren
für d in /root "/home/${NEWUSER}"; Tun
cp -f /etc/skel/.bash_profile "$d/"
cp -f /etc/skel/.bashrc "$d/"
Erledigt

# Benutzerberechtigungen anpassen
chown "${NEWUSER}:${NEWUSER}" \
"/home/${NEWUSER}/.bash_profile" \
"/home/${NEWUSER}/.bashrc"

chmod 644 \
"/home/${NEWUSER}/.bash_profile" \
"/home/${NEWUSER}/.bashrc"
```

## SSH konfigurieren
*(Optional, aber empfohlen)*

Erstellt eine zusätzliche Konfigurationsdatei für **sshd**, wobei die Hauptdatei intakt bleibt.
```bash
mkdir -pv /etc/ssh/sshd_config.d

cat << 'EOF' > /etc/ssh/sshd_config.d/10-custom.conf
# Allgemeine Einstellungen
PermitTTY ja
PrintMotd ja
PrintLastLog ja
Banner /etc/issue.net

# Authentifizierung
PermitRootLogin ja
PasswordAuthentication ja
KbdInteractiveAuthentication ja
ChallengeResponseAuthentication ja
PubkeyAuthentication ja
PubkeyAcceptedKeyTypes=+ssh-rsa
AuthorizedKeysFile .ssh/authorized_keys
Benutze PAM ja

# Merkmale
X11Forwarding ja
Subsystem SFTP internal-sftp
EOF
```
> ⚠️ Es wird empfohlen, diese SSH-Einstellungen nach dem ersten Start zu überprüfen und zu härten, insbesondere auf Systemen, die mit dem Internet verbunden sind.

## Installation abschließen
Sair chrootet
```bash
Ausfahrt
```
Hängen Sie alle auf „/mnt“ gemounteten Partitionen aus (einschließlich Subvolumes und „/boot/efi“):
```bash
umount -R /mnt
```
Starten Sie die physische Maschine oder VM neu, um den tatsächlichen Start zu testen:
```bash
neu starten
```
> 📌 **Hinweis: Vergessen Sie nicht, das Installationsmedium zu entfernen.

# 🎉 Viel Spaß!
**Void Linux** ist jetzt installiert und einsatzbereit.

# HAFTUNGSAUSSCHLUSS

> Dieses Tutorial ist kostenlos: Sie können es nach Belieben verwenden, kopieren, ändern und weiterverbreiten.
> Inhalte werden unter der **MIT-Lizenz** zur Verfügung gestellt und können Auszüge oder Befehle enthalten, die von Open-Source-Software abgeleitet sind, vorbehaltlich ihrer eigenen Lizenzen.
>
> Es werden keine Garantien übernommen – hier wird alles **„wie besehen“** geliefert.
> Nutzung auf eigene Gefahr. Weder der Autor noch die Mitwirkenden noch Void Linux sind verantwortlich für Verluste, Schäden, Systemausfälle oder irgendwelche Folgen aus der Verwendung dieses Materials.
>
> Es steht Ihnen frei, dieses Tutorial zu überprüfen, anzupassen und Ihre eigene Version zu erstellen.

