# File Server running Samba4 on Void Linux Server ;D

## 🎯 Goal – Deploy a File Server on Void Linux (glibc) by compiling Samba4 from source, AD integration, ACLs, services and the entire stack required for a File Server to serve network clients.

## 🔧 Networking lab with QEMU/Virtmanager. Adjust the tutorial to match your own environment.

---

## 📡 Local network layout

- Domain: EDUCATUX.EDU

- Hostname: voidfiles

- Firewall 192.168.70.254 (DNS/GW)

- IP: 192.168.70.251

## Install Void Linux

## Change the default shell on Void

```bash
chsh -s /bin/bash
```

## 🧩 Install dependency packages to compile Samba4 on Void

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
 bind ldns pkg-config wget
```

## 🖥️ Set hostname

```bash
echo "voidfiles" > /etc/hostname
```

## 🏠 /etc/hosts

```bash
vim /etc/hosts
```

## Content:

```bash
127.0.0.1      localhost
127.0.1.1      voidfiles.educatux.edu voidfiles
192.168.70.251 voidfiles.educatux.edu voidfiles
```

## 🌐 Configure static IP

## 👉 We will use the standard Void method, /etc/dhcpcd.conf

```bash
vim /etc/dhcpcd.conf
```

## Add IP, gateway (Router) and DNS (AD):

```bash
interface eth0
static ip_address=192.168.70.251/24
static routers=192.168.70.254
static domain_name_servers=192.168.70.250
```

## Restart network interface:

```bash
sv restart dhcpcd
```

## 🧭 Set DNS address - Point to the PDC

```bash
vim /etc/resolv.conf
```

## Content:

```bash
domain educatux.edu
search educatux.edu
nameserver 192.168.70.250
```

## Lock resolv.conf

```bash
chattr +i /etc/resolv.conf
```

## 🔍 Validate the network interface address

```bash
ip -c addr
ip -br link
```

## 📥 Download and extract Samba4 source code

```bash
wget https://download.samba.org/pub/samba/samba-4.23.3.tar.gz
```

```bash
tar -xvzf samba-4.23.3.tar.gz
```

## Compile and install from source

```bash
cd samba-4.23.3
```

```bash
./configure --prefix=/opt/samba
```

```bash
make -j$(nproc) && make install
```

## Notes:

- Void does not interfere because Samba is compiled into /opt/samba.

- make -j greatly speeds up compilation—still, go grab a coffee.

- After installation, compiled Samba4 does not have any runit services.

- We will create the services manually.

## 📁 Add Samba4 to system PATH and reload environment

```bash
echo 'export PATH=/opt/samba/bin:/opt/samba/sbin:$PATH' >> /etc/profile
```

```bash
source /etc/profile
```

## Test Samba4 PATH insertion into the OS

```bash
samba-tool -V
```

## Output:

```bash
4.23.3
```

## ⚠️ Warning: DO NOT use the provisioning command on the File Server!

## 📝 Create the smb.conf file

```bash
vim /opt/samba/etc/smb.conf
```

```bash
[global]
   workgroup = EDUCATUX
   security = ads
   realm = EDUCATUX.EDU
   encrypt passwords = yes
   interfaces = lo eth0
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

## Create the log file

```bash
mkdir /opt/samba/var
```

## 📂 Create the sharing path

```bash
sudo mkdir -p /srv/samba/public
sudo chown -R root:root /srv/samba/public
sudo chmod -R 0770 /srv/samba/public
```

## Reload Samba4 config

```bash
smbcontrol all reload-config
```

## 🕒 NTP / Chrony Server

## The Domain Controller must be the local Time Server, because with a 5-minute drift Kerberos will no longer authenticate clients.

## Install Chrony

```bash
xbps-install -Syu chrony
```

## Edit config and allow internal network

```bash
vim /etc/chrony.conf
```

## Set Domain Control at the time servers:

```bash
# Comment the external line
#pool pool.ntp.org iburst

# PDC Time Servers
server 192.168.70.250 iburst
```

## Enable chronyd in runit

```bash
ln -s /etc/sv/chronyd/ /var/service
```

## Restart service:

```bash
sv restart chronyd
```

## Validate servers:

```bash
chronyc sources -v
```

## 🔐  Create the Kerberos file - Point to the PDC

```bash
vim /etc/krb5.conf
```

## containing

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

## Kerberos test

```bash
kinit Administrator
```

```bash
klist
```

## Result obtained

```bash
Ticket cache: FILE:/tmp/krb5cc_0
Default principal: administrator@EDUCATUX.EDU

Valid starting       Expires              Service principal
11/12/2025 09:43:40  11/12/2025 19:43:40  krbtgt/EDUCATUX.EDU@EDUCATUX.EDU
	renew until 12/12/2025 09:43:36
```

## 👑 Give Administrator root privileges

```bash
vim /opt/samba/etc/user.map
```

```bash
!root=educatux.edu\Administrator
```

## 🔗 Link Winbind libraries to the system

## Validate libdir path:

```bash
/opt/samba/sbin/smbd -b | grep LIBDIR
```

## Expected:

```bash
LIBDIR: /opt/samba/lib
```

## Create library links (prefer typing manually):

```bash
ln -s /opt/samba/lib/libnss_winbind.so.2 /usr/lib/
```

```bash
ln -s /usr/lib/libnss_winbind.so.2 /usr/lib/libnss_winbind.so
```

## Reload library cache:

```bash
ldconfig
```

## Update nsswitch for Kerberos ticket exchange (add winbind):

```bash
vim /etc/nsswitch.conf
```

```bash
passwd: files winbind
group:  files winbind
```

## Leave the rest untouched

## Join the domain

```bash
net ads join -U Administrator
```

## Result obtained

```bash
Password for [EDUCATUX\Administrator]:
Using short domain name -- EDUCATUX
Joined 'VOIDFILES' to dns domain 'educatux.edu'
```

## 📦 Create the RUNIT services for smbd, winbindd, and optionally nmbd

### Void Linux uses runit as the init system, and its native logger is svlogd, included in the runit base package. No additional packages are required.

## SMBD — Service and Logging

## Create service and log directories

```bash
mkdir -p /etc/sv/smbd/log
mkdir -p /var/log/smbd
```

## Create /etc/sv/smbd/run

```bash
vim /etc/sv/smbd/run
```

## Content

```bash
#!/bin/sh
exec 2>&1
exec /opt/samba/sbin/smbd --foreground --no-process-group
```

## Permission

```bash
chmod +x /etc/sv/smbd/run
```

## Create /etc/sv/smbd/log/run

```bash
vim /etc/sv/smbd/log/run
```

## Content

```bash
#!/bin/sh
exec svlogd -tt /var/log/smbd
```

## Permission

```bash
chmod +x /etc/sv/smbd/log/run
```

## Debug (optional)

```bash
/opt/samba/sbin/smbd -i
```

## Result obtained

```bash
smbd version 4.23.3 started.
Copyright Andrew Tridgell and the Samba Team 1992-2025
daemon 'smbd' : Starting process ...
```

## WINBINDD — Service and Logging

## Create service and log directories

```bash
mkdir -p /etc/sv/winbindd/log
mkdir -p /var/log/winbindd
```

## Create /etc/sv/winbindd/run

```bash
vim /etc/sv/winbindd/run
```

## Content

```bash
#!/bin/sh
exec 2>&1
exec /opt/samba/sbin/winbindd --foreground --no-process-group
```

## Permission

```bash
chmod +x /etc/sv/winbindd/run
```

## Create /etc/sv/winbindd/log/run

```bash
vim /etc/sv/winbindd/log/run
```

## Content

```bash
#!/bin/sh
exec 2>&1
exec /opt/samba/sbin/winbindd --foreground --no-process-group
```

## Permission

```bash
chmod +x /etc/sv/winbindd/log/run
```

## Debug (optional)

```bash
/opt/samba/sbin/winbindd -i
```

## Result obtained

```bash
winbindd version 4.23.3 started.
Copyright Andrew Tridgell and the Samba Team 1992-2025
daemon 'winbindd' : Starting process ...
```

## NMBD — Service and Logging (optional)

### Only enable if your environment requires NetBIOS/SMB1 browsing.

## Create service and log directories

```bash
mkdir -p /etc/sv/nmbd/log
mkdir -p /var/log/nmbd
```

## Create /etc/sv/nmbd/run

```bash
vim /etc/sv/nmbd/run
```

## Content

```bash
#!/bin/sh
exec 2>&1
exec /opt/samba/sbin/nmbd --foreground --no-process-group
```

## Permission

```bash
chmod +x /etc/sv/nmbd/run
```

## Create /etc/sv/nmbd/log/run

```bash
vim /etc/sv/nmbd/log/run
```
 
## Content

```bash
#!/bin/sh
exec svlogd -tt /var/log/nmbd
```

## Permission

```bash
chmod +x /etc/sv/nmbd/log/run
```

## Enable Services

```bash
ln -s /etc/sv/smbd /var/service/
ln -s /etc/sv/winbindd /var/service/
```

## Optional - enable only if using NetBIOS:

```bash
ln -s /etc/sv/nmbd /var/service/
```

## Validate Services

```bash
sv status smbd winbindd nmbd
```

## 🧪 Validate integration

```bash
net ads testjoin
```

## Result obtained

```bash
Join is OK
```

```bash
wbinfo -u
```

## Result obtained

```bash
guest
krbtgt
administrator```

```bash
wbinfo -g
```

## Result obtained

```bash
enterprise read-only domain controllers
protected users
domain controllers
domain guests
read-only domain controllers
schema admins
dnsupdateproxy
domain admins
group policy creator owners
ras and ias servers
dnsadmins
allowed rodc password replication group
enterprise admins
cert publishers
domain users
denied rodc password replication group
domain computers
```

```bash
wbinfo --ping-dc
```

## Result obtained

```bash
checking the NETLOGON for domain[EDUCATUX] dc connection to "voiddc.educatux.edu" succeeded
```

## ✅ FINAL SUMMARY

## 🎉 Congratulations — you have successfully deployed a fully functional File Server on Void Linux!

## 👉 REMEMBER: While Samba4 can be managed via CLI, it was designed to be managed via RSAT Remote Server Administration Tools, which can be installed on a Windows 10 machine without issues!

---

🎯 THAT'S ALL FOLKS!

👉 Contact: zerolies@disroot.org

👉 https://t.me/z3r0l135
