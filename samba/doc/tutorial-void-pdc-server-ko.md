# Void Linux Server에서 Samba4를 실행하는 기본 도메인 컨트롤러(Active Directory) ;D

## 🎯 목표 - 소스 코드에서 Samba4를 컴파일하고 내부 DNS, Kerberos, AD 통합, ACL, 서비스 및 네트워크 클라이언트를 제어하는 데 필요한 전체 스택을 구성하는 Void Linux(glibc)에 기본 도메인 컨트롤러를 업로드합니다.

### 🔧 튜토리얼을 현실에 맞게 조정하세요!

## 📡 로컬 네트워크 레이아웃

- 도메인: EDUCATUX.EDU
- 호스트 이름: pdc01
- 방화벽 192.168.70.254(DNS/GW)
- 아이피: 192.168.70.253

---

## Void Linux의 기본 설치는 이 튜토리얼에서 다루지 않습니다.

## 설치 후 Void의 기본 셸을 변경합니다.

```bash
chsh -s /bin/bash
```

## 🧩 Void에서 Samba4를 컴파일하려면 종속성 패키지를 설치하세요.

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

## ⚠️ 주의: 컴파일된 Samba4에는 기본적으로 내장된(내부 KDC) Heimdal kerberos 코드가 포함되어 있지만 Kerberos 클라이언트는 포함되지 않습니다. 이 경우 리포지토리는 도메인 컨트롤러에서 컴파일된 기본 kerberos heimdal과의 문제나 간섭 없이 설치할 수 있는 MIT의 바이너리 패키지를 제공합니다. 패키지는 mit-krb5 mit-krb5-client mit-krb5-devel입니다. 그러나 어떤 상황에서도 저장소에서 krb5-server 바이너리 패키지를 설치하면 안 됩니다. 그러면 Samba4 내부의 Heimdal kerberos와 서비스 경쟁이 발생할 수 있습니다!

## MIT-krb5 고객이 제공하는 서비스는 다음과 같습니다.

```bash
/usr/bin/kinit
/usr/bin/klist
/usr/bin/kvno
/usr/bin/kdestroy
```

## 🖥️ 세타르 호스트 이름

```bash
echo "pdc01" > /etc/hostname
```

## 🏠 /etc/hosts

```bash
vim /etc/hosts
```

## 콘텐츠:

```bash
127.0.0.1      localhost
127.0.1.1      pdc01.educatux.edu pdc01
192.168.70.253 pdc01.educatux.edu pdc01
```

## 🌐 고정 IP 구성

### 👉 Void의 기본 방법인 /etc/dhcpcd.conf를 사용하겠습니다.

```bash
vim /etc/dhcpcd.conf
```

## IP, 게이트웨이 및 DNS 추가:## 🎯 목적

```bash
interface eth0
static ip_address=192.168.70.253/24
static routers=192.168.70.254
static domain_name_servers=192.168.70.254
```

## 네트워크 인터페이스를 다시 시작합니다.

```bash
sv restart dhcpcd
```

## 🌐 프로비저닝 전 임시 DNS(라우터) 설정

```bash
echo "nameserver 192.168.70.254" > /etc/resolv.conf
```

## resolv.conf 구성 잠금

```bash
chattr +i /etc/resolv.conf
```

## 🔍 네트워크 인터페이스에 할당된 주소 확인

```bash
ip -c addr
```

```bash
ip -br link
```

## 📥 Samba4 소스 코드 다운로드 및 압축 풀기

```bash
wget https://download.samba.org/pub/samba/samba-4.23.4.tar.gz
```

```bash
tar -xvzf samba-4.23.4.tar.gz
```

## 소스코드 컴파일 및 설치

```bash
cd samba-4.23.4
```

```bash
./configure --prefix=/opt/samba
```

```bash
make -j$(nproc) && make install
```

## 논평:

- Samba는 /opt/samba에 컴파일되므로 Void는 설치를 방해하지 않습니다.
- Make -j는 컴파일 속도를 크게 높여줍니다. 어쨌든 가서 커피를 마시세요.
- 설치 후 컴파일된 Samba4에는 runit에서 생성된 서비스가 없습니다.
- 서비스를 수동으로 생성하겠습니다.

## 📁 Samba4를 시스템 PATH에 추가하고 환경을 다시 읽습니다.

```bash
echo 'export PATH=/opt/samba/bin:/opt/samba/sbin:$PATH' >> /etc/profile
```

```bash
source /etc/profile
```

## 운영 체제에서 Samba4 PATH 삽입 테스트

```bash
samba-tool -V
```

## 결과:

```bash
4.23.4
```

🏰 SAMBA4 도메인 프로비저닝(PDC 자체 생성)

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

### Samba4는 다음을 생성합니다:

```bash
/opt/samba/etc/smb.conf
/opt/samba/private/*
/opt/samba/var/locks/sysvol
```

## 요약하면 Samba4:

- AD 포리스트, 기본 DC, 내부 DNS 및 계정 DB를 생성합니다.
- 도메인, 영역, 2016 기능 수준 및 관리자 비밀번호를 정의합니다.
- Void는 기본 Samba를 설치하지 않으므로 충돌이 없습니다.
- 그 후에는 DNS가 PDC 자체가 되므로 /etc/resolv.conf를 127.0.0.1로 조정해야 합니다.

## ⚙️ Active Directory 2016 기능 수준 확인

```bash
samba-tool domain level show
```

## 결과:

```bash
Domain and forest function level for domain 'DC=educatux,DC=edu'
Forest function level: (Windows) 2016
Domain function level: (Windows) 2016
Lowest function level of a DC: (Windows) 2016
```

## 🧪 서비스를 만들기 전에 AD DC 서비스를 수동으로 테스트하세요.

```bash
/opt/samba/sbin/samba -i -M single
```

* -i → 전경
* -M 단일 → 단일 프로세스 모델(runit 제어 외부에서 데몬 분기를 트리거하지 않음)

## 모든 것이 정상이면 다음이 표시됩니다.

```bash
Completed DNS update check OK
Completed SPN update check OK
Registered EDUCATUX<1c> ...
```

## 종료하려면 CTRL+C

## 📦 부팅 시 AD를 업로드하기 위한 samba-ad-dc RUNIT 서비스 생성

## ⚠️ 이 부분이 매우 중요합니다. 기존 서버를 재조정하는 경우 오래된 유적을 삭제하세요!!

```bash
sv stop samba-ad-dc 2>/dev/null
rm -rf /var/service/samba-ad-dc
rm -rf /var/service/a-ad-dc
rm -rf /etc/sv/samba-ad-dc
rm -rf /etc/sv/a-ad-dc
rm -rf /etc/sv/*/supervise
rm -rf /var/service/*/supervise
```

## 이제 시스템 시작 시 runit이 로드될 수 있도록 로그와 함께 samba-ad-dc 서비스 및 권한을 생성해 보겠습니다.

## 무엇보다 서비스 구조를 만들어라

```bash
mkdir -p /etc/sv/samba-ad-dc/log
mkdir -p /var/log/samba-ad-dc
```

## 실행 서비스 만들기

```bash
cat > /etc/sv/samba-ad-dc/run << 'EOF'
#!/bin/sh
exec 2>&1
exec /opt/samba/sbin/samba -i -M single --debuglevel=3
EOF
```

## 서비스 실행 권한 설정

```bash
chmod +x /etc/sv/samba-ad-dc/run
```

## 로그 파일 만들기

```bash
cat > /etc/sv/samba-ad-dc/log/run << 'EOF'
#!/bin/sh
exec svlogd -tt /var/log/samba-ad-dc
EOF
```

## 로그/실행 권한 설정

```bash
chmod +x /etc/sv/samba-ad-dc/log/run
```

## 부팅 시 samba-ad-dc 서비스가 실행되도록 활성화합니다.

```bash
ln -sf /etc/sv/samba-ad-dc/ /var/service/
```

## 실행 중인지 확인

```bash
sv status samba-ad-dc
```

## 다음과 같은 내용이 표시됩니다.

```bash
run: samba-ad-dc: (pid 28032) 4s; run: log: (pid 28031) 4s
```

```bash
samba-tool processes
```

## 받은 결과:

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

## 온라인으로 로그 유효성을 검사합니다.

```bash
tail -f /var/log/samba-ad-dc/current
```

## 올바른 출력은 다음과 같습니다.

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

## 🕒 NTP / Chrony 서버

## 도메인 컨트롤러는 로컬 네트워크의 시간 서버가 되어야 합니다. 5분의 불일치로 인해 Kerberos는 더 이상 클라이언트를 인증하지 않습니다.

## Chrony 서버 패키지 설치

```bash
xbps-install -Syu chrony
```

## 서버 파일 편집, 시간 동기화 저장소 교체, 내부 네트워크 쿼리 해제

```bash
vim /etc/chrony.conf
```

### 브라질의 공개 시간 서버를 지적하세요

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

## RUNIT start에 chronyd 서비스 추가

```bash
ln -sf /etc/sv/chronyd/ /var/service/
```

## 타임서버를 다시 시작하세요:

```bash
sv restart chronyd
```

## 서버의 유효성을 검사합니다. 쿼리 중에 주기적이며 무작위입니다.

```bash
chronyc sources -v
```

## 🔐 Kerberos 파일 생성

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

## 🧭 프로비저닝 후 /etc/resolv.conf를 잠금 해제하고 재설정하고 PDC 자체를 가리킵니다.

```bash
chattr -i /etc/resolv.conf
```

```bash
vim /etc/resolv.conf
```

## 콘텐츠:

```bash
domain educatux.edu
search educatux.edu
nameserver 127.0.0.1
```

## 파일을 다시 잠급니다.

```bash
chattr +i /etc/resolv.conf
```

## 🔗 시스템에서 Winbind 라이브러리 연결

## libdir 경로 검증:

```bash
/opt/samba/sbin/smbd -b | grep LIBDIR
```

## 예상 출력:

```bash
LIBDIR: /opt/samba/lib
```

## 라이브러리 간의 링크를 만듭니다. 여기에 복사하여 붙여넣는 것보다 직접 입력하는 것이 좋습니다.

```bash
ln -sf /opt/samba/lib/libnss_winbind.so.2 /usr/lib/
```

```bash
ln -sf /usr/lib/libnss_winbind.so.2 /usr/lib/libnss_winbind.so
```

## 새로 연결된 라이브러리로 구성을 다시 읽습니다.

```bash
ldconfig
```

## nsswhitch의 두 줄(passwd 및 group)에 winbind를 추가하여 Kerberos 티켓 교환의 효율성을 검증합니다.

```bash
vim /etc/nsswitch.conf
```

```bash
passwd: files winbind
group:  files winbind
```

### 나머지 파일은 그대로 유지됩니다.

## 📝 프로비저닝을 통해 자동으로 생성된 smb.conf의 유효성을 검사합니다.

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

## 🔍 이제 DNS, SMB, Winbind 및 Kerberos와 같은 중요한 PDC 서비스를 검증하겠습니다.

```bash
ps aux | grep samba
```

## 받은 결과:

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

## 받은 결과:

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

## 받은 결과:

```bash
EDUCATUX\administrator
EDUCATUX\guest
EDUCATUX\krbtgt
```

```bash
wbinfo -g
```

## 받은 결과:

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

## 받은 결과:

```bash
EDUCATUX\domain admins:x:3000004:
```

```bash
smbclient -L localhost -U Administrator
```

## 받은 결과:

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

## 받은 결과:

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

## 받은 결과:

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

## 🔐 도메인 사용자의 비밀번호 복잡성을 비활성화합니다(실험실 테스트 촉진 - 프로덕션에는 안전하지 않음!)

```bash
samba-tool domain passwordsettings set --complexity=off
samba-tool domain passwordsettings set --history-length=0
samba-tool domain passwordsettings set --min-pwd-length=0
samba-tool domain passwordsettings set --min-pwd-age=0
samba-tool user setexpiry Administrator --noexpiry
```

## Samba4 설정 다시 읽기

```bash
smbcontrol all reload-config
```

## 🧪 Kerberos 티켓 교환 확인

```bash
kinit Administrator@EDUCATUX.EDU
```

```bash
klist
```

## 받은 결과:

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

## 받은 결과:

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

## 얻은 결과:

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

### ✅ 최종 요약

## 🎉 축하합니다. Void Linux에서 모든 기능을 갖춘 2016 AD 도메인을 설정했습니다!

### 👉 기억하세요: Samba4는 명령줄로 관리할 수 있음에도 불구하고 Windows 10 시스템에 아무 문제 없이 설치할 수 있는 원격 서버 관리 도구인 RSAT로 관리되도록 설계되었습니다!

## 이제 다음을 수행할 수 있습니다.

- Windows 시스템을 도메인에 가입
- GPO 사용
- 테스트 복제(DC2 생성 시)
- RSAT를 통해 사용자/그룹 생성
- sysvol 복제 구성(Rsync 또는 새로운 samba-gpupdate 사용)
- DNS 전달자 추가
- DFS 활성화
- 구성원 파일 서버 생성
- 등.

---

🎯 그게 전부입니다!

👉 문의: zerolies@disroot.org
👉 https://t.me/z3r0l135
