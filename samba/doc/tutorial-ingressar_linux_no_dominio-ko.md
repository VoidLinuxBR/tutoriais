# Linux Mint를 도메인에 가입하기 위한 LightDM 구성

## 🎯 목표 NSS/PAM 통합을 통해 Winbind를 사용하여 Linux를 Samba4 도메인에 연결

---

## 전제조건

- 도메인 컨트롤러(PDC)로서의 Samba4
- PDC(192.168.70.253)와 일치하는 DNS 및 시간을 갖춘 Linux
- 서버에 대한 연결

---

## 🛠️ 단계

## 1. 필수 패키지 설치

```bash
sudo apt update && sudo apt install samba winbind libpam-winbind libnss-winbind krb5-user
```

## 2. /etc/krb5.conf 구성

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

## 3. /etc/samba/smb.conf 구성

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

## 4. /etc/nsswitch.conf 구성

```bash
passwd:         compat winbind
group:          compat winbind
shadow:         compat
```

## 5. 도메인 가입

```bash
sudo net ads join -U Administrator
```

## 6. 시스템 재부팅

```bash
sudo reboot
```

## 7. 수표

```bash
sudo net ads testjoin
wbinfo -u
wbinfo -g
getent passwd usuario
```

## 8. HOME 디렉토리 자동 생성


## /etc/pam.d/common-session을 편집하고 다음을 추가합니다.

```bash
session required pam_mkhomedir.so skel=/etc/skel umask=0022
```

## 9. 서비스 다시 시작

```bash
sudo systemctl restart smbd nmbd winbind
sudo systemctl enable winbind
```

## 10. 시간 동기화

```bash
sudo timedatectl set-ntp true
```

## Mint와 마찬가지로 Lightdm 사용자라면 로컬 사용자가 아닌 네트워크 사용자로 로그인하도록 설정하세요.

## 🛠️ 단계별: 도메인 사용자를 허용하도록 LightDM 구성

## 1. LightDM 구성 파일 편집

```bash
sudo vim /etc/lightdm/lightdm.conf
```

## 파일에서 다음 줄을 편집합니다.

```bash
[Seat:*]
greeter-show-manual-login=true
greeter-hide-users=true
allow-guest=false
```

## 설명:

- Greeting-show-manual-login=true: 사용자 이름을 수동으로 입력할 수 있습니다.
- Greetingr-hide-users=true: 로컬 사용자 목록을 숨깁니다(기업 환경에 유용함).
- allow-guest=false: 게스트 로그인을 방지합니다(보안을 위해).

## 2. PAM이 도메인 사용자를 허용하는지 확인하세요.

## SSSD 또는 Winbind를 사용한 경우 PAM이 이미 올바르게 통합되어 있어야 합니다. 하지만 홈 모듈이 있는지 확인하세요.

```bash
sudo vim /etc/pam.d/common-session
```

## 이 줄이 있는지 확인하거나 추가하세요.

```bash
session required pam_mkhomedir.so skel=/etc/skel umask=0022
```

## 3. LightDM을 다시 시작하세요

```bash
sudo timedatectl set-ntp true
```

## ⚠️ Ubuntu 기반 Linux MInt의 경우 systemd-resolved를 비활성화하여 DNS를 수동으로 제어할 수 있습니다.

```bash
systemctl status systemd-resolved
```

```bash
systemctl stop systemd-resolved
systemctl disable systemd-resolved.service
sudo systemctl mask systemd-resolved
```

## SYSTEMD-Resolved에 의해 생성된 편집 권한 없이 파일 제거:

```bash
rm -f /etc/resolv.conf
```

## 편집 권한이 있는 새 파일 만들기:

```bash
vim /etc/resolv.conf
```

```bash
domain educatux.edu
search educatux.edu.
nameserver 192.168.70.253
```

## 자동 편집에 대해 파일 잠금

```bash
sudo chattr +i /etc/resolv.conf
```

## 서비스 재시작

```bash
sudo systemctl restart NetworkManager
```

---

🎯 그게 전부입니다!

👉 문의: zerolies@disroot.org
👉 https://t.me/z3r0l135

