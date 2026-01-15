# Contrôleur de domaine principal (Active Directory) exécutant Samba4 sous Void Linux Server ;D

## 🎯 Objectif - Télécharger un contrôleur de domaine principal sur Void Linux (glibc) compilant Samba4 à partir du code source, configurant le DNS interne, Kerberos, l'intégration AD, les ACL, les services et toute la pile nécessaire au contrôle des clients réseau.

### 🔧 ADAPTEZ le tutoriel à VOTRE réalité, évidemment !

## 📡 Disposition du réseau local

- Domaine : EDUCATUX.EDU
- Nom d'hôte : pdc01
- Pare-feu 192.168.70.254 (DNS/GW)
- IP : 192.168.70.253

---

## L'installation par défaut de Void Linux ne sera pas abordée dans ce tutoriel.

## Changer le shell par défaut de Void, après l'installation

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

## ⚠️ ATTENTION : Le Samba4 compilé inclut le code Heimdal Kerberos, intégré (KDC interne) par défaut, mais n'inclut pas les clients Kerberos. Dans ce cas, le référentiel fournit des packages binaires du MIT, qui peuvent être installés sans aucun problème ni interférence avec le kerberos heimdal par défaut, compilé sur le contrôleur de domaine. Les packages sont : mit-krb5 mit-krb5-client mit-krb5-devel. CEPENDANT, vous NE DEVEZ en aucun cas installer le paquet binaire krb5-server depuis le dépôt, ce qui provoquerait un service concurrent avec les kerberos Heimdal, internes à Samba4 !

## Les services fournis par les clients du MIT-krb5 sont les suivants :

```bash
/usr/bin/kinit
/usr/bin/klist
/usr/bin/kvno
/usr/bin/kdestroy
```

## 🖥️ Nom d'hôte Setar

```bash
echo "pdc01" > /etc/hostname
```

## 🏠 /etc/hosts

```bash
vim /etc/hosts
```

## Contenu:

```bash
127.0.0.1      localhost
127.0.1.1      pdc01.educatux.edu pdc01
192.168.70.253 pdc01.educatux.edu pdc01
```

## 🌐 Configurer l'IP fixe

### 👉 Nous utiliserons la méthode par défaut de Void, /etc/dhcpcd.conf

```bash
vim /etc/dhcpcd.conf
```

## Ajouter une IP, une passerelle et un DNS :## 🎯 Objectif

```bash
interface eth0
static ip_address=192.168.70.253/24
static routers=192.168.70.254
static domain_name_servers=192.168.70.254
```

## Redémarrez l'interface réseau :

```bash
sv restart dhcpcd
```

## 🌐 Définir un DNS temporaire (routeur) AVANT le provisionnement

```bash
echo "nameserver 192.168.70.254" > /etc/resolv.conf
```

## Verrouiller la configuration de resolv.conf

```bash
chattr +i /etc/resolv.conf
```

## 🔍 Valider l'adresse attribuée à l'interface réseau

```bash
ip -c addr
```

```bash
ip -br link
```

## 📥 Téléchargez et décompressez le code source de Samba4

```bash
wget https://download.samba.org/pub/samba/samba-4.23.4.tar.gz
```

```bash
tar -xvzf samba-4.23.4.tar.gz
```

## Compiler et installer le code source

```bash
cd samba-4.23.4
```

```bash
./configure --prefix=/opt/samba
```

```bash
make -j$(nproc) && make install
```

## Commentaire:

- Void n'interfère pas avec l'installation, car Samba est compilé dans /opt/samba.
- Make -j accélère beaucoup la compilation, de toute façon, va prendre un café.
- Après l'installation, le Samba4 compilé n'a aucun service créé dans le runit.
- Nous allons créer les services manuellement.

## 📁 Ajoutez Samba4 au PATH Système et relisez l'environnement

```bash
echo 'export PATH=/opt/samba/bin:/opt/samba/sbin:$PATH' >> /etc/profile
```

```bash
source /etc/profile
```

## Tester l'insertion du Samba4 PATH dans le système d'exploitation

```bash
samba-tool -V
```

## Résultat:

```bash
4.23.4
```

🏰 Provisionner le domaine SAMBA4 (Création du PDC lui-même)

```bash
samba-tool domain provision \
 --realm=educatux.edu \
 --domain=EDUCATUX \
 --use-rfc2307 \
 --dns-backend=SAMBA_INTERNAL \
 --server-role=dc \
 --adminpass='P@ssw0rd' \
 --option="ad dc functional level = 2016" \
 --function-level=2016
```

### Samba4 créera :

```bash
/opt/samba/etc/smb.conf
/opt/samba/private/*
/opt/samba/var/locks/sysvol
```

## En résumé Samba4 :

- Crée la forêt AD, le contrôleur de domaine principal, le DNS interne et la base de données des comptes.
- Définit le domaine, le domaine, le niveau fonctionnel 2016 et le mot de passe administrateur.
- Void n'installe aucun Samba natif, il n'y a donc pas de conflit.
- Après cela, le DNS devient le PDC lui-même, devant ajuster /etc/resolv.conf à 127.0.0.1.

## ⚙️ Valider le niveau fonctionnel d'Active Directory 2016

```bash
samba-tool domain level show
```

## Résultat:

```bash
Domain and forest function level for domain 'DC=educatux,DC=edu'
Forest function level: (Windows) 2016
Domain function level: (Windows) 2016
Lowest function level of a DC: (Windows) 2016
```

## 🧪 Testez manuellement le service AD DC avant de créer le service

```bash
/opt/samba/sbin/samba -i -M single
```

* -i → premier plan
* -M modèle unique → modèle à processus unique (ne déclenche pas de fork de démon en dehors du contrôle runit)

## Si tout va bien, vous verrez :

```bash
Completed DNS update check OK
Completed SPN update check OK
Registered EDUCATUX<1c> ...
```

## CTRL+C pour quitter

## 📦 Créez le service RUNIT samba-ad-dc pour télécharger AD au démarrage

## ⚠️ Cette partie est très importante. Supprimez les anciens restes SI RÉAJUSTEZ un serveur préexistant !!

```bash
sv stop samba-ad-dc 2>/dev/null
rm -rf /var/service/samba-ad-dc
rm -rf /var/service/a-ad-dc
rm -rf /etc/sv/samba-ad-dc
rm -rf /etc/sv/a-ad-dc
rm -rf /etc/sv/*/supervise
rm -rf /var/service/*/supervise
```

## Créons maintenant les services et autorisations samba-ad-dc avec les journaux, afin que runit puisse être chargé au démarrage du système :

## Créer avant tout la structure du service

```bash
mkdir -p /etc/sv/samba-ad-dc/log
mkdir -p /var/log/samba-ad-dc
```

## Créer le service d'exécution

```bash
cat > /etc/sv/samba-ad-dc/run << 'EOF'
#!/bin/sh
exec 2>&1
exec /opt/samba/sbin/samba -i -M single --debuglevel=3
EOF
```

## Définir l'autorisation d'exécution du service

```bash
chmod +x /etc/sv/samba-ad-dc/run
```

## Créer le fichier journal

```bash
cat > /etc/sv/samba-ad-dc/log/run << 'EOF'
#!/bin/sh
exec svlogd -tt /var/log/samba-ad-dc
EOF
```

## Définir l'autorisation de journalisation/exécution

```bash
chmod +x /etc/sv/samba-ad-dc/log/run
```

## Activez le service samba-ad-dc pour qu'il s'exécute au démarrage :

```bash
ln -sf /etc/sv/samba-ad-dc/ /var/service/
```

## Validez s'il est en cours d'exécution

```bash
sv status samba-ad-dc
```

## Vous devriez voir quelque chose comme :

```bash
run: samba-ad-dc: (pid 28032) 4s; run: log: (pid 28031) 4s
```

```bash
samba-tool processes
```

## Résultat reçu :

```bash
 Service:                          PID
--------------------------------------
cldap_server                      1012
dnssrv                            1012
dnsupdate                         1012
dreplsrv                          1012
ft_scanner                        1012
kccsrv                            1012
kdc_server                        1012
ldap_server                       1012
nbt_server                        1012
notify-daemon                     1016
rpc_server                        1012
samba                             1012
winbind_server                    1019
```

## Validez les journaux en ligne :

```bash
tail -f /var/log/samba-ad-dc/current
```

## Le résultat correct ressemblera à ceci :

```bash
2025-11-27_04:14:23.73604 Completed DNS update check OK
2025-11-27_04:14:25.35809 Registered pdc01<00> with 192.168.70.253 on interface 192.168.70.255
2025-11-27_04:14:25.35814 Registered pdc01<03> with 192.168.70.253 on interface 192.168.70.255
2025-11-27_04:14:25.35815 Registered pdc01<20> with 192.168.70.253 on interface 192.168.70.255
2025-11-27_04:14:25.35941 Registered EDUCATUX<1b> with 192.168.70.253 on interface 192.168.70.255
2025-11-27_04:14:25.35942 Registered EDUCATUX<1c> with 192.168.70.253 on interface 192.168.70.255
2025-11-27_04:14:25.35944 Registered EDUCATUX<00> with 192.168.70.253 on interface 192.168.70.255
2025-11-27_04:14:36.71381 Calling samba_kcc script
2025-11-27_04:14:37.31554 samba_runcmd_io_handler: Child /opt/samba/sbin/samba_kcc exited 0
2025-11-27_04:14:37.31557 Completed samba_kcc OK
```

## 🕒 Serveur NTP/Chrony

## Le contrôleur de domaine devra être le serveur de temps du réseau local, car avec un écart de 5 minutes, Kerberos n'authentifiera plus le client.

## Installez le package Chrony Server

```bash
xbps-install -Syu chrony
```

## Modifiez le fichier serveur, remplacez les référentiels de synchronisation de l'heure et libérez les requêtes réseau internes

```bash
vim /etc/chrony.conf
```

### Signalez les serveurs de temps publics au Brésil

```bash
# Comentar a linha do Servidor externo
#pool pool.ntp.org iburst (AQUI)

# Apontar Servidores de tempo BR
server 0.br.pool.ntp.org iburst
server 1.br.pool.ntp.org iburst
server 2.br.pool.ntp.org iburst
server 3.br.pool.ntp.org iburst

# Permitir sincronização da rede interna
allow 192.168.70.0/24
```

## Ajouter le service chronyd au démarrage de RUNIT

```bash
ln -sf /etc/sv/chronyd/ /var/service/
```

## Redémarrez TimeServer :

```bash
sv restart chronyd
```

## Validez les Serveurs, ils sont cycliques et aléatoires lors des requêtes

```bash
chronyc sources -v
```

## 🔐 Créer un fichier Kerberos

```bash
vim /etc/krb5.conf
```

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

## 🧭 Déverrouillez et réinitialisez /etc/resolv.conf APRÈS le provisionnement, et pointez vers le PDC lui-même

```bash
chattr -i /etc/resolv.conf
```

```bash
vim /etc/resolv.conf
```

## Contenu:

```bash
domain educatux.edu
search educatux.edu
nameserver 127.0.0.1
```

## Verrouillez à nouveau le fichier :

```bash
chattr +i /etc/resolv.conf
```

## 🔗 Lier les bibliothèques Winbind sur le système

## Validez les chemins libdir :

```bash
/opt/samba/sbin/smbd -b | grep LIBDIR
```

## Résultat attendu :

```bash
LIBDIR: /opt/samba/lib
```

## Créer des liens entre les bibliothèques. Préférez taper manuellement plutôt que de copier et coller ici.

```bash
ln -sf /opt/samba/lib/libnss_winbind.so.2 /usr/lib/
```

```bash
ln -sf /usr/lib/libnss_winbind.so.2 /usr/lib/libnss_winbind.so
```

## Relisez la configuration avec les nouvelles bibliothèques liées

```bash
ldconfig
```

## Validez l'efficacité de l'échange de tickets Kerberos, en ajoutant winbind aux deux lignes de nsswhitch (passwd et group) :

```bash
vim /etc/nsswitch.conf
```

```bash
passwd: files winbind
group:  files winbind
```

### Le reste du fichier reste tel quel

## 📝 Validez le smb.conf créé automatiquement par provisioning

```bash
cat /opt/samba/etc/smb.conf
```

```
# Global parameters
[global]
        ad dc functional level = 2016
        dns forwarder = 192.168.70.254
        netbios name = pdc01
        realm = EDUCATUX.EDU
        server role = active directory domain controller
        workgroup = EDUCATUX
        idmap_ldb:use rfc2307 = yes
        # aponte aos serviços, as interfaces ativas
        interfaces = eth0 
        bind interfaces only = yes

[sysvol]
        path = /opt/samba/var/locks/sysvol
        read only = No

[netlogon]
        path = /opt/samba/var/locks/sysvol/educatux.edu/scripts
        read only = No
```

## 🔍 Nous allons maintenant valider les services PDC importants tels que DNS, SMB, Winbind et Kerberos

```bash
ps aux | grep samba
```

## Résultat reçu :

```bash
root     28030  0.0  0.0   2392  1388 ?        Ss   01:14   0:00 runsv samba-ad-dc
root     28031  0.0  0.0   2540  1376 ?        S    01:14   0:00 svlogd -tt /var/log/samba-ad-dc
root     28032  0.1  3.3 129656 66884 ?        S    01:14   0:04 samba: root process  .
root     28033  0.0  1.6 129152 33728 ?        S    01:14   0:00 samba: tfork waiter process(28034)
root     28034  0.0  3.3 133112 67156 ?        Ss   01:14   0:00 /opt/samba/sbin/smbd -D --option=server role check:inhibit=yes --foreground
root     28038  0.0  1.6 129152 33432 ?        S    01:14   0:00 samba: tfork waiter process(28039)
root     28039  0.0  3.1 127588 63240 ?        Ss   01:14   0:00 /opt/samba/sbin/winbindd -D --option=server role check:inhibit=yes --foreground
root     28180  0.0  0.1   6696  2556 pts/0    S+   02:10   0:00 grep samba
```

```bash
samba-tool user show administrator
```

## Résultat reçu :

```bash
dn: CN=Administrator,CN=Users,DC=educatux,DC=edu
objectClass: top
objectClass: person
objectClass: organizationalPerson
objectClass: user
cn: Administrator
description: Built-in account for administering the computer/domain
instanceType: 4
whenCreated: 20251127040618.0Z
uSNCreated: 3889
name: Administrator
objectGUID: 732e3aed-f232-427d-9377-5bf7bc79cd8e
userAccountControl: 512
badPwdCount: 0
codePage: 0
countryCode: 0
badPasswordTime: 0
lastLogoff: 0
pwdLastSet: 134086899781242602
primaryGroupID: 513
objectSid: S-1-5-21-294413610-3908852046-3961109876-500
adminCount: 1
accountExpires: 9223372036854775807
sAMAccountName: Administrator
sAMAccountType: 805306368
objectCategory: CN=Person,CN=Schema,CN=Configuration,DC=educatux,DC=edu
isCriticalSystemObject: TRUE
memberOf: CN=Domain Admins,CN=Users,DC=educatux,DC=edu
memberOf: CN=Schema Admins,CN=Users,DC=educatux,DC=edu
memberOf: CN=Enterprise Admins,CN=Users,DC=educatux,DC=edu
memberOf: CN=Group Policy Creator Owners,CN=Users,DC=educatux,DC=edu
memberOf: CN=Administrators,CN=Builtin,DC=educatux,DC=edu
lastLogonTimestamp: 134086916533352620
whenChanged: 20251127043413.0Z
uSNChanged: 4307
lastLogon: 134086917409338150
logonCount: 5
distinguishedName: CN=Administrator,CN=Users,DC=educatux,DC=edu
```

```bash
wbinfo -u
```

## Résultat reçu :

```bash
EDUCATUX\administrator
EDUCATUX\guest
EDUCATUX\krbtgt
```

```bash
wbinfo -g
```

## Résultat reçu :

```bash
EDUCATUX\administrator
EDUCATUX\guest
EDUCATUX\krbtgt
[root@pdc01 samba-4.23.4]# wbinfo -g
EDUCATUX\cert publishers
EDUCATUX\ras and ias servers
EDUCATUX\allowed rodc password replication group
EDUCATUX\denied rodc password replication group
EDUCATUX\dnsadmins
EDUCATUX\enterprise read-only domain controllers
EDUCATUX\domain admins
EDUCATUX\domain users
EDUCATUX\domain guests
EDUCATUX\domain computers
EDUCATUX\domain controllers
EDUCATUX\schema admins
EDUCATUX\enterprise admins
EDUCATUX\group policy creator owners
EDUCATUX\read-only domain controllers
EDUCATUX\protected users
EDUCATUX\dnsupdateproxy
```

```bash
getent group "Domain Admins"
```

## Résultat reçu :

```bash
EDUCATUX\domain admins:x:3000004:
```

```bash
smbclient -L localhost -U Administrator
```

## Résultat reçu :

```bash
Password for [EDUCATUX\Administrator]:

        Sharename       Type      Comment
        ---------       ----      -------
        sysvol          Disk
        netlogon        Disk
        IPC$            IPC       IPC Service (Samba 4.23.4)
SMB1 disabled -- no workgroup available
```

```bash
samba-tool dns zonelist localhost -U administrator
```

## Résultat reçu :

```bash
Password for [EDUCATUX\administrator]:
  2 zone(s) found

  pszZoneName               : educatux.edu
  Flags                            : DNS_RPC_ZONE_DSINTEGRATED DNS_RPC_ZONE_UPDATE_SECURE
  ZoneType                      : DNS_ZONE_TYPE_PRIMARY
  Version                         : 50
  dwDpFlags                    : DNS_DP_AUTOCREATED DNS_DP_DOMAIN_DEFAULT DNS_DP_ENLISTED
  pszDpFqdn                    : DomainDnsZones.educatux.edu

  pszZoneName               : _msdcs.educatux.edu
  Flags                            : DNS_RPC_ZONE_DSINTEGRATED DNS_RPC_ZONE_UPDATE_SECURE
  ZoneType                      : DNS_ZONE_TYPE_PRIMARY
  Version                         : 50
  dwDpFlags                    : DNS_DP_AUTOCREATED DNS_DP_FOREST_DEFAULT DNS_DP_ENLISTED
  pszDpFqdn                    : ForestDnsZones.educatux.edu
```

```bash
samba-tool user show administrator
```

## Résultat reçu :

```bash
dn: CN=Administrator,CN=Users,DC=educatux,DC=edu
objectClass: top
objectClass: person
objectClass: organizationalPerson
objectClass: user
cn: Administrator
description: Built-in account for administering the computer/domain
instanceType: 4
whenCreated: 20251127040618.0Z
uSNCreated: 3889
name: Administrator
objectGUID: 732e3aed-f232-427d-9377-5bf7bc79cd8e
userAccountControl: 512
badPwdCount: 0
codePage: 0
countryCode: 0
badPasswordTime: 0
lastLogoff: 0
pwdLastSet: 134086899781242602
primaryGroupID: 513
objectSid: S-1-5-21-294413610-3908852046-3961109876-500
adminCount: 1
accountExpires: 9223372036854775807
sAMAccountName: Administrator
sAMAccountType: 805306368
objectCategory: CN=Person,CN=Schema,CN=Configuration,DC=educatux,DC=edu
isCriticalSystemObject: TRUE
memberOf: CN=Domain Admins,CN=Users,DC=educatux,DC=edu
memberOf: CN=Schema Admins,CN=Users,DC=educatux,DC=edu
memberOf: CN=Enterprise Admins,CN=Users,DC=educatux,DC=edu
memberOf: CN=Group Policy Creator Owners,CN=Users,DC=educatux,DC=edu
memberOf: CN=Administrators,CN=Builtin,DC=educatux,DC=edu
lastLogonTimestamp: 134086916533352620
whenChanged: 20251127043413.0Z
uSNChanged: 4307
lastLogon: 134086917409338150
logonCount: 5
distinguishedName: CN=Administrator,CN=Users,DC=educatux,DC=edu
```

## 🔐 Désactivez la complexité des mots de passe pour les utilisateurs du domaine (facilitez les tests en laboratoire - dangereux pour la production !)

```bash
samba-tool domain passwordsettings set --complexity=off
samba-tool domain passwordsettings set --history-length=0
samba-tool domain passwordsettings set --min-pwd-length=0
samba-tool domain passwordsettings set --min-pwd-age=0
samba-tool user setexpiry Administrator --noexpiry
```

## Relisez les paramètres de Samba4

```bash
smbcontrol all reload-config
```

## 🧪 Valider l'échange de billets Kerberos

```bash
kinit Administrator@EDUCATUX.EDU
```

```bash
klist
```

## Résultat reçu :

```bash
Ticket cache: FILE:/tmp/krb5cc_0
Default principal: Administrator@EDUCATUX.EDU

Valid starting       Expires              Service principal
27/11/2025 02:22:52  27/11/2025 12:22:52  krbtgt/EDUCATUX.EDU@EDUCATUX.EDU
        renew until 28/11/2025 02:22:47
```

```bash
samba-tool dns query pdc01.educatux.edu educatux.edu @ A -U Administrator
```

## Résultat reçu :

```bash
Password for [EDUCATUX\Administrator]:

  Name=, Records=1, Children=0
    A: 192.168.70.253 (flags=600000f0, serial=1, ttl=900)
  Name=_msdcs, Records=0, Children=0
  Name=_sites, Records=0, Children=1
  Name=_tcp, Records=0, Children=4
  Name=_udp, Records=0, Children=2
  Name=DomainDnsZones, Records=0, Children=2
  Name=ForestDnsZones, Records=0, Children=2
  Name=pdc01, Records=1, Children=0
    A: 192.168.70.253 (flags=f0, serial=1, ttl=900)
```

```bash
drill google.com @192.168.70.253
```

## Résultat obtenu :

```bash
;; ->>HEADER<<- opcode: QUERY, rcode: NOERROR, id: 50285
;; flags: qr rd ra ; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0
;; QUESTION SECTION:
;; google.com.  IN      A

;; ANSWER SECTION:
google.com.     300     IN      A       172.217.30.142

;; AUTHORITY SECTION:

;; ADDITIONAL SECTION:

;; Query time: 224 msec
;; EDNS: version 0; flags: ; udp: 1232
;; SERVER: 192.168.70.253
;; WHEN: Thu Nov 27 02:30:42 2025
;; MSG SIZE  rcvd: 55
```

```bash
samba_dnsupdate --verbose
```

```bash
IPs: ['192.168.70.253']
Looking for DNS entry A pdc01.educatux.edu 192.168.70.253 as pdc01.educatux.edu.
Looking for DNS entry CNAME a9126dd4-c5ad-46b4-b91b-6ae91313e3b8._msdcs.educatux.edu pdc01.educatux.edu as a9126dd4-c5ad-46b4-b91b-6ae91313e3b8._msdcs.educatux.edu.
Looking for DNS entry NS educatux.edu pdc01.educatux.edu as educatux.edu.
Looking for DNS entry NS _msdcs.educatux.edu pdc01.educatux.edu as _msdcs.educatux.edu.
Looking for DNS entry A educatux.edu 192.168.70.253 as educatux.edu.
Looking for DNS entry SRV _ldap._tcp.educatux.edu pdc01.educatux.edu 389 as _ldap._tcp.educatux.edu.
Checking 0 100 389 pdc01.educatux.edu. against SRV _ldap._tcp.educatux.edu pdc01.educatux.edu 389
Looking for DNS entry SRV _ldap._tcp.dc._msdcs.educatux.edu pdc01.educatux.edu 389 as _ldap._tcp.dc._msdcs.educatux.edu.
Checking 0 100 389 pdc01.educatux.edu. against SRV _ldap._tcp.dc._msdcs.educatux.edu pdc01.educatux.edu 389
Looking for DNS entry SRV _ldap._tcp.f5cccdab-a9d9-4b1f-9344-d2affb3c9855.domains._msdcs.educatux.edu pdc01.educatux.edu 389 as _ldap._tcp.f5cccdab-a9d9-4b1f-9344-d2affb3c9855.domains._msdcs.educatux.edu.
Checking 0 100 389 pdc01.educatux.edu. against SRV _ldap._tcp.f5cccdab-a9d9-4b1f-9344-d2affb3c9855.domains._msdcs.educatux.edu pdc01.educatux.edu 389
Looking for DNS entry SRV _kerberos._tcp.educatux.edu pdc01.educatux.edu 88 as _kerberos._tcp.educatux.edu.
Checking 0 100 88 pdc01.educatux.edu. against SRV _kerberos._tcp.educatux.edu pdc01.educatux.edu 88
Looking for DNS entry SRV _kerberos._udp.educatux.edu pdc01.educatux.edu 88 as _kerberos._udp.educatux.edu.
Checking 0 100 88 pdc01.educatux.edu. against SRV _kerberos._udp.educatux.edu pdc01.educatux.edu 88
Looking for DNS entry SRV _kerberos._tcp.dc._msdcs.educatux.edu pdc01.educatux.edu 88 as _kerberos._tcp.dc._msdcs.educatux.edu.
Checking 0 100 88 pdc01.educatux.edu. against SRV _kerberos._tcp.dc._msdcs.educatux.edu pdc01.educatux.edu 88
Looking for DNS entry SRV _kpasswd._tcp.educatux.edu pdc01.educatux.edu 464 as _kpasswd._tcp.educatux.edu.
Checking 0 100 464 pdc01.educatux.edu. against SRV _kpasswd._tcp.educatux.edu pdc01.educatux.edu 464
Looking for DNS entry SRV _kpasswd._udp.educatux.edu pdc01.educatux.edu 464 as _kpasswd._udp.educatux.edu.
Checking 0 100 464 pdc01.educatux.edu. against SRV _kpasswd._udp.educatux.edu pdc01.educatux.edu 464
Looking for DNS entry SRV _ldap._tcp.Default-First-Site-Name._sites.educatux.edu pdc01.educatux.edu 389 as _ldap._tcp.Default-First-Site-Name._sites.educatux.edu.
Checking 0 100 389 pdc01.educatux.edu. against SRV _ldap._tcp.Default-First-Site-Name._sites.educatux.edu pdc01.educatux.edu 389
Looking for DNS entry SRV _ldap._tcp.Default-First-Site-Name._sites.dc._msdcs.educatux.edu pdc01.educatux.edu 389 as _ldap._tcp.Default-First-Site-Name._sites.dc._msdcs.educatux.edu.
Checking 0 100 389 pdc01.educatux.edu. against SRV _ldap._tcp.Default-First-Site-Name._sites.dc._msdcs.educatux.edu pdc01.educatux.edu 389
Looking for DNS entry SRV _kerberos._tcp.Default-First-Site-Name._sites.educatux.edu pdc01.educatux.edu 88 as _kerberos._tcp.Default-First-Site-Name._sites.educatux.edu.
Checking 0 100 88 pdc01.educatux.edu. against SRV _kerberos._tcp.Default-First-Site-Name._sites.educatux.edu pdc01.educatux.edu 88
Looking for DNS entry SRV _kerberos._tcp.Default-First-Site-Name._sites.dc._msdcs.educatux.edu pdc01.educatux.edu 88 as _kerberos._tcp.Default-First-Site-Name._sites.dc._msdcs.educatux.edu.
Checking 0 100 88 pdc01.educatux.edu. against SRV _kerberos._tcp.Default-First-Site-Name._sites.dc._msdcs.educatux.edu pdc01.educatux.edu 88
Looking for DNS entry SRV _ldap._tcp.pdc._msdcs.educatux.edu pdc01.educatux.edu 389 as _ldap._tcp.pdc._msdcs.educatux.edu.
Checking 0 100 389 pdc01.educatux.edu. against SRV _ldap._tcp.pdc._msdcs.educatux.edu pdc01.educatux.edu 389
Looking for DNS entry A gc._msdcs.educatux.edu 192.168.70.253 as gc._msdcs.educatux.edu.
Looking for DNS entry SRV _gc._tcp.educatux.edu pdc01.educatux.edu 3268 as _gc._tcp.educatux.edu.
Checking 0 100 3268 pdc01.educatux.edu. against SRV _gc._tcp.educatux.edu pdc01.educatux.edu 3268
Looking for DNS entry SRV _ldap._tcp.gc._msdcs.educatux.edu pdc01.educatux.edu 3268 as _ldap._tcp.gc._msdcs.educatux.edu.
Checking 0 100 3268 pdc01.educatux.edu. against SRV _ldap._tcp.gc._msdcs.educatux.edu pdc01.educatux.edu 3268
Looking for DNS entry SRV _gc._tcp.Default-First-Site-Name._sites.educatux.edu pdc01.educatux.edu 3268 as _gc._tcp.Default-First-Site-Name._sites.educatux.edu.
Checking 0 100 3268 pdc01.educatux.edu. against SRV _gc._tcp.Default-First-Site-Name._sites.educatux.edu pdc01.educatux.edu 3268
Looking for DNS entry SRV _ldap._tcp.Default-First-Site-Name._sites.gc._msdcs.educatux.edu pdc01.educatux.edu 3268 as _ldap._tcp.Default-First-Site-Name._sites.gc._msdcs.educatux.edu.
Checking 0 100 3268 pdc01.educatux.edu. against SRV _ldap._tcp.Default-First-Site-Name._sites.gc._msdcs.educatux.edu pdc01.educatux.edu 3268
Looking for DNS entry A DomainDnsZones.educatux.edu 192.168.70.253 as DomainDnsZones.educatux.edu.
Looking for DNS entry SRV _ldap._tcp.DomainDnsZones.educatux.edu pdc01.educatux.edu 389 as _ldap._tcp.DomainDnsZones.educatux.edu.
Checking 0 100 389 pdc01.educatux.edu. against SRV _ldap._tcp.DomainDnsZones.educatux.edu pdc01.educatux.edu 389
Looking for DNS entry SRV _ldap._tcp.Default-First-Site-Name._sites.DomainDnsZones.educatux.edu pdc01.educatux.edu 389 as _ldap._tcp.Default-First-Site-Name._sites.DomainDnsZones.educatux.edu.
Checking 0 100 389 pdc01.educatux.edu. against SRV _ldap._tcp.Default-First-Site-Name._sites.DomainDnsZones.educatux.edu pdc01.educatux.edu 389
Looking for DNS entry A ForestDnsZones.educatux.edu 192.168.70.253 as ForestDnsZones.educatux.edu.
Looking for DNS entry SRV _ldap._tcp.ForestDnsZones.educatux.edu pdc01.educatux.edu 389 as _ldap._tcp.ForestDnsZones.educatux.edu.
Checking 0 100 389 pdc01.educatux.edu. against SRV _ldap._tcp.ForestDnsZones.educatux.edu pdc01.educatux.edu 389
Looking for DNS entry SRV _ldap._tcp.Default-First-Site-Name._sites.ForestDnsZones.educatux.edu pdc01.educatux.edu 389 as _ldap._tcp.Default-First-Site-Name._sites.ForestDnsZones.educatux.edu.
Checking 0 100 389 pdc01.educatux.edu. against SRV _ldap._tcp.Default-First-Site-Name._sites.ForestDnsZones.educatux.edu pdc01.educatux.edu 389
No DNS updates needed
```

### ✅ RÉSUMÉ FINAL

## 🎉 Félicitations, vous venez de configurer un domaine AD 2016 entièrement fonctionnel sur Void Linux !

### 👉 RAPPELEZ-VOUS : Samba4, bien qu'il puisse être géré par ligne de commande, a été conçu pour être géré par des outils de gestion de serveur à distance - RSAT, qui peuvent être installés sur une machine Windows 10, sans aucun problème !

## Vous pouvez désormais :

- joindre des machines Windows au domaine
- utiliser des GPO
- tester la réplication (lors de la création d'un DC2)
- créer des utilisateurs/groupes via RSAT
- configurer la réplication sysvol (avec Rsync ou le nouveau samba-gpupdate)
- ajouter des redirecteurs DNS
- activer DFS
- créer un serveur de fichiers membre
- etc.

---

🎯 C'EST TOUS LES GENS !

👉Contact : zerolies@disroot.org
👉 https://t.me/z3r0l135
