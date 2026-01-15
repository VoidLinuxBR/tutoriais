# Primärer Domänencontroller (Active Directory), auf dem Samba4 unter Void Linux Server ausgeführt wird ;D

## 🎯 Ziel – Hochladen eines primären Domänencontrollers auf Void Linux (glibc), Kompilieren von Samba4 aus dem Quellcode, Konfigurieren von internem DNS, Kerberos, AD-Integration, ACLs, Diensten und dem gesamten Stack, der zur Steuerung von Netzwerk-Clients erforderlich ist.

### 🔧 PASSEN Sie das Tutorial natürlich an IHRE Realität an!

## 📡 Lokales Netzwerklayout

- Domain: EDUCATUX.EDU
- Hostname: pdc01
- Firewall 192.168.70.254 (DNS/GW)
- IP: 192.168.70.253

---

## Die Standardinstallation von Void Linux wird in diesem Tutorial nicht behandelt.

## Ändern Sie nach der Installation die Standard-Shell von Void

```bash
chsh -s /bin/bash
```

## 🧩 Installieren Sie Abhängigkeitspakete, um Samba4 auf Void zu kompilieren

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

## ⚠️ ACHTUNG: Das kompilierte Samba4 enthält den standardmäßig integrierten Heimdal-Kerberos-Code (internes KDC), jedoch keine Kerberos-Clients. In diesem Fall stellt das Repository Binärpakete von MIT bereit, die ohne Probleme oder Beeinträchtigung des auf dem Domänencontroller kompilierten Standard-Kerberos-Heimdals installiert werden können. Die Pakete sind: mit-krb5 mit-krb5-client mit-krb5-devel. Sie DÜRFEN JEDOCH unter keinen Umständen das krb5-server-Binärpaket aus dem Repository installieren, da dies zu einem konkurrierenden Dienst mit den Heimdal-Kerberos führen würde, die in Samba4 integriert sind!

## Die von MIT-krb5-Kunden bereitgestellten Dienste finden Sie unter:

```bash
/usr/bin/kinit
/usr/bin/klist
/usr/bin/kvno
/usr/bin/kdestroy
```

## 🖥️ Setar-Hostnamen

```bash
echo "pdc01" > /etc/hostname
```

## 🏠 /etc/hosts

```bash
vim /etc/hosts
```

## Inhalt:

```bash
127.0.0.1      localhost
127.0.1.1      pdc01.educatux.edu pdc01
192.168.70.253 pdc01.educatux.edu pdc01
```

## 🌐 Konfigurieren Sie eine feste IP

### 👉 Wir verwenden die Standardmethode von Void, /etc/dhcpcd.conf

```bash
vim /etc/dhcpcd.conf
```

## IP, Gateway und DNS hinzufügen:## 🎯 Zweck

```bash
interface eth0
static ip_address=192.168.70.253/24
static routers=192.168.70.254
static domain_name_servers=192.168.70.254
```

## Starten Sie die Netzwerkschnittstelle neu:

```bash
sv restart dhcpcd
```

## 🌐 Legen Sie VOR der Bereitstellung temporäres DNS (Router) fest

```bash
echo "nameserver 192.168.70.254" > /etc/resolv.conf
```

## Sperren Sie die resolv.conf-Konfiguration

```bash
chattr +i /etc/resolv.conf
```

## 🔍 Überprüfen Sie die der Netzwerkschnittstelle zugewiesene Adresse

```bash
ip -c addr
```

```bash
ip -br link
```

## 📥 Laden Sie den Samba4-Quellcode herunter und entpacken Sie ihn

```bash
wget https://download.samba.org/pub/samba/samba-4.23.4.tar.gz
```

```bash
tar -xvzf samba-4.23.4.tar.gz
```

## Kompilieren und installieren Sie den Quellcode

```bash
cd samba-4.23.4
```

```bash
./configure --prefix=/opt/samba
```

```bash
make -j$(nproc) && make install
```

## Kommentar:

- Void stört die Installation nicht, da Samba in /opt/samba kompiliert wird.
- Make -j beschleunigt die Kompilierung auf jeden Fall erheblich, gehen Sie einen Kaffee trinken.
- Nach der Installation sind im Runit des kompilierten Samba4 keine Dienste erstellt.
- Wir werden die Dienste manuell erstellen.

## 📁 Fügen Sie Samba4 zum Systempfad hinzu und lesen Sie die Umgebung erneut

```bash
echo 'export PATH=/opt/samba/bin:/opt/samba/sbin:$PATH' >> /etc/profile
```

```bash
source /etc/profile
```

## Testen Sie das Einfügen des Samba4-PATHs im Betriebssystem

```bash
samba-tool -V
```

## Ergebnis:

```bash
4.23.4
```

🏰 Bereitstellung der SAMBA4-Domäne (Erstellen des PDC selbst)

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

### Samba4 erstellt:

```bash
/opt/samba/etc/smb.conf
/opt/samba/private/*
/opt/samba/var/locks/sysvol
```

## Zusammenfassend Samba4:

- Erstellt die AD-Gesamtstruktur, den primären Domänencontroller, das interne DNS und die Kontendatenbank.
- Definiert Domäne, Realm, Funktionsebene 2016 und Administratorkennwort.
- Void installiert kein natives Samba, daher gibt es keinen Konflikt.
- Danach wird der DNS zum PDC selbst und muss /etc/resolv.conf auf 127.0.0.1 anpassen.

## ⚙️ Validieren Sie die Funktionsebene von Active Directory 2016

```bash
samba-tool domain level show
```

## Ergebnis:

```bash
Domain and forest function level for domain 'DC=educatux,DC=edu'
Forest function level: (Windows) 2016
Domain function level: (Windows) 2016
Lowest function level of a DC: (Windows) 2016
```

## 🧪 Testen Sie den AD DC-Dienst manuell, bevor Sie den Dienst erstellen

```bash
/opt/samba/sbin/samba -i -M single
```

* -i → Vordergrund
* -M Single → Einzelprozessmodell (löst kein Daemon-Forking außerhalb der Runit-Kontrolle aus)

## Wenn alles in Ordnung ist, sehen Sie:

```bash
Completed DNS update check OK
Completed SPN update check OK
Registered EDUCATUX<1c> ...
```

## STRG+C zum Beenden

## 📦 Erstellen Sie den RUNIT-Dienst samba-ad-dc, um AD beim Booten hochzuladen

## ⚠️ Dieser Teil ist sehr wichtig. Löschen Sie alte Reste, wenn Sie einen bereits vorhandenen Server neu anpassen!!

```bash
sv stop samba-ad-dc 2>/dev/null
rm -rf /var/service/samba-ad-dc
rm -rf /var/service/a-ad-dc
rm -rf /etc/sv/samba-ad-dc
rm -rf /etc/sv/a-ad-dc
rm -rf /etc/sv/*/supervise
rm -rf /var/service/*/supervise
```

## Jetzt erstellen wir die Samba-AD-DC-Dienste und -Berechtigungen mit Protokollen, damit Runit beim Systemstart geladen werden kann:

## Erstellen Sie zunächst die Servicestruktur

```bash
mkdir -p /etc/sv/samba-ad-dc/log
mkdir -p /var/log/samba-ad-dc
```

## Erstellen Sie den Ausführungsdienst

```bash
cat > /etc/sv/samba-ad-dc/run << 'EOF'
#!/bin/sh
exec 2>&1
exec /opt/samba/sbin/samba -i -M single --debuglevel=3
EOF
```

## Legen Sie die Berechtigung zum Ausführen des Dienstes fest

```bash
chmod +x /etc/sv/samba-ad-dc/run
```

## Erstellen Sie die Protokolldatei

```bash
cat > /etc/sv/samba-ad-dc/log/run << 'EOF'
#!/bin/sh
exec svlogd -tt /var/log/samba-ad-dc
EOF
```

## Legen Sie die Protokollierungs-/Ausführungsberechtigung fest

```bash
chmod +x /etc/sv/samba-ad-dc/log/run
```

## Aktivieren Sie den Dienst „samba-ad-dc“ so, dass er beim Booten ausgeführt wird:

```bash
ln -sf /etc/sv/samba-ad-dc/ /var/service/
```

## Überprüfen Sie, ob es ausgeführt wird

```bash
sv status samba-ad-dc
```

## Sie sollten etwa Folgendes sehen:

```bash
run: samba-ad-dc: (pid 28032) 4s; run: log: (pid 28031) 4s
```

```bash
samba-tool processes
```

## Ergebnis erhalten:

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

## Protokolle online validieren:

```bash
tail -f /var/log/samba-ad-dc/current
```

## Die korrekte Ausgabe wird etwa so aussehen:

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

## 🕒 NTP / Chrony Server

## Der Domänencontroller muss der Zeitserver des lokalen Netzwerks sein, da Kerberos bei einer Abweichung von 5 Minuten den Client nicht mehr authentifiziert

## Installieren Sie das Chrony Server-Paket

```bash
xbps-install -Syu chrony
```

## Bearbeiten Sie die Serverdatei, ersetzen Sie Zeitsynchronisierungs-Repositorys und geben Sie interne Netzwerkabfragen frei

```bash
vim /etc/chrony.conf
```

### Weisen Sie auf öffentliche Zeitserver in Brasilien hin

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

## Fügen Sie den chronyd-Dienst zum RUNIT-Start hinzu

```bash
ln -sf /etc/sv/chronyd/ /var/service/
```

## TimeServer neu starten:

```bash
sv restart chronyd
```

## Validieren Sie die Server. Sie sind bei Abfragen zyklisch und zufällig

```bash
chronyc sources -v
```

## 🔐 Kerberos-Datei erstellen

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

## 🧭 Entsperren und setzen Sie /etc/resolv.conf NACH der Bereitstellung zurück und verweisen Sie auf den PDC selbst

```bash
chattr -i /etc/resolv.conf
```

```bash
vim /etc/resolv.conf
```

## Inhalt:

```bash
domain educatux.edu
search educatux.edu
nameserver 127.0.0.1
```

## Sperren Sie die Datei erneut:

```bash
chattr +i /etc/resolv.conf
```

## 🔗 Winbind-Bibliotheken auf dem System verknüpfen

## Libdir-Pfade validieren:

```bash
/opt/samba/sbin/smbd -b | grep LIBDIR
```

## Erwartete Ausgabe:

```bash
LIBDIR: /opt/samba/lib
```

## Erstellen Sie Verknüpfungen zwischen Bibliotheken. Geben Sie hier lieber manuell ein als zu kopieren und einzufügen.

```bash
ln -sf /opt/samba/lib/libnss_winbind.so.2 /usr/lib/
```

```bash
ln -sf /usr/lib/libnss_winbind.so.2 /usr/lib/libnss_winbind.so
```

## Lesen Sie die Konfiguration mit den neuen verknüpften Bibliotheken erneut

```bash
ldconfig
```

## Überprüfen Sie die Wirksamkeit des Kerberos-Ticketaustauschs, indem Sie winbind zu den beiden Zeilen von nsswhitch (passwd und group) hinzufügen:

```bash
vim /etc/nsswitch.conf
```

```bash
passwd: files winbind
group:  files winbind
```

### Der Rest der Datei bleibt unverändert

## 📝 Validieren Sie die durch die Bereitstellung automatisch erstellte smb.conf

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

## 🔍 Jetzt werden wir wichtige PDC-Dienste wie DNS, SMB, Winbind und Kerberos validieren

```bash
ps aux | grep samba
```

## Ergebnis erhalten:

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

## Ergebnis erhalten:

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

## Ergebnis erhalten:

```bash
EDUCATUX\administrator
EDUCATUX\guest
EDUCATUX\krbtgt
```

```bash
wbinfo -g
```

## Ergebnis erhalten:

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

## Ergebnis erhalten:

```bash
EDUCATUX\domain admins:x:3000004:
```

```bash
smbclient -L localhost -U Administrator
```

## Ergebnis erhalten:

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

## Ergebnis erhalten:

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

## Ergebnis erhalten:

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

## 🔐 Deaktivieren Sie die Kennwortkomplexität für Domänenbenutzer (erleichtern Sie Labortests – unsicher für die Produktion!)

```bash
samba-tool domain passwordsettings set --complexity=off
samba-tool domain passwordsettings set --history-length=0
samba-tool domain passwordsettings set --min-pwd-length=0
samba-tool domain passwordsettings set --min-pwd-age=0
samba-tool user setexpiry Administrator --noexpiry
```

## Lesen Sie die Samba4-Einstellungen erneut

```bash
smbcontrol all reload-config
```

## 🧪 Validieren Sie den Kerberos-Ticketaustausch

```bash
kinit Administrator@EDUCATUX.EDU
```

```bash
klist
```

## Ergebnis erhalten:

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

## Ergebnis erhalten:

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

## Erhaltenes Ergebnis:

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

### ✅ ABSCHLIESSENDE ZUSAMMENFASSUNG

## 🎉 Herzlichen Glückwunsch – Sie haben gerade eine voll funktionsfähige 2016 AD-Domäne auf Void Linux eingerichtet!

### 👉 Denken Sie daran: Obwohl Samba4 über die Befehlszeile verwaltet werden kann, wurde es für die Verwaltung durch Remote-Serververwaltungstools entwickelt – RSAT, das problemlos auf einem Windows 10-Computer installiert werden kann!

## Jetzt können Sie:

- Fügen Sie Windows-Computer der Domäne hinzu
- Verwenden Sie Gruppenrichtlinienobjekte
- Testreplikation (beim Erstellen eines DC2)
- Erstellen Sie Benutzer/Gruppen über RSAT
- Konfigurieren Sie die Sysvol-Replikation (mit Rsync oder dem neuen Samba-Gpupdate).
- Fügen Sie DNS-Weiterleitungen hinzu
- DFS aktivieren
- Erstellen Sie einen Mitgliedsdateiserver
- usw.

---

🎯 DAS IST ALLES, LEUTE!

👉 Kontakt: zerolies@disroot.org
👉 https://t.me/z3r0l135
