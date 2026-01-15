
# Serveur de fichiers exécutant Samba4 sur Void Linux Server ;D

## 🎯 Objectif – Déployer un serveur de fichiers sur Void Linux (glibc) en compilant Samba4 à partir des sources, de l'intégration AD, des ACL, des services et de toute la pile requise pour qu'un serveur de fichiers serve les clients du réseau.

## 🔧 Laboratoire de mise en réseau avec QEMU/Virtmanager et Proxmox. Ajustez le didacticiel en fonction de votre propre environnement.

---

## 📡 Disposition du réseau local

- Domaine : EDUCATUX.EDU

- Nom d'hôte : serveur de fichiers

- Pare-feu 192.168.70.254 (DNS/GW)

- IP : 192.168.70.251

## Installer Void Linux

## Changer le shell par défaut sur Void

```bash
chsh -s /bin/bash
```

## 🧩 Installez les packages de dépendances pour compiler Samba4 sur Void

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

## 🖥️ Définir le nom d'hôte

```bash
echo "fileserver" > /etc/hostname
```

## 🏠 /etc/hosts

```bash
vim /etc/hosts
```

## Contenu:

```bash
127.0.0.1      localhost
127.0.1.1      fileserver.educatux.edu fileserver
192.168.70.251 fileserver.educatux.edu fileserver
```

## 🌐 Configurer l'IP statique

## 👉 Nous utiliserons la méthode standard Void, /etc/dhcpcd.conf

```bash
vim /etc/dhcpcd.conf
```

## Ajoutez IP, passerelle (routeur) et DNS (AD) :

```bash
interface eth0
static ip_address=192.168.70.251/24
static routers=192.168.70.254
static domain_name_servers=192.168.70.253
```

## Redémarrez l'interface réseau :

```bash
sv restart dhcpcd
```

## 🧭 Définir l'adresse DNS - Pointez vers le PDC

```bash
vim /etc/resolv.conf
```

## Contenu:

```bash
domain educatux.edu
search educatux.edu
nameserver 192.168.70.253
```

## Verrouiller resolv.conf

```bash
chattr +i /etc/resolv.conf
```

## 🔍 Validez l'adresse de l'interface réseau

```bash
ip -c addr
ip -br link
```

## 📥 Téléchargez et extrayez le code source de Samba4

```bash
wget https://download.samba.org/pub/samba/samba-4.23.4.tar.gz
```

```bash
tar -xvzf samba-4.23.4.tar.gz
```

## Compiler et installer à partir des sources

```bash
cd samba-4.23.4
```

```bash
./configure --prefix=/opt/samba
```

```bash
make -j$(nproc) && make install
```

## Remarques :

- Void n'interfère pas car Samba est compilé dans /opt/samba.

- make -j accélère considérablement la compilation. Allez quand même prendre un café.

- Après l'installation, Samba4 compilé ne dispose d'aucun service d'exécution.

- Nous allons créer les services manuellement.

## 📁 Ajoutez Samba4 au PATH système et rechargez l'environnement

```bash
echo 'export PATH=/opt/samba/bin:/opt/samba/sbin:$PATH' >> /etc/profile
```

```bash
source /etc/profile
```

## Testez l'insertion de Samba4 PATH dans le système d'exploitation

```bash
samba-tool -V
```

## Sortir:

```bash
4.23.4
```

## ⚠️ Attention : N'utilisez PAS la commande provisioning sur le serveur de fichiers !

## 📝 Créez le fichier smb.conf

```bash
vim /opt/samba/etc/smb.conf
```

```bash
[global]
   workgroup = EDUCATUX
   security = ads
   realm = EDUCATUX.EDU
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
   guest ok = no
   create mask = 0660
   directory mask = 0770
``` 

## Créer le fichier journal

```bash
mkdir /opt/samba/var
```

## 📂 Créer le parcours de partage

```bash
sudo mkdir -p /srv/samba/public
sudo chown -R root:root /srv/samba/public
sudo chmod -R 0770 /srv/samba/public
```

## Recharger la configuration de Samba4

```bash
smbcontrol all reload-config
```

## 🕒 Serveur NTP/Chrony

## Le contrôleur de domaine doit être le serveur de temps local, car avec une dérive de 5 minutes, Kerberos n'authentifiera plus les clients.

## Installer Chrony

```bash
xbps-install -Syu chrony
```

## Modifier la configuration et autoriser le réseau interne

```bash
vim /etc/chrony.conf
```

## Définissez le contrôle de domaine sur les serveurs de temps :

```bash
# Comment the external line
#pool pool.ntp.org iburst

# PDC Time Servers
server 192.168.70.253 iburst
```

## Activer Chronyd dans Runit

```bash
ln -sf /etc/sv/chronyd/ /var/service/
```

## Redémarrer le service :

```bash
sv restart chronyd
```

## Valider les serveurs :

```bash
chronyc sources -v
```

## 🔐 Créez le fichier Kerberos - Pointez sur le PDC

```bash
vim /etc/krb5.conf
```

## contenant

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

## Résultat obtenu

```bash
Ticket cache: FILE:/tmp/krb5cc_0
Default principal: administrator@EDUCATUX.EDU

Valid starting       Expires              Service principal
11/12/2025 09:43:40  11/12/2025 19:43:40  krbtgt/EDUCATUX.EDU@EDUCATUX.EDU
	renew until 12/12/2025 09:43:36
```

## 🔗 Lier les bibliothèques Winbind au système

## Validez le chemin libdir :

```bash
/opt/samba/sbin/smbd -b | grep LIBDIR
```

## Attendu:

```bash
LIBDIR: /opt/samba/lib
```

## Créez des liens de bibliothèque (préférez saisir manuellement) :

```bash
ln -sf /opt/samba/lib/libnss_winbind.so.2 /usr/lib/
```

```bash
ln -sf /usr/lib/libnss_winbind.so.2 /usr/lib/libnss_winbind.so
```

## Recharger le cache de la bibliothèque :

```bash
ldconfig
```

## Mettre à jour nsswitch pour l'échange de tickets Kerberos (ajouter winbind) :

```bash
vim /etc/nsswitch.conf
```

```bash
passwd: files winbind
group:  files winbind
```

## Laissez le reste intact

## Rejoindre le domaine

```bash
net ads join -U Administrator
```

## Résultat obtenu

```bash
Password for [EDUCATUX\Administrator]:
Using short domain name -- EDUCATUX
Joined 'VOIDFILES' to dns domain 'educatux.edu'
```

## 📦 Créez les services RUNIT pour smbd, winbindd et éventuellement nmbd

### Void Linux utilise runit comme système d'initialisation et son enregistreur natif est svlogd, inclus dans le package de base runit. Aucun forfait supplémentaire n’est requis.

## SMBD — Service et journalisation

## Créer des répertoires de services et de journaux

```bash
mkdir -p /etc/sv/smbd/log
mkdir -p /var/log/smbd
```

## Créer /etc/sv/smbd/run

```bash
vim /etc/sv/smbd/run
```

## Contenu

```bash
#!/bin/sh
exec 2>&1
exec /opt/samba/sbin/smbd --foreground --no-process-group
```

## Autorisation

```bash
chmod +x /etc/sv/smbd/run
```

## Créer /etc/sv/smbd/log/run

```bash
vim /etc/sv/smbd/log/run
```

## Contenu

```bash
#!/bin/sh
exec svlogd -tt /var/log/smbd
```

## Autorisation

```bash
chmod +x /etc/sv/smbd/log/run
```

## Débogage (facultatif)

```bash
/opt/samba/sbin/smbd -i
```

## Résultat obtenu

```bash
smbd version 4.23.4 started.
Copyright Andrew Tridgell and the Samba Team 1992-2025
daemon 'smbd' : Starting process ...
```

## WINBINDD — Service et journalisation

## Créer des répertoires de services et de journaux

```bash
mkdir -p /etc/sv/winbindd/log
mkdir -p /var/log/winbindd
```

## Créer /etc/sv/winbindd/run

```bash
vim /etc/sv/winbindd/run
```

## Contenu

```bash
#!/bin/sh
exec 2>&1
exec /opt/samba/sbin/winbindd --foreground --no-process-group
```

## Autorisation

```bash
chmod +x /etc/sv/winbindd/run
```

## Créer /etc/sv/winbindd/log/run

```bash
vim /etc/sv/winbindd/log/run
```

## Contenu

```bash
#!/bin/sh
exec 2>&1
exec /opt/samba/sbin/winbindd --foreground --no-process-group
```

## Autorisation

```bash
chmod +x /etc/sv/winbindd/log/run
```

## Débogage (facultatif)

```bash
/opt/samba/sbin/winbindd -i
```

## Résultat obtenu

```bash
winbindd version 4.23.4 started.
Copyright Andrew Tridgell and the Samba Team 1992-2025
daemon 'winbindd' : Starting process ...
```

## NMBD — Service et journalisation (facultatif)

### Activez-la uniquement si votre environnement nécessite la navigation NetBIOS/SMB1.

## Créer des répertoires de services et de journaux

```bash
mkdir -p /etc/sv/nmbd/log
mkdir -p /var/log/nmbd
```

## Créer /etc/sv/nmbd/run

```bash
vim /etc/sv/nmbd/run
```

## Contenu

```bash
#!/bin/sh
exec 2>&1
exec /opt/samba/sbin/nmbd --foreground --no-process-group
```

## Autorisation

```bash
chmod +x /etc/sv/nmbd/run
```

## Créer /etc/sv/nmbd/log/run

```bash
vim /etc/sv/nmbd/log/run
```
 
## Contenu

```bash
#!/bin/sh
exec svlogd -tt /var/log/nmbd
```

## Autorisation

```bash
chmod +x /etc/sv/nmbd/log/run
```

## Activer les services

```bash
ln -sf /etc/sv/smbd /var/service/
ln -sf /etc/sv/winbindd /var/service/
```

## Facultatif - activez uniquement si vous utilisez NetBIOS :

```bash
ln -sf /etc/sv/nmbd /var/service/
```

## Valider les services

```bash
sv status smbd winbindd nmbd
```

## 🧪 Valider l'intégration

```bash
net ads testjoin
```

## Résultat obtenu

```bash
Join is OK
```

```bash
wbinfo -u
```

## Résultat obtenu

```bash
guest
krbtgt
administrator```

```bash
wbinfo -g
```

## Result obtained

```bash
contrôleurs de domaine d'entreprise en lecture seule
utilisateurs protégés
contrôleurs de domaine
invités du domaine
contrôleurs de domaine en lecture seule
administrateurs de schéma
proxy de mise à jour DNS
administrateurs de domaine
propriétaires de créateurs de stratégie de groupe
serveurs ras et ias
administrateurs DNS
groupe de réplication de mot de passe rodc autorisé
administrateurs d'entreprise
éditeurs de certificats
utilisateurs du domaine
groupe de réplication de mot de passe rodc refusé
ordinateurs du domaine
```

```bash
wbinfo --ping-dc
```

## Result obtained

```bash
la vérification du NETLOGON pour la connexion DC du domaine [EDUCATUX] à "voiddc.educatux.edu" a réussi
```

## ✅ FINAL SUMMARY

## 🎉 Congratulations — you have successfully deployed a fully functional File Server on Void Linux!

## 👉 REMEMBER: While Samba4 can be managed via CLI, it was designed to be managed via RSAT Remote Server Administration Tools, which can be installed on a Windows 10 machine without issues!

---

🎯 THAT'S ALL FOLKS!

👉 Contact: zerolies@disroot.org

👉 https://t.me/z3r0l135
