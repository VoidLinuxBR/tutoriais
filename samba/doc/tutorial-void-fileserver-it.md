
# File server che esegue Samba4 su Void Linux Server ;D

## 🎯 Obiettivo: distribuire un file server su Void Linux (glibc) compilando Samba4 dal sorgente, dall'integrazione AD, dalle ACL, dai servizi e dall'intero stack richiesto affinché un file server possa servire i client di rete.

## 🔧 Laboratorio di networking con QEMU/Virtmanager e Proxmox. Modifica il tutorial per adattarlo al tuo ambiente.

---

## 📡 Layout della rete locale

- Dominio: EDUCATUX.EDU

- Nome host: file server

- Firewall 192.168.70.254 (DNS/GW)

- IP: 192.168.70.251

## Installa VoidLinux

## Cambia la shell predefinita su Void

```bash
chsh -s /bin/bash
```

## 🧩 Installa i pacchetti di dipendenze per compilare Samba4 su Void

```bash
xbps-install -S \
 net-tools rsync acl attr attr-devel autoconf automake libtool \
 binutils bison gcc make ccache chrpath curl \
 docbook-xml docbook-xsl flex gdb git htop \
 mit-krb5 mit-krb5-client mit-krb5-devel \
 libarchive-devel avahi avahi-libs libblkid-devel \
 libbsd-devel libcap-devel cups-devel dbus-devel glib-devel \
 gnutls-devel gpgme-devel icu-devel jansson-devel \
 lmdb lmdb-devel libldap-devel ncurses-devel pam-devel perl \
 perl-Text-ParseWords perl-JSON perl-Parse-Yapp \
 libpcap-devel popt-devel readline-devel \
 libtasn1 libtasn1-devel libunwind-devel python3 python3-devel \
 python3-dnspython python3-cryptography \
 python3-matplotlib python3-pexpect python3-pyasn1 \
 tree libuuid-devel wget xfsprogs-devel zlib-devel \
 bind ldns pkg-config vim
```

## 🖥️ Imposta il nome host

```bash
echo "fileserver" > /etc/hostname
```

## 🏠 /etc/hosts

```bash
vim /etc/hosts
```

## Contenuto:

```bash
127.0.0.1      localhost
127.0.1.1      fileserver.educatux.edu fileserver
192.168.70.251 fileserver.educatux.edu fileserver
```

## 🌐 Configura IP statico

## 👉 Utilizzeremo il metodo Void standard, /etc/dhcpcd.conf

```bash
vim /etc/dhcpcd.conf
```

## Aggiungi IP, gateway (Router) e DNS (AD):

```bash
interface eth0
static ip_address=192.168.70.251/24
static routers=192.168.70.254
static domain_name_servers=192.168.70.253
```

## Riavviare l'interfaccia di rete:

```bash
sv restart dhcpcd
```

## 🧭 Imposta l'indirizzo DNS: punta al PDC

```bash
vim /etc/resolv.conf
```

## Contenuto:

```bash
domain educatux.edu
search educatux.edu
nameserver 192.168.70.253
```

## Blocca resolv.conf

```bash
chattr +i /etc/resolv.conf
```

## 🔍 Convalida l'indirizzo dell'interfaccia di rete

```bash
ip -c addr
ip -br link
```

## 📥 Scarica ed estrai il codice sorgente di Samba4

```bash
wget https://download.samba.org/pub/samba/samba-4.23.4.tar.gz
```

```bash
tar -xvzf samba-4.23.4.tar.gz
```

## Compilare e installare dal sorgente

```bash
cd samba-4.23.4
```

```bash
./configure --prefix=/opt/samba
```

```bash
make -j$(nproc) && make install
```

## Note:

- Void non interferisce perché Samba è compilato in /opt/samba.

- make -j velocizza notevolmente la compilazione, comunque vai a prendere un caffè.

- Dopo l'installazione, Samba4 compilato non dispone di alcun servizio runit.

- Creeremo i servizi manualmente.

## 📁 Aggiungi Samba4 al PATH del sistema e ricarica l'ambiente

```bash
echo 'export PATH=/opt/samba/bin:/opt/samba/sbin:$PATH' >> /etc/profile
```

```bash
source /etc/profile
```

## Testare l'inserimento del PATH Samba4 nel sistema operativo

```bash
samba-tool -V
```

## Produzione:

```bash
4.23.4
```

## ⚠️ Attenzione: NON utilizzare il comando di provisioning sul File Server!

## 📝 Crea il file smb.conf

```bash
vim /opt/samba/etc/smb.conf
```

```bash
[global]
   workgroup = EDUCATUX
   security = ads
   realm = EDUCATUX.EDU
   netbios name = fileserver
   encrypt passwords = yes
   # point to the services, the active interfaces
   interfaces = eth0
   bind interfaces only = yes

   log file = /opt/samba/var/log.%m
   max log size = 50

   winbind use default domain = yes
   winbind enum users = yes
   winbind enum groups = yes
   winbind refresh tickets = yes

   idmap config * : backend = tdb
   idmap config * : range = 3000-7999
   idmap config EDUCATUX : backend = rid
   idmap config EDUCATUX : range = 10000-999999

   template shell = /bin/bash
   template homedir = /home/%U

[Public]
   path = /srv/samba/public
   browsable = yes
   writable = yes
``` 

## Creare il file di registro

```bash
mkdir /opt/samba/var
```

## 📂Crea il percorso di condivisione

```bash
sudo mkdir -p /srv/samba/arquivos
sudo chown -R root:"Domain Admins" /srv/samba/arquivos
sudo chmod -R 0770 /srv/samba/arquivos
```

## Ricarica la configurazione di Samba4

```bash
smbcontrol all reload-config
```

## 🕒Server NTP/Crony

## Il Domain Controller deve essere il Time Server locale, perché con uno scostamento di 5 minuti Kerberos non autenticherà più i client.

## Installa Chrony

```bash
xbps-install -Syu chrony
```

## Modifica la configurazione e consenti la rete interna

```bash
vim /etc/chrony.conf
```

## Imposta il controllo del dominio sui time server:

```bash
# Comment the external line
#pool pool.ntp.org iburst

# PDC Time Servers
server 192.168.70.253 iburst
```

## Abilita chronyd in runit

```bash
ln -sf /etc/sv/chronyd/ /var/service/
```

## Riavvia il servizio:

```bash
sv restart chronyd
```

## Convalida i server:

```bash
chronyc sources -v
```

## 🔐 Crea il file Kerberos: punta al PDC

```bash
vim /etc/krb5.conf
```

## contenente

```bash
[libdefaults]
    default_realm = EDUCATUX.EDU
    dns_lookup_realm = true
    dns_lookup_kdc = true
    rdns = false
    forwardable = true
    proxiable = true

[realms]
    EDUCATUX.EDU = {
        kdc = 192.168.70.253
        admin_server = 192.168.70.253
        default_domain = educatux.edu
    }

[domain_realm]
    .educatux.edu = EDUCATUX.EDU
    educatux.edu = EDUCATUX.EDU
```

## Test Kerberos

```bash
kinit Administrator
```

```bash
klist
```

## Risultato ottenuto

```bash
Ticket cache: FILE:/tmp/krb5cc_0
Default principal: administrator@EDUCATUX.EDU

Valid starting       Expires              Service principal
11/12/2025 09:43:40  11/12/2025 19:43:40  krbtgt/EDUCATUX.EDU@EDUCATUX.EDU
	renew until 12/12/2025 09:43:36
```

## 🔗 Collega le librerie Winbind al sistema

## Convalida il percorso della directory lib:

```bash
/opt/samba/sbin/smbd -b | grep LIBDIR
```

## Previsto:

```bash
LIBDIR: /opt/samba/lib
```

## Crea collegamenti alla libreria (preferisci digitare manualmente):

```bash
ln -sf /opt/samba/lib/libnss_winbind.so.2 /usr/lib/
```

```bash
ln -sf /usr/lib/libnss_winbind.so.2 /usr/lib/libnss_winbind.so
```

## Ricarica la cache della libreria:

```bash
ldconfig
```

## Aggiorna nsswitch per lo scambio di ticket Kerberos (aggiungi winbind):

```bash
vim /etc/nsswitch.conf
```

```bash
passwd: files winbind
group:  files winbind
```

## Lascia intatto il resto

## Unisciti al dominio

```bash
net ads join -U Administrator
```

## Risultato ottenuto

```bash
Password for [EDUCATUX\Administrator]:
Using short domain name -- EDUCATUX
Joined 'VOIDFILES' to dns domain 'educatux.edu'
```

## 📦 Crea i servizi RUNIT per smbd, winbindd e facoltativamente nmbd

### Void Linux utilizza runit come sistema init e il suo logger nativo è svlogd, incluso nel pacchetto base runit. Non sono richiesti pacchetti aggiuntivi.

## SMBD: servizio e registrazione

## Creare directory di servizi e log

```bash
mkdir -p /etc/sv/smbd/log
mkdir -p /var/log/smbd
```

## Crea /etc/sv/smbd/run

```bash
vim /etc/sv/smbd/run
```

## Contenuto

```bash
#!/bin/sh
exec 2>&1
exec /opt/samba/sbin/smbd --foreground --no-process-group
```

## Autorizzazione

```bash
chmod +x /etc/sv/smbd/run
```

## Crea /etc/sv/smbd/log/run

```bash
vim /etc/sv/smbd/log/run
```

## Contenuto

```bash
#!/bin/sh
exec svlogd -tt /var/log/smbd
```

## Autorizzazione

```bash
chmod +x /etc/sv/smbd/log/run
```

## Debug (facoltativo)

```bash
/opt/samba/sbin/smbd -i
```

## Risultato ottenuto

```bash
smbd version 4.23.4 started.
Copyright Andrew Tridgell and the Samba Team 1992-2025
daemon 'smbd' : Starting process ...
```

## WINBINDD: servizio e registrazione

## Creare directory di servizi e log

```bash
mkdir -p /etc/sv/winbindd/log
mkdir -p /var/log/winbindd
```

## Crea /etc/sv/winbindd/run

```bash
vim /etc/sv/winbindd/run
```

## Contenuto

```bash
#!/bin/sh
exec 2>&1
exec /opt/samba/sbin/winbindd --foreground --no-process-group
```

## Autorizzazione

```bash
chmod +x /etc/sv/winbindd/run
```

## Crea /etc/sv/winbindd/log/run

```bash
vim /etc/sv/winbindd/log/run
```

## Contenuto

```bash
#!/bin/sh
exec 2>&1
exec /opt/samba/sbin/winbindd --foreground --no-process-group
```

## Autorizzazione

```bash
chmod +x /etc/sv/winbindd/log/run
```

## Debug (facoltativo)

```bash
/opt/samba/sbin/winbindd -i
```

## Risultato ottenuto

```bash
winbindd version 4.23.4 started.
Copyright Andrew Tridgell and the Samba Team 1992-2025
daemon 'winbindd' : Starting process ...
```

## NMBD: servizio e registrazione (facoltativo)

### Abilitalo solo se il tuo ambiente richiede la navigazione NetBIOS/SMB1.

## Creare directory di servizi e log

```bash
mkdir -p /etc/sv/nmbd/log
mkdir -p /var/log/nmbd
```

## Crea /etc/sv/nmbd/run

```bash
vim /etc/sv/nmbd/run
```

## Contenuto

```bash
#!/bin/sh
exec 2>&1
exec /opt/samba/sbin/nmbd --foreground --no-process-group
```

## Autorizzazione

```bash
chmod +x /etc/sv/nmbd/run
```

## Crea /etc/sv/nmbd/log/run

```bash
vim /etc/sv/nmbd/log/run
```
 
## Contenuto

```bash
#!/bin/sh
exec svlogd -tt /var/log/nmbd
```

## Autorizzazione

```bash
chmod +x /etc/sv/nmbd/log/run
```

## Abilita servizi

```bash
ln -sf /etc/sv/smbd /var/service/
ln -sf /etc/sv/winbindd /var/service/
```

## Facoltativo: abilita solo se utilizzi NetBIOS:

```bash
ln -sf /etc/sv/nmbd /var/service/
```

## Convalidare i servizi

```bash
sv status smbd winbindd nmbd
```

## 🧪 Convalida l'integrazione

```bash
net ads testjoin
```

## Risultato ottenuto

```bash
Join is OK
```

```bash
wbinfo -u
```

## Risultato ottenuto

```bash
guest
krbtgt
administrator```

```bash
wbinfo -g
```

## Result obtained

```bash
controller di dominio aziendali di sola lettura
utenti protetti
controller di dominio
ospiti del dominio
controller di dominio di sola lettura
amministratori dello schema
dnsupdateproxy
amministratori di dominio
proprietari dei creatori dei criteri di gruppo
server ras e ias
dnsadmins
gruppo di replica password rodc consentito
amministratori aziendali
editori di certificati
utenti del dominio
gruppo di replica della password rodc negato
computer del dominio
```

```bash
wbinfo --ping-dc
```

## Result obtained

```bash
il controllo del NETLOGON per la connessione dc del dominio [EDUCATUX] a "voiddc.educatux.edu" è riuscito
```

## ✅ FINAL SUMMARY

## 🎉 Congratulations — you have successfully deployed a fully functional File Server on Void Linux!

## 👉 REMEMBER: While Samba4 can be managed via CLI, it was designed to be managed via RSAT Remote Server Administration Tools, which can be installed on a Windows 10 machine without issues!

---

🎯 THAT'S ALL FOLKS!

👉 Contact: zerolies@disroot.org

👉 https://t.me/z3r0l135
