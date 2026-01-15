
# Void Linux 서버에서 Samba4를 실행하는 파일 서버 ;D

## 🎯 목표 – 네트워크 클라이언트를 제공하기 위해 파일 서버에 필요한 소스, AD 통합, ACL, 서비스 및 전체 스택에서 Samba4를 컴파일하여 Void Linux(glibc)에 파일 서버를 배포합니다.

## 🔧 QEMU/Virtmanager 및 Proxmox를 사용한 네트워킹 연구실. 자신의 환경에 맞게 튜토리얼을 조정하세요.

---

## 📡 로컬 네트워크 레이아웃

- 도메인: EDUCATUX.EDU

- 호스트 이름: 파일 서버

- 방화벽 192.168.70.254(DNS/GW)

- IP: 192.168.70.251

## 보이드 리눅스 설치

## Void의 기본 셸 변경

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

## 🖥️ 호스트 이름 설정

```bash
echo "fileserver" > /etc/hostname
```

## 🏠 /etc/hosts

```bash
vim /etc/hosts
```

## 콘텐츠:

```bash
127.0.0.1      localhost
127.0.1.1      fileserver.educatux.edu fileserver
192.168.70.251 fileserver.educatux.edu fileserver
```

## 🌐 고정 IP 구성

## 👉 표준 Void 방법인 /etc/dhcpcd.conf를 사용합니다.

```bash
vim /etc/dhcpcd.conf
```

## IP, 게이트웨이(라우터) 및 DNS(AD)를 추가합니다.

```bash
interface eth0
static ip_address=192.168.70.251/24
static routers=192.168.70.254
static domain_name_servers=192.168.70.253
```

## 네트워크 인터페이스를 다시 시작합니다.

```bash
sv restart dhcpcd
```

## 🧭 DNS 주소 설정 - PDC를 가리킵니다.

```bash
vim /etc/resolv.conf
```

## 콘텐츠:

```bash
domain educatux.edu
search educatux.edu
nameserver 192.168.70.253
```

## resolv.conf 잠금

```bash
chattr +i /etc/resolv.conf
```

## 🔍 네트워크 인터페이스 주소 확인

```bash
ip -c addr
ip -br link
```

## 📥 Samba4 소스 코드 다운로드 및 추출

```bash
wget https://download.samba.org/pub/samba/samba-4.23.4.tar.gz
```

```bash
tar -xvzf samba-4.23.4.tar.gz
```

## 소스에서 컴파일 및 설치

```bash
cd samba-4.23.4
```

```bash
./configure --prefix=/opt/samba
```

```bash
make -j$(nproc) && make install
```

## 참고:

- Samba는 /opt/samba에 컴파일되므로 Void가 방해하지 않습니다.

- make -j는 컴파일 속도를 크게 높여줍니다. 그래도 가서 커피를 마시세요.

- 설치 후 컴파일된 Samba4에는 runit 서비스가 없습니다.

- 서비스를 수동으로 생성하겠습니다.

## 📁 시스템 PATH에 Samba4를 추가하고 환경을 다시 로드하세요.

```bash
echo 'export PATH=/opt/samba/bin:/opt/samba/sbin:$PATH' >> /etc/profile
```

```bash
source /etc/profile
```

## OS에 Samba4 PATH 삽입 테스트

```bash
samba-tool -V
```

## 산출:

```bash
4.23.4
```

## ⚠️ 경고: 파일 서버에서 프로비저닝 명령을 사용하지 마십시오!

## 📝 smb.conf 파일 생성

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

## 로그 파일 만들기

```bash
mkdir /opt/samba/var
```

## 📂 공유 경로 만들기

```bash
sudo mkdir -p /srv/samba/public
sudo chown -R root:root /srv/samba/public
sudo chmod -R 0770 /srv/samba/public
```

## Samba4 구성 다시 로드

```bash
smbcontrol all reload-config
```

## 🕒 NTP / Chrony 서버

## 도메인 컨트롤러는 로컬 시간 서버여야 합니다. 5분 간격으로 Kerberos가 더 이상 클라이언트를 인증하지 않기 때문입니다.

## 크로니 설치

```bash
xbps-install -Syu chrony
```

## 구성 수정 및 내부 네트워크 허용

```bash
vim /etc/chrony.conf
```

## 시간 서버에서 도메인 제어를 설정합니다.

```bash
# Comment the external line
#pool pool.ntp.org iburst

# PDC Time Servers
server 192.168.70.253 iburst
```

## runit에서 chronyd 활성화

```bash
ln -sf /etc/sv/chronyd/ /var/service/
```

## 서비스 다시 시작:

```bash
sv restart chronyd
```

## 서버 검증:

```bash
chronyc sources -v
```

## 🔐 Kerberos 파일 만들기 - PDC를 가리킵니다.

```bash
vim /etc/krb5.conf
```

## 포함하는

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

## Kerberos 테스트

```bash
kinit Administrator
```

```bash
klist
```

## 얻은 결과

```bash
Ticket cache: FILE:/tmp/krb5cc_0
Default principal: administrator@EDUCATUX.EDU

Valid starting       Expires              Service principal
11/12/2025 09:43:40  11/12/2025 19:43:40  krbtgt/EDUCATUX.EDU@EDUCATUX.EDU
	renew until 12/12/2025 09:43:36
```

## 🔗 Winbind 라이브러리를 시스템에 연결

## libdir 경로 확인:

```bash
/opt/samba/sbin/smbd -b | grep LIBDIR
```

## 예상되는:

```bash
LIBDIR: /opt/samba/lib
```

## 라이브러리 링크 만들기(수동으로 입력하는 것을 선호함):

```bash
ln -sf /opt/samba/lib/libnss_winbind.so.2 /usr/lib/
```

```bash
ln -sf /usr/lib/libnss_winbind.so.2 /usr/lib/libnss_winbind.so
```

## 라이브러리 캐시 다시 로드:

```bash
ldconfig
```

## Kerberos 티켓 교환을 위한 nsswitch 업데이트(winbind 추가):

```bash
vim /etc/nsswitch.conf
```

```bash
passwd: files winbind
group:  files winbind
```

## 나머지는 그대로 두세요

## 도메인에 가입하세요

```bash
net ads join -U Administrator
```

## 얻은 결과

```bash
Password for [EDUCATUX\Administrator]:
Using short domain name -- EDUCATUX
Joined 'VOIDFILES' to dns domain 'educatux.edu'
```

## 📦 smbd, winbindd 및 선택적으로 nmbd에 대한 RUNIT 서비스 생성

### Void Linux는 runit을 init 시스템으로 사용하며 기본 로거는 runit 기본 패키지에 포함된 svlogd입니다. 추가 패키지는 필요하지 않습니다.

## SMBD — 서비스 및 로깅

## 서비스 및 로그 디렉터리 생성

```bash
mkdir -p /etc/sv/smbd/log
mkdir -p /var/log/smbd
```

## /etc/sv/smbd/run 생성

```bash
vim /etc/sv/smbd/run
```

## 콘텐츠

```bash
#!/bin/sh
exec 2>&1
exec /opt/samba/sbin/smbd --foreground --no-process-group
```

## 허가

```bash
chmod +x /etc/sv/smbd/run
```

## /etc/sv/smbd/log/run 생성

```bash
vim /etc/sv/smbd/log/run
```

## 콘텐츠

```bash
#!/bin/sh
exec svlogd -tt /var/log/smbd
```

## 허가

```bash
chmod +x /etc/sv/smbd/log/run
```

## 디버그(선택사항)

```bash
/opt/samba/sbin/smbd -i
```

## 얻은 결과

```bash
smbd version 4.23.4 started.
Copyright Andrew Tridgell and the Samba Team 1992-2025
daemon 'smbd' : Starting process ...
```

## WINBINDD — 서비스 및 로깅

## 서비스 및 로그 디렉터리 생성

```bash
mkdir -p /etc/sv/winbindd/log
mkdir -p /var/log/winbindd
```

## /etc/sv/winbindd/run 생성

```bash
vim /etc/sv/winbindd/run
```

## 콘텐츠

```bash
#!/bin/sh
exec 2>&1
exec /opt/samba/sbin/winbindd --foreground --no-process-group
```

## 허가

```bash
chmod +x /etc/sv/winbindd/run
```

## /etc/sv/winbindd/log/run 생성

```bash
vim /etc/sv/winbindd/log/run
```

## 콘텐츠

```bash
#!/bin/sh
exec 2>&1
exec /opt/samba/sbin/winbindd --foreground --no-process-group
```

## 허가

```bash
chmod +x /etc/sv/winbindd/log/run
```

## 디버그(선택사항)

```bash
/opt/samba/sbin/winbindd -i
```

## 얻은 결과

```bash
winbindd version 4.23.4 started.
Copyright Andrew Tridgell and the Samba Team 1992-2025
daemon 'winbindd' : Starting process ...
```

## NMBD - 서비스 및 로깅(선택 사항)

### 환경에 NetBIOS/SMB1 검색이 필요한 경우에만 활성화하십시오.

## 서비스 및 로그 디렉터리 생성

```bash
mkdir -p /etc/sv/nmbd/log
mkdir -p /var/log/nmbd
```

## /etc/sv/nmbd/run 생성

```bash
vim /etc/sv/nmbd/run
```

## 콘텐츠

```bash
#!/bin/sh
exec 2>&1
exec /opt/samba/sbin/nmbd --foreground --no-process-group
```

## 허가

```bash
chmod +x /etc/sv/nmbd/run
```

## /etc/sv/nmbd/log/run 생성

```bash
vim /etc/sv/nmbd/log/run
```
 
## 콘텐츠

```bash
#!/bin/sh
exec svlogd -tt /var/log/nmbd
```

## 허가

```bash
chmod +x /etc/sv/nmbd/log/run
```

## 서비스 활성화

```bash
ln -sf /etc/sv/smbd /var/service/
ln -sf /etc/sv/winbindd /var/service/
```

## 선택 사항 - NetBIOS를 사용하는 경우에만 활성화합니다.

```bash
ln -sf /etc/sv/nmbd /var/service/
```

## 서비스 검증

```bash
sv status smbd winbindd nmbd
```

## 🧪 통합 확인

```bash
net ads testjoin
```

## 얻은 결과

```bash
Join is OK
```

```bash
wbinfo -u
```

## 얻은 결과

```bash
guest
krbtgt
administrator```

```bash
wbinfo -g
```

## Result obtained

```bash
엔터프라이즈 읽기 전용 도메인 컨트롤러
보호된 사용자
도메인 컨트롤러
도메인 손님
읽기 전용 도메인 컨트롤러
스키마 관리자
dnsupdate프록시
도메인 관리자
그룹 정책 작성자 소유자
ras 및 ias 서버
dnsadmins
허용된 Rodc 비밀번호 복제 그룹
기업 관리자
인증서 게시자
도메인 사용자
거부된 RODC 비밀번호 복제 그룹
도메인 컴퓨터
```

```bash
wbinfo --ping-dc
```

## Result obtained

```bash
"voiddc.educatux.edu"에 대한 도메인[EDUCATUX] DC 연결에 대한 NETLOGON을 확인하는 데 성공했습니다.
```

## ✅ FINAL SUMMARY

## 🎉 Congratulations — you have successfully deployed a fully functional File Server on Void Linux!

## 👉 REMEMBER: While Samba4 can be managed via CLI, it was designed to be managed via RSAT Remote Server Administration Tools, which can be installed on a Windows 10 machine without issues!

---

🎯 THAT'S ALL FOLKS!

👉 Contact: zerolies@disroot.org

👉 https://t.me/z3r0l135
