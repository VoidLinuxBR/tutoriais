# Primary Domain Controller (Active Directory) running Samba4 under Void Linux Server ;D

## 🎯 Objective - Upload a Primary Domain Controller on Void Linux (glibc) compiling Samba4 from source code, configuring internal DNS, Kerberos, AD integration, ACLs, services and the entire stack necessary to control network clients.

### 🔧 ADAPT the tutorial to YOUR reality, obviously!

## 📡 Local network layout

- Domain: EDUCATUX.EDU
- Hostname: pdc01
- Firewall 192.168.70.254 (DNS/GW)
- Ip: 192.168.70.253

---

## The default installation of Void Linux will not be covered in this tutorial.

## Change the default shell of Void, after installed

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
 bind ldns pkg-config vim
```

## ⚠️ ATTENTION: The compiled Samba4 includes the Heimdal kerberos code, built-in (internal KDC) by default, but does not include Kerberos clients. In this case, the repository provides binary packages from MIT, which can be installed without any problems or interference with the default kerberos heimdal, compiled on the Domain Controller. The packages are: mit-krb5 mit-krb5-client mit-krb5-devel. HOWEVER, you MUST NOT under any circumstances install the krb5-server binary package from the repository, which would cause a competing service with the Heimdal kerberos, internal to Samba4!

## The services provided by MIT-krb5 customers are at:

```bash
/usr/bin/kinit
/usr/bin/klist
/usr/bin/kvno
/usr/bin/kdestroy
```

## 🖥️ Setar hostname

```bash
echo "pdc01" > /etc/hostname
```

## 🏠 /etc/hosts

```bash
vim /etc/hosts
```

## Content:

```bash
127.0.0.1      localhost
127.0.1.1      pdc01.educatux.edu pdc01
192.168.70.253 pdc01.educatux.edu pdc01
```

## 🌐 Configure fixed IP

### 👉 We will use Void's default method, /etc/dhcpcd.conf

```bash
vim /etc/dhcpcd.conf
```

## Add ip, gateway and dns:## 🎯 Purpose

```bash
interface eth0
static ip_address=192.168.70.253/24
static routers=192.168.70.254
static domain_name_servers=192.168.70.254
```

## Restart the network interface:

```bash
sv restart dhcpcd
```

## 🌐 Set temporary DNS (router) BEFORE provisioning

```bash
echo "nameserver 192.168.70.254" > /etc/resolv.conf
```

## Lock resolv.conf configuration

```bash
chattr +i /etc/resolv.conf
```

## 🔍 Validate address assigned to the network interface

```bash
ip -c addr
```

```bash
ip -br link
```

## 📥 Download and unzip the Samba4 source code

```bash
wget https://download.samba.org/pub/samba/samba-4.23.4.tar.gz
```

```bash
tar -xvzf samba-4.23.4.tar.gz
```

## Compile and install the source code

```bash
cd samba-4.23.4
```

```bash
./configure --prefix=/opt/samba
```

```bash
make -j$(nproc) && make install
```

## Comment:

- Void does not interfere with the installation, as Samba is compiled in /opt/samba.
- Make -j speeds up compilation a lot, anyway, go get a coffee.
- After installing, the compiled Samba4 has no services created in the runit.
- We will create the services manually.

## 📁 Add Samba4 to the System PATH and reread the environment

```bash
echo 'export PATH=/opt/samba/bin:/opt/samba/sbin:$PATH' >> /etc/profile
```

```bash
source /etc/profile
```

## Test the insertion of the Samba4 PATH in the Operating System

```bash
samba-tool -V
```

## Result:

```bash
4.23.4
```

🏰 Provision the SAMBA4 domain (Creating the PDC itself)

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

### Samba4 will create:

```bash
/opt/samba/etc/smb.conf
/opt/samba/private/*
/opt/samba/var/locks/sysvol
```

## In summary Samba4:

- Creates the AD forest, primary DC, internal DNS and accounts DB.
- Defines domain, realm, 2016 functional level and Administrator password.
- Void doesn't install any native Samba, so there is no conflict.
- After this, the DNS becomes the PDC itself, needing to adjust /etc/resolv.conf to 127.0.0.1.

## ⚙️ Validate Active Directory 2016 functional level

```bash
samba-tool domain level show
```

## Result:

```bash
Domain and forest function level for domain 'DC=educatux,DC=edu'
Forest function level: (Windows) 2016
Domain function level: (Windows) 2016
Lowest function level of a DC: (Windows) 2016
```

## 🧪 Manually test the AD DC service before creating the service

```bash
/opt/samba/sbin/samba -i -M single
```

* -i → foreground
* -M single → single-process model (does not trigger daemon forking outside of runit control)

## If everything is ok, you will see:

```bash
Completed DNS update check OK
Completed SPN update check OK
Registered EDUCATUX<1c> ...
```

## CTRL+C to exit

## 📦 Create the samba-ad-dc RUNIT service to upload AD at boot

## ⚠️ This part is very important. Delete old remains IF READJUSTING a pre-existing Server!!

```bash
sv stop samba-ad-dc 2>/dev/null
rm -rf /var/service/samba-ad-dc
rm -rf /var/service/a-ad-dc
rm -rf /etc/sv/samba-ad-dc
rm -rf /etc/sv/a-ad-dc
rm -rf /etc/sv/*/supervise
rm -rf /var/service/*/supervise
```

## Now let's create the samba-ad-dc services and permissions with logs, so that runit can be loaded at system startup:

## Create the service structure first and foremost

```bash
mkdir -p /etc/sv/samba-ad-dc/log
mkdir -p /var/log/samba-ad-dc
```

## Create the run service

```bash
cat > /etc/sv/samba-ad-dc/run << 'EOF'
#!/bin/sh
exec 2>&1
exec /opt/samba/sbin/samba -i -M single --debuglevel=3
EOF
```

## Set the run service permission

```bash
chmod +x /etc/sv/samba-ad-dc/run
```

## Create the log file

```bash
cat > /etc/sv/samba-ad-dc/log/run << 'EOF'
#!/bin/sh
exec svlogd -tt /var/log/samba-ad-dc
EOF
```

## Set log/run permission

```bash
chmod +x /etc/sv/samba-ad-dc/log/run
```

## Enable the samba-ad-dc service to run at boot:

```bash
ln -sf /etc/sv/samba-ad-dc/ /var/service/
```

## Validate if it is running

```bash
sv status samba-ad-dc
```

## You should see something like:

```bash
run: samba-ad-dc: (pid 28032) 4s; run: log: (pid 28031) 4s
```

```bash
samba-tool processes
```

## Result received:

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

## Validate logs online:

```bash
tail -f /var/log/samba-ad-dc/current
```

## The correct output will be something like this:

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

## The Domain Controller will need to be the Time Server of the local network, as with a discrepancy of 5 minutes, Kerberos will no longer authenticate the client

## Install the Chrony Server package

```bash
xbps-install -Syu chrony
```

## Edit the Server file, replace time synchronization repositories and release internal network queries

```bash
vim /etc/chrony.conf
```

### Point out public time servers in Brazil

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

## Add the chronyd service to RUNIT start

```bash
ln -sf /etc/sv/chronyd/ /var/service/
```

## Restart TimeServer:

```bash
sv restart chronyd
```

## Validate the Servers, they are cyclical and random during queries

```bash
chronyc sources -v
```

## 🔐 Create Kerberos file

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

## 🧭 Unlock and reset /etc/resolv.conf AFTER provisioning, and point to the PDC itself

```bash
chattr -i /etc/resolv.conf
```

```bash
vim /etc/resolv.conf
```

## Content:

```bash
domain educatux.edu
search educatux.edu
nameserver 127.0.0.1
```

## Lock the file again:

```bash
chattr +i /etc/resolv.conf
```

## 🔗 Link Winbind libraries on the System

## Validate libdir paths:

```bash
/opt/samba/sbin/smbd -b | grep LIBDIR
```

## Expected output:

```bash
LIBDIR: /opt/samba/lib
```

## Create links between libraries. Prefer typing manually instead of copying and pasting here.

```bash
ln -sf /opt/samba/lib/libnss_winbind.so.2 /usr/lib/
```

```bash
ln -sf /usr/lib/libnss_winbind.so.2 /usr/lib/libnss_winbind.so
```

## Reread the configuration with the new linked libraries

```bash
ldconfig
```

## Validate the effectiveness of the Kerberos ticket exchange, adding winbind to the two lines of nsswhitch (passwd and group):

```bash
vim /etc/nsswitch.conf
```

```bash
passwd: files winbind
group:  files winbind
```

### The rest of the file remains as is

## 📝 Validate the smb.conf created automatically by provisioning

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

## 🔍 Now we will validate important PDC services such as DNS, SMB, Winbind and Kerberos

```bash
ps aux | grep samba
```

## Result received:

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

## Result received:

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

## Result received:

```bash
EDUCATUX\administrator
EDUCATUX\guest
EDUCATUX\krbtgt
```

```bash
wbinfo -g
```

## Result received:

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

## Result received:

```bash
EDUCATUX\domain admins:x:3000004:
```

```bash
smbclient -L localhost -U Administrator
```

## Result received:

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

## Result received:

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

## Result received:

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

## 🔐 Disable password complexity for domain users (facilitate lab testing - Unsafe for production!)

```bash
samba-tool domain passwordsettings set --complexity=off
samba-tool domain passwordsettings set --history-length=0
samba-tool domain passwordsettings set --min-pwd-length=0
samba-tool domain passwordsettings set --min-pwd-age=0
samba-tool user setexpiry Administrator --noexpiry
```

## Reread Samba4 settings

```bash
smbcontrol all reload-config
```

## 🧪 Validate Kerberos ticket exchange

```bash
kinit Administrator@EDUCATUX.EDU
```

```bash
klist
```

## Result received:

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

## Result received:

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

## Result obtained:

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

### ✅ FINAL SUMMARY

## 🎉 Congratulations — you've just set up a fully functional 2016 AD domain on Void Linux!

### 👉 REMEMBER: Samba4, despite being able to be managed by command line, was designed to be managed by remote Server Management tools - RSAT, which can be installed on a Windows 10 machine, without any problems!

## Now you can:

- join Windows machines to the domain
- use GPOs
- test replication (when creating a DC2)
- create users/groups via RSAT
- configure sysvol replication (with Rsync or the new samba-gpupdate)
- add DNS forwarders
- enable DFS
- create member File Server
- etc.

---

🎯 THAT'S ALL FOLKS!

👉 Contact: zerolies@disroot.org
👉 https://t.me/z3r0l135
