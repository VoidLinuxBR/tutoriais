# 🧩 VOID LINUX 튜토리얼 - 방화벽 구현 - EDUCATUX LABORATORY

😀 공용 IP를 갖춘 방화벽, Void Linux(glibc), IPTables(레거시), NAT, 포트 노킹, Fail2ban, DHCP 서버 및 재귀 DNS

---

## ✅ 1. 네트워크 토폴로지

```bash
Internet
   |
[Roteador do ISP]
LAN: 192.168.0.1/24
DMZ → 192.168.0.254
   |
[Firewall VM - Void Linux]
eth0 (WAN): 192.168.0.254/24
eth1 (LAN): 192.168.70.254/24
   |
[Rede interna / Switch]
```

다른 각도에서 보기

```bash
Internet
  |
[ Knock correto ]
  |
iptables (xt_recent libera SSH por X segundos)
  |
sshd (porta 2222)
  |
Fail2ban (analisa auth.log)
  |
iptables (ban definitivo do IP)
```

방화벽은 인터넷에 노출되는 유일한 호스트입니다.

## ✅ 2. 목표 및 가정

- 기본 정책 거부
- 활성 IPv4 라우팅
- 스캐너가 문을 보지 못함
- 유일한 진입점인 방화벽
- 게시된 웹 대시보드가 없습니다.
- 포트 노킹으로 보호되는 SSH
- Fail2ban을 통한 무차별 대입 제어
- LAN용 제어 NAT
- SSH 터널을 통한 원격 관리

## ✅ 3. 필요한 패키지 업데이트 및 설치

시스템 업데이트

```bash
sudo xbps-install -Syu
```

패키지 설치

```bash
sudo xbps-install -y \
  vim \
  bash-completion \
  iptables \
  iproute2 \
  openssh \
  tcpdump \
  conntrack-tools \
  fail2ban
```

## ✅ 4. SSH 구성

```bash
sudo vim /etc/ssh/sshd_config
```

뾰족한 선을 조정하세요

```bash
Port 2222
ListenAddress 0.0.0.0

PermitRootLogin yes        # TEMPORÁRIO (remover após hardening)
PasswordAuthentication yes
UsePAM no

SyslogFacility AUTH
LogLevel INFO
```

Fail2ban은 로그에 따라 다르며 라인을 보장합니다.

```bash
SyslogFacility AUTH
LogLevel INFO
```

로그 생성 확인

```bash
sudo tail -f /var/log/auth.log
```

## 서비스 활성화

```bash
sudo ln -s /etc/sv/sshd /var/service/
sudo sv start sshd
```

## 전체 배포 후:

- 루트 로그인 비활성화

- 키 인증만 사용

## ✅ 5. 방화벽 네트워크 설정

```bash
sudo vim /etc/dhcpcd.conf
```

콘텐츠

```bash
# CONFIGURAÇÃO DE REDE DO FIREWALL

# WAN – 192.168.0.0/24
interface eth0
static ip_address=192.168.0.254/24
static routers=192.168.0.1
static domain_name_servers=192.168.0.1 8.8.8.8

# LAN – 192.168.70.0/24
interface eth1
static ip_address=192.168.70.254/24
nogateway
```

적용하다

```bash
sudo sv restart dhcpcd
```

## ✅ 6. 포트 노킹 – 커널 지원

필요한 모듈을 로드하세요.

```bash
sudo modprobe xt_recent
```

검증:

```bash
sudo lsmod | grep xt_recent
```

예상되는 결과

```bash
xt_recent              24576  0
x_tables               65536  1 xt_recent
```

## ✅ 7. 방화벽 IP테이블

방화벽 네트워크 카드 간 라우팅 활성화

```bash
sudo vim /etc/sysctl.conf
```

콘텐츠

```bash
net.ipv4.ip_forward=1
```

재부팅 없이 적용:

```bash
sudo sysctl --system
```

/usr/local/bin에 방화벽 스크립트를 만듭니다.

```bash
sudo vim /usr/local/bin/firewall
```

콘텐츠

```bash
#!/bin/sh
# Firewall – Void Linux
# NAT + Port Knocking + Compatível com Fail2ban

WAN="eth0"
LAN="eth1"

LAN_NET="192.168.70.0/24"

SSH_PORT="2222"
KNOCK_PORT="12345"
KNOCK_NAME="SSH_KNOCK"
KNOCK_TIMEOUT="15"

# LIMPEZA
iptables -F
iptables -X
iptables -t nat -F
iptables -t mangle -F

# POLÍTICAS
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT

# LOOPBACK
iptables -A INPUT -i lo -j ACCEPT

# CONEXÕES ESTABELECIDAS
iptables -A INPUT   -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A FORWARD -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# ============================
# PORT KNOCKING – SSH (WAN)
# ============================

# Knock: registra IP
iptables -A INPUT -i $WAN -p tcp --dport $KNOCK_PORT \
  -m conntrack --ctstate NEW \
  -m recent --set --name $KNOCK_NAME --rsource \
  -j DROP

# SSH liberado UMA VEZ e remove o knock
iptables -A INPUT -i $WAN -p tcp --dport $SSH_PORT \
  -m conntrack --ctstate NEW \
  -m recent --rcheck --seconds $KNOCK_TIMEOUT \
  --name $KNOCK_NAME --rsource \
  -m recent --remove --name $KNOCK_NAME --rsource \
  -j ACCEPT

# ============================
# SSH LOCAL (FAILSAFE – LAN)
# ============================

iptables -A INPUT -i $LAN -s $LAN_NET -p tcp --dport $SSH_PORT -j ACCEPT

# ============================
# FORWARD E NAT DA LAN
# ============================

iptables -A FORWARD -i $LAN -o $WAN -s $LAN_NET \
  -m conntrack --ctstate NEW,ESTABLISHED,RELATED -j ACCEPT

iptables -t nat -A POSTROUTING -s $LAN_NET -o $WAN -j MASQUERADE

# ============================
# ICMP CONTROLADO
# ============================

iptables -A INPUT -p icmp --icmp-type echo-request \
  -m limit --limit 1/s -j ACCEPT

# ============================
# DHCP NA LAN
# ============================
iptables -A INPUT  -i $LAN -p udp --sport 67:68 --dport 67:68 -j ACCEPT
iptables -A OUTPUT -o $LAN -p udp --sport 67:68 --dport 67:68 -j ACCEPT

# ============================
# ANTISCAN
# ============================

iptables -A INPUT -p tcp --tcp-flags ALL NONE -j DROP
iptables -A INPUT -p tcp --tcp-flags ALL ALL -j DROP
iptables -A INPUT -p tcp --tcp-flags SYN,FIN SYN,FIN -j DROP

exit 0
```

권한 적용 및 실행

```bash
sudo chmod +x /usr/local/bin/firewall
sudo bash /usr/local/bin/firewall
```

## ✅ 8. RUNIT의 방화벽 지속성

디렉터리 만들기

```bash
sudo mkdir -p /etc/sv/firewall
```

파일 만들기

```bash
sudo vim /etc/sv/firewall/run
```

콘텐츠

```bash
#!/bin/sh
exec /usr/local/bin/firewall
```

활성화, 실행 및 상태 확인

```bash
sudo chmod +x /etc/sv/firewall/run
sudo ln -s /etc/sv/firewall /var/service/
sudo sv status firewall
```

## ✅ 9. 포트 노킹 테스트 및 검증(핫)

방화벽 없이 터미널 노크를 모니터링하세요.

```bash
sudo tcpdump -ni eth0 tcp port 12345
```

외부 액세스를 통해 노트북으로 노크 보내기

```bash
sudo nc -z 39.236.83.109 12345
```

✔ SYN 도착
✔ 삭제되었습니다.
✔ 등록 상태 유지
✔ 상태가 표시됩니다

tcpdump의 예상 결과

```bash
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes

14:21:14.986974 IP 99.336.74.209.58634 > 192.168.0.254.12345: Flags [S], seq 4021117238, win 64240, options [mss 1436,sackOK,TS val 2035986741 ecr 0,nop,wscale 7], length 0
14:21:14.987007 IP 192.168.0.254.12345 > 99.336.74.209.58634: Flags [R.], seq 0, ack 4021117239, win 0, length 0
^C
2 packets captured
3 packets received by filter
0 packets dropped by kernel
```

중요한 기술 노트

- RST는 TCP 스택을 통해 전송됩니다.
- 패키지가 xt_recent에 의해 등록되었습니다.
- 포트가 서비스로 응답하지 않습니다.
- 배너나 지문이 없습니다.

IP 등록 확인

```bash
sudo cat /proc/net/xt_recent/SSH_KNOCK
```

예상되는 결과

```bash
src=99.336.74.209 ttl: 61 last_seen: 4302299386 oldest_pkt: 7 4302292227, 4302293242, 4302294266, 4302295290, 4302296314, 4302297338, 4302299386
```

모든 노크를 없애고 싶다면

```bash
sudo echo clear > /proc/net/xt_recent/SSH_KNOCK
```

## ✅ 10. 외부 관리 액세스 수행

노크를 실행

```bash
nc -z 39.236.83.109 12345
```

15초 이내에 접속

```bash
ssh -p 2222 supertux@39.236.83.109
```

권장 별칭

```bash
vim ~/.bashrc
```

콘텐츠

```bash
alias knock='nc -z 39.236.83.109 12345'
alias firewall='ssh -p 2222 supertux@39.236.83.109'
```

유효성 검사를 위해 파일을 다시 읽습니다.

```bash
source ~/.bashrc
```

11. ✅ FAIL2BAN – 노크 후 보호

Fail2ban을 준수하기 위한 로그 조정

```bash
sudo xbps-install -y socklog-void
sudo ln -s /etc/sv/socklog-unix /var/service/
sudo ln -s /etc/sv/nanoklogd /var/service/
sudo touch /var/log/auth.log
```

구성 파일 생성(jail.conf를 편집하지 않음)

```bash
sudo vim /etc/fail2ban/jail.local
```

콘텐츠:

```bash
[DEFAULT]
bantime  = 24h
findtime = 10m
maxretry = 3
backend  = auto
banaction = iptables-multiport

[sshd]
enabled  = true
port     = 2222
logpath  = /var/log/auth.log
maxretry = 3
findtime = 5m
bantime  = 24h
```

Runit 활성화

```bash
sudo ln -s /etc/sv/fail2ban /var/service/
sudo sv start fail2ban
sudo sv status fail2ban
```

## 12. ✅ FAIL2BAN 테스트(주의하세요. 스스로 잠길 수 있습니다!)

실행 o 노크

```bash
nc -z 39.236.83.109 12345
```

잘못된 비밀번호로 SSH를 3번 시도해보세요

금지사항을 확인하세요

```bash
sudo fail2ban-client status sshd
```

수동으로 차단 해제:

```bash
sudo fail2ban-client set sshd unbanip X.X.X.X
```

## ⚠️ 주의: 재귀 DNS 및 DHCP 서버를 다루는 다음 섹션 13 및 14는 Samba4를 PDC로 업그레이드한 후 폐기해야 합니다!!

## 13. ✅ 내부 네트워크에 서비스를 제공하기 위해 임시 재귀 DNS 배포

```bash
sudo xbps-install -y unbound
```

최소 구성

```bash
sudo vim /etc/unbound/unbound.conf
```

콘텐츠

```bash
server:
  interface: 127.0.0.1
  interface: 192.168.70.254
  access-control: 192.168.70.0/24 allow
  do-ip4: yes
  do-udp: yes
  do-tcp: yes
  hide-identity: yes
  hide-version: yes
  qname-minimisation: yes
```

서비스 활성화(runit):

```bash
ln -s /etc/sv/unbound /var/service/
sv start unbound
```

## 14. ✅ 내부 네트워크 서비스를 위한 임시 DHCP 서버 구현

패키지 설치

```bash
sudo xbps-install -y dhcp
```

이 패키지는 다음을 설치합니다:

- dhcpd(서버)
- Runit 서비스 구조:
/etc/sv/dhcpd4
/etc/sv/dhcpd6

파일 편집 및 내부 네트워크 설정 구성

```bash
sudo vim /etc/dhcpd.conf
```

콘텐츠

```bash
authoritative;

default-lease-time 600;
max-lease-time 7200;

option domain-name "educatux.edu";
option domain-name-servers 192.168.70.254;

subnet 192.168.70.0 netmask 255.255.255.0 {

  range 192.168.70.100 192.168.70.200;

  option routers 192.168.70.254;
  option subnet-mask 255.255.255.0;
  option broadcast-address 192.168.70.255;

  option domain-name-servers 192.168.70.254;
}
```

임대 파일을 만듭니다.

```bash
sudo mkdir -p /var/lib/dhcp
sudo touch /var/lib/dhcp/dhcpd.leases
```

Runit 서비스 생성

```bash
sudo vim /etc/sv/dhcpd4/conf
```

콘텐츠

```bash
OPTS="-4 -q -cf /etc/dhcpd.conf eth1"
```

설명:

- -4 → IPv4
- -q → 자동 모드
- -cf → 올바른 dhcpd.conf 경로
- eth1 → 인터페이스 LAN

runit에서 서비스를 활성화합니다:

```bash
sudo ln -s /etc/sv/dhcpd4 /var/service/
```

시작/다시 시작:

```bash
sudo sv restart dhcpd4
```

상태 확인:

```bash
sudo sv status dhcpd4
```

예상 결과:

```bash
run: dhcpd4: (pid 17652) 831s; run: log: (pid 15544) 1213s
```

포트 67 수신 대기를 확인하세요.

```bash
UNCONN 0      0            0.0.0.0:67        0.0.0.0:*    users:(("dhcpd",pid=17652,fd=6))  
```

실시간으로 DHCP 모니터링

```bash
sudo tcpdump -ni eth1 port 67 or port 68
```

예상되는 결과

```bash
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth1, link-type EN10MB (Ethernet), snapshot length 262144 bytes
```

직접 디버깅용(runit 없이)

```bash
sudo dhcpd -4 -d -cf /etc/dhcpd.conf eth1
```

이 표시되어야합니다

- DHCP 검색
- DHCP 제안
- DHCP요청
- DHCPACK

중요한 파일

- /etc/dhcpd.conf → 기본 구성
- /var/lib/dhcp/dhcpd.leases → 임대
- /etc/sv/dhcpd4/run → 스크립트 runit
- /etc/sv/dhcpd4/conf → 서비스 매개변수
- /var/service/dhcpd4 → 서비스 활성화

LAN에서 DHCP를 허용하도록 iptables 스크립트를 조정하십시오. 암시적 DROP 규칙 앞에 추가합니다.

```bash
# ============================
# DHCP LAN
# ============================

iptables -A INPUT  -i $LAN -p udp --sport 67:68 --dport 67:68 -j ACCEPT
iptables -A OUTPUT -o $LAN -p udp --sport 67:68 --dport 67:68 -j ACCEPT
```

💡 DHCP는 브로드캐스트를 사용합니다. → 이것이 없으면 클라이언트는 IP를 얻지 못합니다.

방화벽을 다시 적용합니다.

```bash
sudo /usr/local/bin/firewall
```

LAN VM에서 테스트

```bash
dhclient -v
```

방화벽에서 모니터링

```bash
sudo tail -f /var/log/messages
```

또는

```bash
sudo tcpdump -ni eth1 port 67 or port 68
```

## 15. 🎉 체크리스트 최종

- 노크 없이 보이지 않는 SSH
- 일회용 노크
- 짧은 액세스 창
- Fail2ban 활성 사후 인증
- 노크 무시 금지
- 기능적 NAT
- 영구 방화벽
- Proxmox는 터널을 통해서만 접근 가능
- 최소 재귀 DNS(PDC가 진입할 때까지)
- DHCP 서버

---

🎯 그게 전부입니다!

👉 https://t.me/z3r0l135
👉 https://t.me/vcatafesta
