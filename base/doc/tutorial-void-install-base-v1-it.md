# 🔥 Tutorial sull'installazione di Base Void Linux

# Prima di iniziare

Questo tutorial descrive un'**installazione manuale di Void Linux**, utilizzando il partizionamento diretto del disco, `chroot` e la configurazione esplicita del sistema.
**Non è un programma di installazione automatico**.

## ⚠️Leggi attentamente

- Questa guida **presuppone familiarità con Linux**, terminale e concetti di base del sistema (dischi, partizioni, avvio, servizi).
- Diversi comandi **eliminano permanentemente i dati** (`parted`, `mkfs`, `umount -R`).
- Un errore durante la definizione del disco (`/dev/sdX`, `/dev/nvmeX`) può comportare la **perdita totale dei dati**.
- Leggi **l'intero tutorial prima di eseguire qualsiasi comando**.

## 🖥️Ambiente consigliato

- **VM (VirtualBox, QEMU, KVM, ecc.)** per test e apprendimento.
- Hardware dedicato **nessun dato importante**.
- Ambiente di laboratorio o struttura cosciente.

❌ **Non consigliato** per l'uso diretto in produzione senza adattamenti.

## 🔐 A proposito di sicurezza

Durante il processo di installazione, alcune impostazioni **danno priorità alla praticità**, non alla sicurezza:
- L'accesso dell'utente "root" tramite SSH può essere temporaneamente abilitato.
- L'autenticazione della password potrebbe essere attiva.
- Potrebbe essere consentita la compatibilità legacy (ad esempio `ssh-rsa`).

👉 **Queste impostazioni devono essere riviste dopo l'installazione**, soprattutto sui sistemi esposti alla rete.

## 🧠 Importante da sapere

- Esegui i comandi **uno per uno**, controllando l'output.
- Modifica i nomi dei dischi, le interfacce di rete e gli utenti in base al tuo sistema.
- **Non copiare e incollare ciecamente**.
- In caso di dubbi, **interrompi** e rivedi il passaggio corrente.

## 🛠️ In caso di errore

Se qualcosa va storto:
- Non riavviare alla cieca.
- Rimontare le partizioni.
- Accedi nuovamente al sistema con `chroot`.
- Controlla GRUB, EFI e "initramfs".

Commettere errori ne fa parte. Comprendere l'errore è ciò che distingue l'utente dall'operatore.

---

> Questa guida è rivolta agli utenti che preferiscono il **controllo completo** dell'installazione, seguendo l'approccio classico Unix:
> **comprendi → configura → convalida → continua**.

## Avvia l'installazione
Inizia con la ISO di Void Linux (x86_64 glibc o musl).

1. Accedi come root
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

4. Connettersi a Internet
- Per **Wi-Fi** *(se via cavo, salta questo passaggio)*:
```bash
wpa_passphrase "WIFI_NETWORK_NAME" "NETWORK_PASSWORD" > wifi.conf
wpa_supplicant -B -i wlan0 -c wifi.conf
dhcpcd wlan0
```
> 📌 **Nota:** `wlan0` può variare (`wlp2s0`, `wlp0s3`, ecc.).
> Utilizzare il comando seguente per identificare l'interfaccia corretta:
>
> ```bash
> ip -br a
> ```

5. Testare la connessione:
```bash
ping -c3 8.8.8.8
ping -c3 repository-default.voidlinux.org
```

## Abilita l'accesso dell'utente **root** tramite SSH (opzionale).
Questo passaggio si applica solo quando il sistema è in esecuzione in una VM; in caso di boot locale (senza VM), l'installazione può procedere normalmente tramite terminale locale.
- Ciò è necessario per accedere alla **VM dall'host** e continuare l'installazione da remoto; successivamente i comandi possono essere incollati/eseguiti direttamente nel terminale tramite SSH.

1. Configura ssh
```bash
echo 'PermitRootLogin sì' >> /etc/ssh/sshd_config
```
2. Riavviare il servizio ssh
```bash
sv riavvia sshd
```

3. Visualizza l'IP dell'interfaccia di rete
```bash
ip -4 percorso ottieni 1.1.1.1 | awk '{stampa $7}'
```
>Annotare l'IP dell'interfaccia di rete e utilizzarlo per connettersi alla VM tramite SSH.

4. Accedi alla VM tramite SSH dall'host.
```bash
sudo ssh <ip-da-vm>
```
> Password predefinita: `voidlinux`

## Configura un prompt colorato nel terminale (opzionale)
Verrà visualizzato l'utente, l'host, il percorso corrente e lo stato dell'ultimo comando:
```bash
esporta PROMPT_COMMAND='RET=$?'
esporta PS1='\[\e[1;33m\]\u\[\e[0m\]@\[\e[1;35m\]\h\[\e[0m\]:\[\e[0;37m\]\w\[\e[0m\] \[\e[1;32m\]$( [ $RET -eq 0 ] && printf ✔ || printf "\e[1;31m✘$RET" )\[\e[0m\] \$ '
```
> 📌 Questo suggerimento è valido solo per la sessione corrente; per renderlo permanente aggiungilo a `.bashrc`.

## Installa i pacchetti richiesti
⚠️ **IMPORTANTE:**
```bash
xbps-install -Sy xbps parted nano vim zstd xz bash-completion
```

## Partizionare il disco
1. Identificare il disco
```bash
fdisk -l | grep -E '^(Disco|Disco) '
```
> Per il tutorial assumeremo `/dev/sda`

2. Modificare le variabili seguenti in base al disco che verrà utilizzato (**IMPORTANTE**):
```bash
# Dischi SATA/SCSI (sdX)
esporta DISPOSITIVO=/dev/sda
esporta DEV_EFI=${DEVICE}2
esporta DEV_ROOT=${DEVICE}3
```

> 📌 **Nota:**
> Per i dischi **NVMe**, il suffisso della partizione cambia (`p`):
> ```bash
> esporta DISPOSITIVO=/dev/nvme0n1
> esporta DEV_EFI=${DEVICE}p2
> esporta DEV_RAIZ=${DEVICE}p3
> ```

3. Partizionare il disco utilizzando **parted** (modalità automatica).
Questo schema crea:
- Partizione BIOS (bios_grub)
- Partizione EFI (ESP)
- Partizione root (ROOT)
```bash
wipefs -a "${DEVICE}"
parted --script "${DEVICE}" -- \
etichetta mkgpt \
mkpart primario 1MiB 2MiB nome 1 BIOS impostato 1 bios_grub su \
mkpart primario fat32 2MiB 514MiB nome 2 EFI set 2 esp on \
mkpart primario 514MiB 100% nome 3 ROOT \
allineamento-controllo ottimale 1 \
allineamento-controllo ottimale 2 \
allineamento-controllo ottimale 3
parted --script "${DEVICE}" -- stampa
```

## Formatta le partizioni
```bash
# Formatta la partizione root (ext4)
mkfs.ext4 -F ${DEV_RAIZ}

# Formatta la partizione EFI (FAT32)
mkfs.fat -F32 -I ${DEV_EFI}
```

## Monta i volumi in `/mnt`
```bash
# Monta la partizione root
monta ${DEV_RAIZ} /mnt

# Crea i punti di montaggio necessari
/t /mnt/{hame,boot/efi,var/log,var/cache, proced, proc, Proc,)

# Montare la partizione EFI
montare ${DEV_EFI} /mnt/boot/efi
```

## Installa il sistema di base
Installa il sistema base Void Linux nell'ambiente montato `/mnt`, inclusi kernel, firmware, bootloader, rete e strumenti essenziali.
```bash
xbps-install -Sy -R https://repo-default.voidlinux.org/current \
-r /mnt \
sistema base e2fsprogs grub-x86_64-efi dracut linux \
intestazioni-linux firmware-linux rete-firmware-linux glibc-locales \
xtools dhcpcd openssh vim nano grc zstd xz bash-completion vpm vsv \
socklog-void wget net-tools tmate ncurses chrony
```

> 📌 **Nota:**
> - `grub-x86_64-efi` → bootloader UEFI
> - `linux` → kernel
> - `linux-firmware-network` → driver di rete
> - `xtools` → richiesto per usare `xgenfstab` senza errori

## Crea `fstab`
Genera automaticamente il file di montaggio permanente del sistema.
```bash
xgenfstab -U /mnt > /mnt/etc/fstab
```

## Entrar e sistema (chroot)
Accedi al sistema installato in "/mnt" per continuare la configurazione.
```bash
xchroot /mnt /bin/bash
```

## Genera INITRAMFS
Configurazione Dracut per ambienti di virtualizzazione (VM-safe)
```bash
cat > /etc/dracut.conf.d/99-vm-safe.conf << 'EOF'
# /etc/dracut.conf.d/99-vm-safe.conf
solo host=no
comprimere="gzip"
add_drivers+=" virtio virtio_pci virtio_blk virtio_net virtio_scsi "
EOF
```

Rileva automaticamente la versione del kernel installata e genera il corrispondente `initramfs` utilizzando **dracut**.
```bash
mods=(/usr/lib/modules/*)
KVER=$(nomebase "${mods[0]}")
echo ${KVER}
dracut --force --kver ${KVER}
```

## Configura GRUB

> 📌 Entrambi i metodi (BIOS e UEFI) sono installati appositamente.
> Ciò consente l'avvio dello stesso disco su entrambi i sistemi **Legacy BIOS** e **UEFI**, aumentando la portabilità tra le macchine.

1. Crea la directory di supporto di GRUB:
```bash
mkdir -p /boot/grub
```

2. Installa GRUB per **BIOS (legacy)**:
```bash
grub-install --target=i386-pc ${DEVICE}
```

3. Installa GRUB per **UEFI**:
```bash
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=void
```

4. Creare il fallback UEFI (avvio universale).
Questo file garantisce l'avvio anche se la NVRAM viene cancellata:
```bash
mkdir -p /boot/efi/EFI/BOOT
cp -f /boot/efi/EFI/void/grubx64.efi /boot/efi/EFI/BOOT/BOOTX64.EFI
```

5. Genera il file di configurazione GRUB finale:
```bash
grub-mkconfig -o /boot/grub/grub.cfg
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

## Impostazioni di base
```bash
# Imposta il nome host
echo void > /etc/nomehost

# Imposta l'ora locale
ln -sf /usr/share/zoneinfo/America/Sao_Paulo /etc/localtime

# Setar Locale
sed -i 's/#en_US.UTF-8/en_US.UTF-8/' /etc/default/libc-locales
sed -i 's/#pt_BR.UTF-8/pt_BR.UTF-8/' /etc/default/libc-locales

# Genera localizzazioni:
xbps-reconfigure -f glibc-locales

# Correggi il possibile errore nel collegamento simbolico /var/service (importante):
rm -f /var/servizio
ln -sf /etc/runit/runsvdir/default /var/service

# Attiva alcuni servizi:
ln -sf /etc/sv/dbus /var/servizio/
ln -sf /etc/sv/dhcpcd /var/service/
ln -sf /etc/sv/sshd /var/servizio/
ln -sf /etc/sv/nanoklogd /var/service/
ln -sf /etc/sv/socklog-unix /var/service/
ln -sf /etc/sv/chronyd /var/service/

# scarica svlogtail personalizzato (opzionale, ma consigliato):
wget --quiet --no-check-certificate -O /usr/bin/svlogtail \
"https://raw.githubusercontent.com/voidlinux-br/void-install/refs/heads/main/svlogtail" && \
chmod +x /usr/bin/svlogtail

# Crea un resolv.conf
printf 'server dei nomi 1.1.1.1\nserver dei nomi 8.8.8.8\n' > /etc/resolv.conf

#Configure sudo - wheel group (facoltativo, ma consigliato)
cat << 'EOF' > /etc/sudoers.d/g_wheel
%wheel ALL=(ALL:ALL) NOPASSWD: TUTTI
EOF

#Autorizzazioni richieste
chmod 440 /etc/sudoers.d/g_wheel
```

## Personalizza `/etc/xbps.d/00-repository-main.conf`
*(Facoltativo, ma consigliato)*

Crea la directory di configurazione **XBPS** (se non esiste già) e definisce un elenco di repository ufficiali e alternativi.
I repository **repo-fastly** tendono ad avere una latenza migliore.

```bash
mkdir -pv /etc/xbps.d

cat << 'EOF' > /etc/xbps.d/00-repository-main.conf
# Repository ufficiale (Fastly – migliore latenza)
archivio=https://repo-fastly.voidlinux.org/current
#repository=https://repo-fastly.voidlinux.org/current/nonfree
#repository=https://repo-fastly.voidlinux.org/current/multilib
#repository=https://repo-fastly.voidlinux.org/current/multilib/nonfree

# Repository alternativo (Chili Linux)
archivio=https://void.chililinux.com/voidlinux/current
#repository=https://void.chililinux.com/voidlinux/current/extras
#repository=https://void.chililinux.com/voidlinux/current/nonfree
#repository=https://void.chililinux.com/voidlinux/current/multilib
#repository=https://void.chililinux.com/voidlinux/current/multilib/nonfree
EOF
```

## Personalizza `/etc/rc.conf`
Imposta il fuso orario, il layout della tastiera e il carattere predefiniti della console.
Cambia secondo necessità.
```bash
cat << 'EOF' > /etc/rc.conf
FUSO ORARIO=America/San_Paolo
MAPPATASTI=br-abnt2
FONT=Lat2-Terminus16
EOF
```

Moduli Virtio (macchina virtuale).
```bash
cat > /etc/modules-load.d/virtio.conf << 'EOF'
virtuoso
virtio_pci
virtio_net
virtio_blk
virtio_scsi
EOF
```

## Personalizza l'utente `.bashrc`
Crea un `.bash_profile` predefinito e garantisce che `.bashrc` venga caricato automaticamente all'accesso.
> ⚠️ Assicurati che l'utente sia stato creato nel passaggio precedente.

```bash
# Scarica il file .bashrc predefinito in /etc/skel
wget --quiet --no-check-certificate \
-O /etc/skel/.bashrc \
"https://raw.githubusercontent.com/voidlinux-br/void-install/refs/heads/main/.bashrc"

chown root:root /etc/skel/.bashrc
chmod 644 /etc/skel/.bashrc

# Crea il .bash_profile predefinito
cat << 'EOF' > /etc/skel/.bash_profile
# ~/.bash_profile — carica .bashrc in Void

# Se esiste .bashrc, caricalo
if [ -f ~/.bashrc ]; Poi
fonte ~/.bashrc
Essere
EOF

# Copia su root e utente
for d in /root "/home/${NEWUSER}"; Fare
cp -f /etc/skel/.bash_profile "$d/"
cp -f /etc/skel/.bashrc "$d/"
Fatto

# Modifica le autorizzazioni dell'utente
chown "${NEWUSER}:${NEWUSER}" \
"/home/${NEWUSER}/.bash_profile" \
"/home/${NEWUSER}/.bashrc"

chmod 644 \
"/home/${NEWUSER}/.bash_profile" \
"/home/${NEWUSER}/.bashrc"
```

## Configura SSH
*(Facoltativo, ma consigliato)*

Crea un file di configurazione supplementare per **sshd**, lasciando intatto il file principale.
```bash
mkdir -pv /etc/ssh/sshd_config.d

cat << 'EOF' > /etc/ssh/sshd_config.d/10-custom.conf
# Impostazioni generali
PermessoTTY sì
PrintMotd sì
StampaUltimoLog sì
Banner /etc/issue.net

# Autenticazione
Permetti RootLogin sì
Autenticazione password sì
KbdInteractiveAuthentication sì
ChallengeResponseAutenticazione sì
Autenticazione Pubkey sì
PubkeyAcceptedKeyTypes=+ssh-rsa
AuthorizedKeysFile .ssh/authorized_keys
Usa PAM sì

# Caratteristiche
X11Inoltro sì
Sottosistema sftp interno-sftp
EOF
```
> ⚠️ Si consiglia di rivedere e rafforzare queste impostazioni SSH dopo il primo avvio, soprattutto sui sistemi esposti a Internet.

## Finalizza l'installazione
Sair fa chroot
```bash
Uscita
```
Smonta tutte le partizioni montate su `/mnt` (compresi i sottovolumi e `/boot/efi`):
```bash
smonta -R /mnt
```
Riavviare la macchina fisica o la VM per testare l'avvio effettivo:
```bash
riavviare
```
> 📌 **Nota: non dimenticare di rimuovere il supporto di installazione.

# 🎉 Buon divertimento!
**Void Linux** è ora installato e pronto per l'uso.

# DISCLAIMER

> Questo tutorial è gratuito: puoi utilizzare, copiare, modificare e ridistribuire come desideri.
> Il contenuto è reso disponibile sotto la **Licenza MIT** e può includere estratti o comandi derivati da software open source, soggetti alle rispettive licenze.
>
> Non viene fornita alcuna garanzia: qui tutto viene consegnato **"così com'è"**.
> Utilizzare a proprio rischio. Né l'autore, né i contributori, né Void Linux sono responsabili per perdite, danni, guasti di sistema o qualsiasi conseguenza derivante dall'uso di questo materiale.
>
> Sei libero di rivedere, adattare e generare la tua versione di questo tutorial.

