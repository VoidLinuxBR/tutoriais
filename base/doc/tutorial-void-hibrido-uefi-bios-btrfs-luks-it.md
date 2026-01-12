# 🧩 TUTORIAL VOID LINUX — INSTALLAZIONE IBRIDA (UEFI + BIOS) CON EXT4, XFS, JFS O BTRFS (SOTTOVOLUMI), LUKS, HIBERNATION E ZRAM
### VERSIONE REVISIONATA E CONVALIDA — PARTIZIONAMENTO CORRETTO + BOOT UNIVERSALE

Questa guida installa un Void Linux completamente **ibrido**, in grado di avviare qualsiasi tipo di macchina: vecchia, nuova o problematica:

- 💾 **UEFI moderno** (con input normale e fallback)
- 🧮 **BIOS/Legacy** (piena compatibilità)
- 🧰 **GPT con BIOS Boot (EF02)**: massimo supporto per il vecchio hardware
- 🚀 **Btrfs con sottovolumi** (opzionale), istantanee già pronte
- 🔐 **LUKS1 pienamente compatibile con GRUB**
- 🌙 **Ibernazione reale tramite file di scambio**
- 🧊 **ZRAM configurata per le prestazioni**
- 🧱 **Supporto completo per EXT4, XFS, JFS e BTRFS**
- 💡 **Initramfs/GRUB configurato automaticamente (LUKS + curriculum)**

📌 **Nessun compromesso, nessuna reinstallazione di GRUB, nessuna perdita di tempo.**
📌 **Avvio garantito anche su una macchina con NVRAM cancellata (fallback BOOTX64.EFI).**

---

# ▶️ 1. Avvia o Live ISO

Suggerimento: utilizzare la versione glibc per una compatibilità superiore:
- scarica l'iso da:
```
https://repo-default.voidlinux.org/live/current/void-live-x86_64-20250202-base.iso
```
- oppure cerca l'ultima versione su:
```
https://voidlinux.org/download/
```

1. Accedi come root.
```bash
login: root
password: voidlinux
```

2. Cambia la shell da *sh* a *bash*.
*dash/sh* **NON implementa** diverse funzionalità utilizzate da molti script.
```bash
bash
```

3. Cambia il layout della tastiera in **ABNT2**, assicurando la corretta mappatura di accenti e simboli:
```bash
tasti di caricamento br-abnt2
```

4. Incolla nel terminale (facoltativo): prompt con colori, utente@host:percorso e stato dell'ultimo comando ( ✔/✘). Utile e bello.
```bash
esporta PS1='\[\e[1;32m\]\u\[\e[1;33m\]@\[\e[1;36m\]\h\[\e[1;31m\]:\w \
$([[ $? -eq 0 ]] && echo -e "\e[1;32m ✔" || echo -e "\e[1;31m✘$?") \
\[\e[0m\]\$ '
```

# ▶️ 2. Connettiti a Internet
- Per **Wi-Fi** *(se via cavo, salta questo passaggio)*:
```bash
wpa_passphrase "SSID" "PASSWORD" > wifi.conf
wpa_supplicant -B -i wlan0 -c wifi.conf
dhcpcd wlan0
```

1. Testare la connessione:
```bash
ping -c3 8.8.8.8
ping -c3 repository-default.voidlinux.org
```

2. Installa i pacchetti richiesti:
⚠️ **IMPORTANTE:**
```bash
xbps-install -Sy xbps parted jfsutils xfsprogs nano zstd xz bash-completion
```
---

# ▶️ 3. Identificare il disco
1. Elenca i dischi disponibili e annota il nome del dispositivo (es: `/dev/sda`, `/dev/vda`, `/dev/nvme0n1`):
```bash
fdisk -l | grep -E '^(Disco|Disco) '
```

# ▶️ 4. Definire le variabili utilizzate nel tutorial:
⚠️ **IMPORTANTE:**

1. Definire i dispositivi **PRIMA** dell'uso:
> 1. **Presupporremo** per il tutorial `/dev/sda` (normale) o `/dev/nvme0n1` (nvme)
> 2. **Regola** in base al tuo disco (scegli solo **un** o **un altro** modello)

Per dischi **normali** (ad esempio /dev/sda)
```bash
esporta DISPOSITIVO=/dev/sda
esporta DEV_BIOS=${DEVICE}1
esporta DEV_EFI=${DEVICE}2
esporta DEV_ROOT=${DEVICE}3
esporta DEV_LUKS=/dev/mapper/cryptroot
```
Per i dischi **NVMe** (ad esempio /dev/nvme0n1), il suffisso della partizione cambia (`p`)
```bash
esporta DISPOSITIVO=/dev/nvme0n1
esporta DEV_BIOS=${DEVICE}p1
esporta DEV_EFI=${DEVICE}p2
esporta DEV_RAIZ=${DEVICE}p3
esporta DEV_LUKS=/dev/mapper/cryptroot
```

> 📌 **Nota:**
> DISPOSITIVO → intero disco
DEV_BIOS → Partizione di avvio del BIOS (1–2 MiB, senza FS, non si monta)
DEV_EFI → Partizione EFI (FAT32)
DEV_ROOT → partizione root (normale o LUKS)
DEV_LUKS → Mappatura LUKS (/dev/mapper/cryptroot)

- 👉Qui definisci l'anatomia del disco. Tutto il resto della guida segue semplicemente queste variabili.
- 🔎 Perché è necessario?
Perché dichiarare tutto all'inizio rende il processo successivo a prova di errori di battitura.

2. Definire **KEYMAP** e **TIMEZONE** (modificare secondo necessità):
```bash
esporta KEYMAP=br-abnt2
```
```bash
esporta TIMEZONE=America/San_Paolo
```

---

# ▶️ 5. Disco delle partizioni
> - La partizione del BIOS **DEVE** essere la prima. Ciò aumenta la compatibilità con schede madri più vecchie, bootloader problematici e BIOS che prevedono codice di avvio nelle prime aree del disco.
> - L'ESP può arrivare più tardi senza alcun problema: UEFI non si preoccupa della posizione.

### Ordine ideale e corretto:

- 1️⃣ Avvio del BIOS (EF02)
- 2️⃣ ESP (sistema EFI, FAT32)
- 3️⃣ Btrfs/Ext4/Xfs/Jfs (root)

### Partizionare utilizzando parted (automatico)
> Qui il **DEVICE** è già definito lassù, quindi non esiste una variabile “magica”.
```
wipefs -a "${DEVICE}"
parted --script "${DEVICE}" -- \
etichetta mkgpt \
mkpart primario 1MiB 2MiB nome 1 BIOS impostato 1 bios_grub su \
mkpart primario fat32 2MiB 514MiB nome 2 EFI set 2 esp on \
mkpart primario 514MiB 100% nome 3 ROOT \
allineamento-controllo ottimale 1 \
allineamento-controllo ottimale 2 \
allineamento-controllo ottimale 1
parted --script "${DEVICE}" -- stampa
```
> - Partizione 1 → Avvio del BIOS (bios_grub, no FS, non si monta)
> - Partizione 2 → EFI (FAT32)
> - Partizione 3 → ROOT (la formatteremo in seguito con EXT4/XFS/JFS/BTRFS, con o senza LUKS)
> - Ho utilizzato mkpart primario 514MiB al 100% senza specificare FS proprio per evitare di impegnare il FS. Scegli FS più tardi.
---

# ▶️ 6. Scegli la modalità di installazione (NORMAL o LUKS)
⚠️ **IMPORTANTE:**
> Scegli SOLO UNO dei due blocchi sottostanti.
**NON** per eseguire entrambi i passaggi.

1. INSTALLAZIONE NORMALE **(senza LUKS)**
```bash
esporta DISCO="${DEV_RAIZ}"
```
- Imposta DISK sul dispositivo reale /dev/sda3

2. INSTALLAZIONE **CON LUKS** (root crittografato)
```
# Crittografa SOLO la partizione root su LUKS1 (compatibile con GRUB) - mai l'intero disco
# Crittografa la partizione confermando con SI:
cryptsetup luksFormat --type luks1 "${DEV_RAIZ}"

# Apri la partizione con la tua passphrase.
cryptsetup apri la cryptroot "${DEV_RAIZ}".

# Da ora in poi, la vera root sarà il dispositivo mappato
esporta DISCO="${DEV_LUKS}"
```
- LUKS si trova sopra /dev/sda3, non sull'intero disco
- Il sistema verrà installato in /dev/mapper/cryptroot

👉 Da qui in poi, TUTTO utilizza $DISK.

---

# ▶️ 7. Crea il file system (FS) e monta root
⚠️ **IMPORTANTE:**
> Scegli SOLO UNO dei due blocchi sottostanti.

1. **EST4**
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
4. **BTRFS semplice**
```
mkfs.btrfs -f "${DISK}" -L ROOT
mount -v "${DISK}" /mnt
```
5. **BTRFS con sottovolumi**
```
mkfs.btrfs -f "${DISK}" -L ROOT

monta ${DISK} /mnt
il sottovolume btrfs crea /mnt/@
Il sottovolume btrfs crea /mnt/@home
Il sottovolume btrfs crea /mnt/@log
Il sottovolume btrfs crea /mnt/@cache
Il sottovolume btrfs crea /mnt/@snapshots
smonta /mnt

mount -o defaults,noatime,ssd,compress=zstd:3,discard=async,space_cache=v2,commit=300,subvol=/@ ${DISK} /mnt
mkdir -p /mnt/{boot/efi,home,var/log,var/cache,.snapshots,swap}

mount -o defaults,noatime,ssd,compress=zstd:3,discard=async,space_cache=v2,commit=300,subvol=/@home ${DISK} /mnt/home
mount -o defaults,noatime,ssd,compress=zstd:3,discard=async,space_cache=v2,commit=300,subvol=/@cache ${DISK} /mnt/var/cache
mount -o defaults,noatime,ssd,compress=zstd:3,discard=async,space_cache=v2,commit=300,subvol=/@log ${DISK} /mnt/var/log
mount -o defaults,noatime,ssd,compress=zstd:3,discard=async,space_cache=v2,commit=300,subvol=/@snapshots ${DISK} /mnt/.snapshots
```
---

# ▶️ 8. Preparare e assemblare l'ESP (EFI)
```
mkfs.fat -F32 -I "${DEV_EFI}"
mkdir -p /mnt/boot/efi
mount -v "${DEV_EFI}" /mnt/boot/efi
```
>💡 La partizione del BIOS (${DEV_BIOS}) non ha file system, non si formatta, non si monta.
---

# ▶️ 9. Installa o Void Linux senza chroot

1. Copiare le chiavi del repository (chiavi XBPS) da utilizzare nel chroot (/mnt)
```
mkdir -p /A{tc, vapore/xbps/xbps)
cp -rpaf /var/db/xbps/keys/*.plist /mnt/var/db/xbps/keys/
cp -fpa /etc/resolv.conf /mnt/etc/resolv.conf
```

2. Installa il sistema di base sul disco appena montato:
```
xbps-install -Sy -R https://repo-default.voidlinux.org/current \
-r /mnt \
sistema base btrfs-progs cryptsetup grub grub-x86_64-efi dracut linux \
intestazioni-linux firmware-linux rete-firmware-linux glibc-locales \
xtools dhcpcd openssh vim nano grc zstd xz bash-completion vpm vsv \
socklog-void wget net-tools tmate ncurses jfsutils xfsprogs duf tree eza chrony
```
---

# ▶️ 10. Gerar fstab no /mnt (chroot)
```bash
# Genera fstab in /mnt/etc/fstab
xgenfstab -U /mnt > /mnt/etc/fstab
```

```bash
# controlla se è stato generato correttamente
cat /mnt/etc/fstab
```

# ▶️ 11. Accedi al sistema installato utilizzando chroot

1. Impiegare non croit:
```
xchroot /mnt /bin/bash
```
---

# ▶️ 12. Impostazioni iniziali (in chroot)
```
# configura hostname: definisce il nome host
echo void > /etc/nomehost

# configura fuso orario: definisce il fuso orario
ln -sfv /usr/share/zoneinfo/"${TIMEZONE}" /etc/localtime

# configura le impostazioni locali
-i -e 's/^#\(en_.Utf-8 UTF-8\)/)/' \
-E 's/^#\pt_br.br.utf-8 UTF-8\)/' \' \
/etc/default/libc-locales

# genera locali
xbps-reconfigure -f glibc-locales

# Correggi il possibile errore nel collegamento simbolico /var/service (importante):
rm -f /var/servizio
ln -sf /etc/runit/runsvdir/default /var/service

# Attiva alcuni servizi
ln -sf /etc/sv/dbus /var/servizio/
ln -sf /etc/sv/dhcpcd /var/service/
ln -sf /etc/sv/sshd /var/servizio/
ln -sf /etc/sv/nanoklogd /var/service/
ln -sf /etc/sv/socklog-unix /var/service/
ln -sf /etc/sv/chronyd /var/service/

# Configura sudo - wheel group (opzionale, ma consigliato)
cat << 'EOF' > /etc/sudoers.d/g_wheel
%wheel ALL=(ALL:ALL) NOPASSWD: TUTTI
EOF
#Autorizzazioni richieste
chmod 440 /etc/sudoers.d/g_wheel
```

## Creazione e configurazione degli utenti

⚠️ **IMPORTANTE:** definisci di seguito il nome utente reale.
```bash
esporta NEWUSER=tuo_utente_qui
```

Crea l'utente con la directory home, i gruppi di base e la shell Bash:
```bash
useradd -m -G audio,video,ruota,tty -s /bin/bash ${NEWUSER}
```

Imposta la tua password utente (***IMPORTANTE***)
```bash
passwd ${NEWUSER}
```

Imposta la password dell'utente root (***IMPORTANTE***)
```bash
password root
```

Cambia la shell predefinita dell'utente root in Bash
```bash
chsh -s /bin/bash root
```
---

# ▶️ 13. Configura gli UUID
⚠️ **IMPORTANTE:**
- Ottieni gli UUID delle partizioni:
```
UUID_LUKS=$(blkid -s UUID -o valore "${DEV_RAIZ}")
UUID_ROOT=$(blkid -s UUID -o valore "${DISK}")
UUID_EFI=$(blkid -s UUID -o valore "${DEV_EFI}")
```
---

# ▶️ 14. Crea file di scambio con supporto per l'ibernazione (opzionale)

### Note importanti
```
- Il file di scambio in Btrfs appare sempre come **prealloc**, è normale.
- Non è necessario che abbia la dimensione intera della RAM.
- Nella maggior parte dei casi, il 60% è sufficiente per l'ibernazione.
- Per carichi pesanti → utilizzare il 70% o l'80%.
```

1. Calcola automaticamente la dimensione ottimale del file di scambio
```
# Raccomandazione moderna per l'ibernazione: 60% della RAM totale
SWAP_GB=$(LC_ALL=C awk '/MemTotal/ {print int($2 * 0.60 / 1024 / 1024)}' /proc/meminfo)
echo "File di scambio consigliato: ${SWAP_GB}G"
```
- oppure, imposta manualmente la dimensione desiderata:
```
SWAP_GB=4
echo "File di scambio definito dall'utente: ${SWAP_GB}G"
```
2. Creare la directory per il file di scambio
```
mkdir -p /scambia
swapoff -a 2>/dev/null
rm -f /scambia/scambiafile
```
3. Disabilita COW (richiesto in Btrfs)
```
chattr +C /scambia
```
4. Creare il file di scambio con la dimensione definita in precedenza
```
fallocate -l ${SWAP_GB}G /swap/swapfile
chmod 600 /swap/swapfile
```
5. Formattare il file di scambio e attivare lo scambio
```
mkswap /swap/swapfile
swapon /swap/swapfile
```
6. Ottieni l'offset:
```
# Installa il pacchetto per filefrag
xbps-install -Sy e2fsprogs

# Ottieni l'offset
offset=$(filefrag -v /swap/swapfile | awk '/^ *0:/{print $4}')
```
---

# ▶️ 15. Configura GRUB
⚠️ **IMPORTANTE:**
> Questo blocco è intelligente:
- Rileva automaticamente se stai utilizzando LUKS
- Rileva se hai creato il file di scambio con l'ibernazione
- Regola /etc/default/grub senza duplicare nulla
- Crea le righe necessarie solo se mancano
- Non cambiare nulla se non è necessario

Utilizza esattamente il blocco seguente:
```
HAS_RESUME=falso
HAS_LUKS=falso

[[ -n "${offset}" ]] && HAS_RESUME=true
[[ "${DISK}" = "${DEV_LUKS}" ]] && HAS_LUKS=true

# Rimuovi la vecchia linea per sicurezza
sed -i '/^[[:spazio:]]*GRUB_CMDLINE_LINUX_DEFAULT=/d' /etc/default/grub

#GRUB_CMDLINE_LINUX

# Valore base
BASE="livellolog=4"

# Aggiungi riepilogo
se $HAS_RESUME; Poi
BASE="$BASE curriculum=UUID=${UUID_ROOT} curriculum_offset=${offset}"
Essere

# Aggiungi LUKS
se $HAS_LUKS; Poi
grep -q '^GRUB_ENABLE_CRYPTODISK=y' /etc/default/grub || echo 'GRUB_ENABLE_CRYPTODISK=y' >> /etc/default/grub
grep -q '^GRUB_PRELOAD_MODULES=' /etc/default/grub || echo 'GRUB_PRELOAD_MODULES="luks criptodisco gcry_rijndael"' >> /etc/default/grub
BASE="$BASE rd.luks.uuid=${UUID_LUKS} rd.luks.name=${UUID_LUKS}=cryptroot root=/dev/mapper/cryptroot"
Essere

# Ricrea correttamente la riga finale
echo "GRUB_CMDLINE_LINUX_DEFAULT=\"${BASE}\"" >> /etc/default/grub
```
---

# ▶️ 16. Ricrea l'initrd
⚠️ **IMPORTANTE:**
```
mods=(/usr/lib/modules/*)
KVER=$(nomebase "${mods[0]}")
echo ${KVER}
dracut --force --kver ${KVER}
```
---

# ▶️ 17. Crea un file di chiavi per evitare di chiedere la password due volte all'avvio (solo LUKS)
> Se il sistema NON utilizza LUKS, saltare questo passaggio.
```
se ["${DISK}" = "${DEV_LUKS}"]; Poi
echo "Rilevato LUKS: creazione del file di chiavi per lo sblocco automatico..."

# Crea un file di chiavi sicuro
dd if=/dev/urandom of=/boot/volume.key bs=64 count=1
chmod 000 /boot/volume.key

# Aggiungi il file di chiavi a LUKS (ti chiederà la tua password attuale)
cryptsetup luksAddKey "${DEV_RAIZ}" /boot/volume.key

# Configura /etc/crypttab
cat << EOF >> /etc/crypttab
cryptroot ${DEV_RAIZ} /boot/volume.key luks
EOF

# Include file di chiavi e crypttab in initramfs
mkdir -p /etc/dracut.conf.d
cat << EOF >> /etc/dracut.conf.d/10-crypt.conf
install_items+=" /boot/volume.key /etc/crypttab "
EOF

# Rigenera initramfs con il supporto dei file di chiavi
xbps-riconfigurare -fa
altro
echo "Sistema senza LUKS: saltare la creazione del file di chiavi."
Essere
```

# ▶️ 18. Installa GRUB in **BIOS** e **UEFI** (vero ibrido)
1. Installa GRUB per BIOS (legacy)
```
grub-install --target=i386-pc ${DEVICE}
```
2. Installa GRUB per UEFI
```
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=void
```
3. Creare il fallback UEFI (avvio universale). Questo file garantisce l'avvio anche quando la NVRAM viene cancellata.
```
mkdir -p /boot/efi/EFI/BOOT
cp -f /boot/efi/EFI/void/grubx64.efi /boot/efi/EFI/BOOT/BOOTX64.EFI
```
4. Genera il file GRUB finale
```
grub-mkconfig -o /boot/grub/grub.cfg
```

---

# ▶️ 19. Impostazioni utente personalizzate:

1. Impostazioni dell'ambiente:

```
# Personalizza /etc/xbps.d/00-repository-main.conf
mkdir -pv /etc/xbps.d
cat << 'EOF' >> /etc/xbps.d/00-repository-main.conf
archivio=https://repo-fastly.voidlinux.org/current
#repository=https://repo-fastly.voidlinux.org/current/nonfree
#repository=https://repo-fastly.voidlinux.org/current/multilib
#repository=https://repo-fastly.voidlinux.org/current/multilib/nonfree

archivio=https://void.chililinux.com/voidlinux/current
#repository=https://void.chililinux.com/voidlinux/current/extras
#repository=https://void.chililinux.com/voidlinux/current/nonfree
#repository=https://void.chililinux.com/voidlinux/current/multilib
#repository=https://void.chililinux.com/voidlinux/current/multilib/nonfree
EOF

# Personalizza /etc/rc.conf. Imposta il fuso orario, il layout della tastiera e il carattere predefiniti della console. Cambia secondo necessità.
cat << EOF >> /etc/rc.conf
FUSO ORARIO="${TIMEZONE}"
KEYMAP="${KEYMAP}"
FONT=Lat2-Terminus16
EOF

# Personalizza il file .bashrc di root
wget --quiet --no-check-certificate \
-O /etc//skel/.bashrc \
"https://raw.githubusercontent.com/voidlinux-br/void-install/refs/heads/main/.bashrc"
chown root:root /etc/skel/.bashrc
chmod 644 /etc/skel/.bashrc

cat << 'EOF' > /etc/skel/.bash_profile
# ~/.bash_profile — carica .bashrc in Void

# Se esiste .bashrc, caricalo
if [ -f ~/.bashrc ]; Poi
fonte ~/.bashrc
Essere
EOF

# copia su root e utente
for d in /root "/home/${NEWUSER}"; Fare
cp -f /etc/skel/.bash_profile "$d/"
cp -f /etc/skel/.bashrc "$d/"
Fatto

chown "${NEWUSER}:${NEWUSER}" "/home/${NEWUSER}/.bash_profile" "/home/${NEWUSER}/.bashrc"
chmod 644 "/home/${NEWUSER}/.bash_profile" "/home/${NEWUSER}/.bashrc"

# scarica svlogtail personalizzato
wget --quiet --no-check-certificate \
-O /usr/bin/svlogtail\
"https://raw.githubusercontent.com/voidlinux-br/void-install/refs/heads/main/svlogtail"
chmod +x /usr/bin/svlogtail
```

2. Configura ssh (opzionale, ma consigliato):
```
mkdir -pv /etc/ssh/sshd_config.d/
cat << 'EOF' > /etc/ssh/sshd_config.d/10-custom.conf
PermessoTTY sì
PrintMotd sì
StampaUltimoLog sì
Banner /etc/issue.net

Permetti RootLogin sì
KbdInteractiveAuthentication sì
X11Inoltro sì
Autenticazione Pubkey sì
PubkeyAcceptedKeyTypes=+ssh-rsa
AuthorizedKeysFile .ssh/authorized_keys
Autenticazione password sì
ChallengeResponseAutenticazione sì
Usa PAM sì

Sottosistema sftp interno-sftp
EOF
```
---

# ▶️ 20. Abilita ZRAM (opzionale)
Void Linux utilizza il servizio zramen per abilitare ZRAM, creando un blocco di memoria compressa che riduce l'utilizzo dello scambio SSD e migliora le prestazioni sotto carico.
1. Installa zramen
```
xbps-install -Sy zramen
```
2. Configura ZRAM (configurazione consigliata):
```
cat << 'EOF' > /etc/zramen.conf
zram_frazione=0,5
dispositivi_zram=1
algoritmo_zram=zstd
EOF
```
3. Attivare il servizio in runit
```
ln -s
```
> ZRAM verrà attivata automaticamente ad ogni avvio

---

# ▶️ 21. Termina l'installazione
1. Sair do chroot:
```
Uscita
```
2. Smontare tutte le partizioni montate su /mnt (sottovolumi e /boot/efi):
```
smonta -R /mnt
```
3. Disabilitare qualsiasi file di scambio o partizione di scambio che è stata attivata all'interno del chroot:
```
scambio -a
```
4. Riavviare la macchina fisica o la VM per testare l'avvio effettivo:
```
riavviare
```
> Non dimenticare di rimuovere il supporto di installazione e avviare dal disco appena installato.
Godere!

---

# 🎉SISTEMA COMPLETO, IBRIDO, A PROVA DI FUTURO
-Avvio BIOS + UEFI
- UEFI di riserva
- Btrfs con istantanee (pronto per Snapper/Timeshift)
- Ibernazione reale con file di scambio
- Zram per le prestazioni

Questo SSD avvia **qualsiasi macchina sul pianeta**.

# DISCLAIMER

```
Questo tutorial è gratuito: puoi utilizzare, copiare, modificare e ridistribuire come desideri.
Il contenuto è reso disponibile sotto la **Licenza MIT** e può includere frammenti o comandi derivati da software open source soggetti alle proprie licenze.

Non viene fornita alcuna garanzia: qui tutto viene consegnato "così com'è".
Utilizzare a proprio rischio. Né l'autore, né i contributori, né Void Linux sono responsabili per perdite, danni, guasti di sistema o qualsiasi conseguenza derivante dall'uso di questo materiale.

Se lo desideri, puoi ottenere il codice sorgente, rivedere, adattare e generare la tua versione di questo tutorial.
```

