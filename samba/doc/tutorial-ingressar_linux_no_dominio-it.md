# Configurazione LightDM per unire Linux Mint al dominio

## 🎯 Obiettivo Unire Linux al dominio Samba4 con Winbind attraverso l'integrazione NSS/PAM

---

## Prerequisiti

- Samba4 come controller di dominio (PDC)
- Linux con DNS e orario allineato con PDC (192.168.70.253)
- Connettività al server

---

## 🛠️Passi

## 1. Installa i pacchetti richiesti

```bash
sudo apt update && sudo apt install samba winbind libpam-winbind libnss-winbind krb5-user
```

## 2. Configurare /etc/krb5.conf

```bash
[libdefaults]
    default_realm = EDUCATUX.EDU
    dns_lookup_realm = false
    dns_lookup_kdc = true
    kdc_timesync = 1
    ccache_type = 4
    forwardable = true
    proxiable = true
    rdns = false
    fcc-mit-ticketflags = true

[realms]
    EDUCATUX.EDU = {
        kdc = 192.168.70.253
        admin_server = 192.168.70.253
        default_domain = educatux.edu
    }

[domain_realm]
    .officinas.edu = EDUCATUX.EDU
    educatux.edu = EDUCATUX.EDU
```

## 3. Configurare /etc/samba/smb.conf

```bash
[global]
   workgroup = EDUCATUX
   security = ads
   realm = EDUCATUX.EDU
   server role = member server
   interfaces = lo eth0
   bind interfaces only = yes


   winbind use default domain = true
   winbind enum users = yes
   winbind enum groups = yes
   winbind refresh tickets = yes
   winbind offline logon = yes

   idmap config * : backend = tdb
   idmap config * : range = 10000-19999

   idmap config EDUCATUX : backend = rid
   idmap config EDUCATUX : range = 20000-999999

   template shell = /bin/bash
   template homedir = /home/%U
```

## 4. Configurare /etc/nsswitch.conf

```bash
passwd:         compat winbind
group:          compat winbind
shadow:         compat
```

## 5. Unisciti al dominio

```bash
sudo net ads join -U Administrator
```

## 6. Riavvio del sistema

```bash
sudo reboot
```

## 7. Controlli

```bash
sudo net ads testjoin
wbinfo -u
wbinfo -g
getent passwd usuario
```

## 8. Crea automaticamente le directory HOME


## Modifica /etc/pam.d/common-session e aggiungi:

```bash
session required pam_mkhomedir.so skel=/etc/skel umask=0022
```

## 9. Riavviare i servizi

```bash
sudo systemctl restart smbd nmbd winbind
sudo systemctl enable winbind
```

## 10. Sincronizzazione dell'ora

```bash
sudo timedatectl set-ntp true
```

## SE sei un utente Lightdm, come nel caso di Mint, impostalo per accedere con un utente di rete, anziché solo con un utente locale.

## 🛠️ Passo dopo passo: configura LightDM per accettare gli utenti del dominio

## 1. Modifica il file di configurazione di LightDM

```bash
sudo vim /etc/lightdm/lightdm.conf
```

## Modifica le seguenti righe nel file:

```bash
[Seat:*]
greeter-show-manual-login=true
greeter-hide-users=true
allow-guest=false
```

## Spiegazioni:

- greeter-show-manual-login=true: consente di inserire manualmente il nome utente.
- greeter-hide-users=true: nasconde l'elenco locale degli utenti (utile per ambienti aziendali).
- allow-guest=false: impedisce l'accesso ospite (per sicurezza).

## 2. Assicurati che PAM consenta agli utenti del dominio

## Se hai utilizzato SSSD o Winbind, PAM dovrebbe già essere integrato correttamente. Ma assicurati che il modulo home sia presente:

```bash
sudo vim /etc/pam.d/common-session
```

## Conferma che questa riga esiste OPPURE aggiungila:

```bash
session required pam_mkhomedir.so skel=/etc/skel umask=0022
```

## 3. Riavvia LightDM

```bash
sudo timedatectl set-ntp true
```

## ⚠️ Per Linux MInt basato su Ubuntu, puoi disabilitare systemd-resolved per controllare manualmente il DNS.

```bash
systemctl status systemd-resolved
```

```bash
systemctl stop systemd-resolved
systemctl disable systemd-resolved.service
sudo systemctl mask systemd-resolved
```

## RIMOZIONE DEL FILE SENZA PERMESSO DI MODIFICA CREATO DA SYSTEMD-RESOLVED:

```bash
rm -f /etc/resolv.conf
```

## Creazione di un nuovo file con autorizzazione di modifica:

```bash
vim /etc/resolv.conf
```

```bash
domain educatux.edu
search educatux.edu.
nameserver 192.168.70.253
```

## Bloccare il file contro la modifica automatica

```bash
sudo chattr +i /etc/resolv.conf
```

## Riavvio del servizio

```bash
sudo systemctl restart NetworkManager
```

---

🎯 E' TUTTO RAGAZZE!

👉 Contatto: zerolies@disroot.org
👉 https://t.me/z3r0l135

