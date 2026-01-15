
# Dateiserver mit Samba4 auf Void Linux Server ;D

## 🎯 Ziel – Stellen Sie einen Dateiserver unter Void Linux (glibc) bereit, indem Sie Samba4 aus der Quelle, AD-Integration, ACLs, Diensten und dem gesamten Stack kompilieren, der für einen Dateiserver zur Bedienung von Netzwerk-Clients erforderlich ist.

## 🔧 Netzwerklabor mit QEMU/Virtmanager. Passen Sie das Tutorial an Ihre eigene Umgebung an.

---

## 📡 Lokales Netzwerklayout

- Domain: EDUCATUX.EDU

- Hostname: voidfiles

- Firewall 192.168.70.254 (DNS/GW)

- IP: 192.168.70.251

## Installieren Sie Void Linux

## Ändern Sie die Standard-Shell auf Void

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

## 🖥️ Hostnamen festlegen

```bash
echo "voidfiles" > /etc/hostname
```

## 🏠 /etc/hosts

```bash
vim /etc/hosts
```

## Inhalt:

```bash
127.0.0.1      localhost
127.0.1.1      voidfiles.educatux.edu voidfiles
192.168.70.251 voidfiles.educatux.edu voidfiles
```

## 🌐 Statische IP konfigurieren

## 👉 Wir werden die Standard-Void-Methode /etc/dhcpcd.conf verwenden

```bash
vim /etc/dhcpcd.conf
```

## IP, Gateway (Router) und DNS (AD) hinzufügen:

```bash
interface eth0
static ip_address=192.168.70.251/24
static routers=192.168.70.254
static domain_name_servers=192.168.70.250
```

## Netzwerkschnittstelle neu starten:

```bash
sv restart dhcpcd
```

## 🧭 DNS-Adresse festlegen – Zeigen Sie auf den PDC

```bash
vim /etc/resolv.conf
```

## Inhalt:

```bash
domain educatux.edu
search educatux.edu
nameserver 192.168.70.250
```

## Resolv.conf sperren

```bash
chattr +i /etc/resolv.conf
```

## 🔍 Überprüfen Sie die Netzwerkschnittstellenadresse

```bash
ip -c addr
ip -br link
```

## 📥 Laden Sie den Samba4-Quellcode herunter und extrahieren Sie ihn

```bash
wget https://download.samba.org/pub/samba/samba-4.23.4.tar.gz
```

```bash
tar -xvzf samba-4.23.4.tar.gz
```

## Von der Quelle kompilieren und installieren

```bash
cd samba-4.23.4
```

```bash
./configure --prefix=/opt/samba
```

```bash
make -j$(nproc) && make install
```

## Hinweise:

- Void stört nicht, da Samba in /opt/samba kompiliert wird.

- make -j beschleunigt die Kompilierung erheblich – gehen Sie trotzdem einen Kaffee trinken.

- Nach der Installation verfügt das kompilierte Samba4 über keine Runit-Dienste.

- Wir werden die Dienste manuell erstellen.

## 📁 Fügen Sie Samba4 zum Systempfad hinzu und laden Sie die Umgebung neu

```bash
echo 'export PATH=/opt/samba/bin:/opt/samba/sbin:$PATH' >> /etc/profile
```

```bash
source /etc/profile
```

## Testen Sie das Einfügen von Samba4 PATH in das Betriebssystem

```bash
samba-tool -V
```

## Ausgabe:

```bash
4.23.4
```

## ⚠️ Warnung: Verwenden Sie NICHT den Bereitstellungsbefehl auf dem Dateiserver!

## 📝 Erstellen Sie die Datei smb.conf

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

## Erstellen Sie die Protokolldatei

```bash
mkdir /opt/samba/var
```

## 📂 Erstellen Sie den Freigabepfad

```bash
sudo mkdir -p /srv/samba/public
sudo chown -R root:root /srv/samba/public
sudo chmod -R 0770 /srv/samba/public
```

## Laden Sie die Samba4-Konfiguration neu

```bash
smbcontrol all reload-config
```

## 🕒 NTP / Chrony Server

## Der Domänencontroller muss der lokale Zeitserver sein, da Kerberos bei einer Abweichung von 5 Minuten keine Clients mehr authentifiziert.

## Installieren Sie Chrony

```bash
xbps-install -Syu chrony
```

## Bearbeiten Sie die Konfiguration und erlauben Sie das interne Netzwerk

```bash
vim /etc/chrony.conf
```

## Stellen Sie die Domänenkontrolle auf den Zeitservern ein:

```bash
# Comment the external line
#pool pool.ntp.org iburst

# PDC Time Servers
server 192.168.70.250 iburst
```

## Aktivieren Sie Chronyd in Runit

```bash
ln -sf /etc/sv/chronyd/ /var/service/
```

## Dienst neu starten:

```bash
sv restart chronyd
```

## Server validieren:

```bash
chronyc sources -v
```

## 🔐 Erstellen Sie die Kerberos-Datei – Zeigen Sie auf den PDC

```bash
vim /etc/krb5.conf
```

## enthaltend

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
        kdc = 192.168.70.250
        admin_server = 192.168.70.250
        default_domain = educatux.edu
    }

[domain_realm]
    .educatux.edu = EDUCATUX.EDU
    educatux.edu = EDUCATUX.EDU
```

## Kerberos-Test

```bash
kinit Administrator
```

```bash
klist
```

## Ergebnis erhalten

```bash
Ticket cache: FILE:/tmp/krb5cc_0
Default principal: administrator@EDUCATUX.EDU

Valid starting       Expires              Service principal
11/12/2025 09:43:40  11/12/2025 19:43:40  krbtgt/EDUCATUX.EDU@EDUCATUX.EDU
	renew until 12/12/2025 09:43:36
```

## 🔗 Winbind-Bibliotheken mit dem System verknüpfen

## Libdir-Pfad validieren:

```bash
/opt/samba/sbin/smbd -b | grep LIBDIR
```

## Erwartet:

```bash
LIBDIR: /opt/samba/lib
```

## Erstellen Sie Bibliothekslinks (bevorzugen Sie die manuelle Eingabe):

```bash
ln -sf /opt/samba/lib/libnss_winbind.so.2 /usr/lib/
```

```bash
ln -sf /usr/lib/libnss_winbind.so.2 /usr/lib/libnss_winbind.so
```

## Bibliothekscache neu laden:

```bash
ldconfig
```

## Aktualisieren Sie nsswitch für den Kerberos-Ticketaustausch (fügen Sie winbind hinzu):

```bash
vim /etc/nsswitch.conf
```

```bash
passwd: files winbind
group:  files winbind
```

## Lassen Sie den Rest unberührt

## Treten Sie der Domäne bei

```bash
net ads join -U Administrator
```

## Ergebnis erhalten

```bash
Password for [EDUCATUX\Administrator]:
Using short domain name -- EDUCATUX
Joined 'VOIDFILES' to dns domain 'educatux.edu'
```

## 📦 Erstellen Sie die RUNIT-Dienste für smbd, winbindd und optional nmbd

### Void Linux verwendet Runit als Init-System und sein nativer Logger ist svlogd, der im Runit-Basispaket enthalten ist. Es sind keine zusätzlichen Pakete erforderlich.

## SMBD – Service und Protokollierung

## Erstellen Sie Dienst- und Protokollverzeichnisse

```bash
mkdir -p /etc/sv/smbd/log
mkdir -p /var/log/smbd
```

## Erstellen Sie /etc/sv/smbd/run

```bash
vim /etc/sv/smbd/run
```

## Inhalt

```bash
#!/bin/sh
exec 2>&1
exec /opt/samba/sbin/smbd --foreground --no-process-group
```

## Erlaubnis

```bash
chmod +x /etc/sv/smbd/run
```

## Erstellen Sie /etc/sv/smbd/log/run

```bash
vim /etc/sv/smbd/log/run
```

## Inhalt

```bash
#!/bin/sh
exec svlogd -tt /var/log/smbd
```

## Erlaubnis

```bash
chmod +x /etc/sv/smbd/log/run
```

## Debuggen (optional)

```bash
/opt/samba/sbin/smbd -i
```

## Ergebnis erhalten

```bash
smbd version 4.23.4 started.
Copyright Andrew Tridgell and the Samba Team 1992-2025
daemon 'smbd' : Starting process ...
```

## WINBINDD – Dienst und Protokollierung

## Erstellen Sie Dienst- und Protokollverzeichnisse

```bash
mkdir -p /etc/sv/winbindd/log
mkdir -p /var/log/winbindd
```

## Erstellen Sie /etc/sv/winbindd/run

```bash
vim /etc/sv/winbindd/run
```

## Inhalt

```bash
#!/bin/sh
exec 2>&1
exec /opt/samba/sbin/winbindd --foreground --no-process-group
```

## Erlaubnis

```bash
chmod +x /etc/sv/winbindd/run
```

## Erstellen Sie /etc/sv/winbindd/log/run

```bash
vim /etc/sv/winbindd/log/run
```

## Inhalt

```bash
#!/bin/sh
exec 2>&1
exec /opt/samba/sbin/winbindd --foreground --no-process-group
```

## Erlaubnis

```bash
chmod +x /etc/sv/winbindd/log/run
```

## Debuggen (optional)

```bash
/opt/samba/sbin/winbindd -i
```

## Ergebnis erhalten

```bash
winbindd version 4.23.4 started.
Copyright Andrew Tridgell and the Samba Team 1992-2025
daemon 'winbindd' : Starting process ...
```

## NMBD – Service und Protokollierung (optional)

### Aktivieren Sie diese Option nur, wenn Ihre Umgebung NetBIOS/SMB1-Browsing erfordert.

## Erstellen Sie Dienst- und Protokollverzeichnisse

```bash
mkdir -p /etc/sv/nmbd/log
mkdir -p /var/log/nmbd
```

## Erstellen Sie /etc/sv/nmbd/run

```bash
vim /etc/sv/nmbd/run
```

## Inhalt

```bash
#!/bin/sh
exec 2>&1
exec /opt/samba/sbin/nmbd --foreground --no-process-group
```

## Erlaubnis

```bash
chmod +x /etc/sv/nmbd/run
```

## Erstellen Sie /etc/sv/nmbd/log/run

```bash
vim /etc/sv/nmbd/log/run
```
 
## Inhalt

```bash
#!/bin/sh
exec svlogd -tt /var/log/nmbd
```

## Erlaubnis

```bash
chmod +x /etc/sv/nmbd/log/run
```

## Dienste aktivieren

```bash
ln -sf /etc/sv/smbd /var/service/
ln -sf /etc/sv/winbindd /var/service/
```

## Optional – nur aktivieren, wenn NetBIOS verwendet wird:

```bash
ln -sf /etc/sv/nmbd /var/service/
```

## Dienste validieren

```bash
sv status smbd winbindd nmbd
```

## 🧪 Integration validieren

```bash
net ads testjoin
```

## Ergebnis erhalten

```bash
Join is OK
```

```bash
wbinfo -u
```

## Ergebnis erhalten

```bash
guest
krbtgt
administrator```

```bash
wbinfo -g
```

## Result obtained

```bash
schreibgeschützte Domänencontroller für Unternehmen
geschützte Benutzer
Domänencontroller
Domain-Gäste
schreibgeschützte Domänencontroller
Schemaadministratoren
dnsupdateproxy
Domain-Administratoren
Besitzer von Gruppenrichtlinienerstellern
ras- und ias-server
DNS-Administratoren
erlaubte Rodc-Passwort-Replikationsgruppe
Unternehmensadministratoren
Zertifikatsverlage
Domänenbenutzer
Rodc-Passwort-Replikationsgruppe verweigert
Domänencomputer
```

```bash
wbinfo --ping-dc
```

## Result obtained

```bash
Die Überprüfung des NETLOGON für die DC-Verbindung der Domäne [EDUCATUX] zu „voiddc.educatux.edu“ war erfolgreich
```

## ✅ FINAL SUMMARY

## 🎉 Congratulations — you have successfully deployed a fully functional File Server on Void Linux!

## 👉 REMEMBER: While Samba4 can be managed via CLI, it was designed to be managed via RSAT Remote Server Administration Tools, which can be installed on a Windows 10 machine without issues!

---

🎯 THAT'S ALL FOLKS!

👉 Contact: zerolies@disroot.org

👉 https://t.me/z3r0l135
