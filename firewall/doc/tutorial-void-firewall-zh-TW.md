# 🧩 VOID LINUX 教程 - 防火牆實現 - EDUCATUX LABORATORY

📌 具有公共 IP、Void Linux (glibc)、IPTables（舊版）、NAT、端口敲門、Fail2ban、DHCP 服務器和遞歸 DNS 的防火牆

---

## ✅ 1. 網絡拓撲

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

從另一個角度看

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

防火牆是唯一暴露於互聯網的主機。

## ✅ 2. 目標和假設

- 拒絕默認策略
- 主動 IPv4 路由
- 掃描儀永遠看不到門
- 防火牆作為唯一的入口點
- 沒有發佈網絡儀表板
- 受端口敲門保護的 SSH
- 通過 Fail2ban 進行暴力控制
- LAN 的受控 NAT
- 通過 SSH 隧道進行遠程管理

## ✅ 3.更新並安裝必要的軟件包

更新系統

```bash
sudo xbps-install -Syu
```

安裝軟件包

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

## ✅ 4.SSH 配置

```bash
sudo vim /etc/ssh/sshd_config
```

調整尖線

```bash
Port 2222
ListenAddress 0.0.0.0

PermitRootLogin yes        # TEMPORÁRIO (remover após hardening)
PasswordAuthentication yes
UsePAM no

SyslogFacility AUTH
LogLevel INFO
```

Fail2ban依賴日誌，保證線路

```bash
SyslogFacility AUTH
LogLevel INFO
```

確認日誌生成

```bash
sudo tail -f /var/log/auth.log
```

## 服務激活

```bash
sudo ln -s /etc/sv/sshd /var/service/
sudo sv start sshd
```

## 全面部署後：

- 禁用 root 登錄

- 僅使用密鑰身份驗證

## ✅ 5. 防火牆網絡設置

```bash
sudo vim /etc/dhcpcd.conf
```

內容

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

申請

```bash
sudo sv restart dhcpcd
```

## ✅ 6. 端口敲擊 – 內核支持

加載所需模塊

```bash
sudo modprobe xt_recent
```

證實：

```bash
sudo lsmod | grep xt_recent
```

預期結果

```bash
xt_recent              24576  0
x_tables               65536  1 xt_recent
```

## ✅ 7. 防火牆 IPTables

啟用防火牆網卡之間的路由

```bash
sudo vim /etc/sysctl.conf
```

內容

```bash
net.ipv4.ip_forward=1
```

無需重啟即可應用：

```bash
sudo sysctl --system
```

在 /usr/local/bin 中創建防火牆腳本

```bash
sudo vim /usr/local/bin/firewall
```

內容

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

申請權限並執行

```bash
sudo chmod +x /usr/local/bin/firewall
sudo bash /usr/local/bin/firewall
```

## ✅ 8. RUNIT 中的防火牆持久性

創建目錄

```bash
sudo mkdir -p /etc/sv/firewall
```

創建文件

```bash
sudo vim /etc/sv/firewall/run
```

內容

```bash
#!/bin/sh
exec /usr/local/bin/firewall
```

激活、運行和驗證狀態

```bash
sudo chmod +x /etc/sv/firewall/run
sudo ln -s /etc/sv/firewall /var/service/
sudo sv status firewall
```

## ✅ 9. 端口敲擊的測試和驗證（熱）

無需防火牆即可監控終端敲擊

```bash
sudo tcpdump -ni eth0 tcp port 12345
```

通過外部訪問通過筆記本發送敲門聲

```bash
sudo nc -z 39.236.83.109 12345
```

✔ SYN 到達
✔ 已刪除
✔ 保持註冊狀態
✔ 狀態可見

tcpdump 中的預期結果

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

重要技術說明

- RST 通過 TCP 堆棧發送
- 該包由 xt_recent 註冊
- 端口不作為服務響應
- 沒有橫幅或指紋

驗證IP註冊

```bash
sudo cat /proc/net/xt_recent/SSH_KNOCK
```

預期結果

```bash
src=99.336.74.209 ttl: 61 last_seen: 4302299386 oldest_pkt: 7 4302292227, 4302293242, 4302294266, 4302295290, 4302296314, 4302297338, 4302299386
```

如果你想清除所有的敲門聲

```bash
sudo echo clear > /proc/net/xt_recent/SSH_KNOCK
```

## ✅ 10. 執行外部管理訪問

執行敲擊

```bash
nc -z 39.236.83.109 12345
```

15秒內，訪問

```bash
ssh -p 2222 supertux@39.236.83.109
```

推薦別名

```bash
vim ~/.bashrc
```

內容

```bash
alias knock='nc -z 39.236.83.109 12345'
alias firewall='ssh -p 2222 supertux@39.236.83.109'
```

重新讀取文件進行驗證

```bash
source ~/.bashrc
```

11. ✅ FAIL2BAN – 爆震後保護

日誌調整以符合fail2ban

```bash
sudo xbps-install -y socklog-void
sudo ln -s /etc/sv/socklog-unix /var/service/
sudo ln -s /etc/sv/nanoklogd /var/service/
sudo touch /var/log/auth.log
```

創建配置文件（切勿編輯jail.conf）

```bash
sudo vim /etc/fail2ban/jail.local
```

內容：

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

運行單元激活

```bash
sudo ln -s /etc/sv/fail2ban /var/service/
sudo sv start fail2ban
sudo sv status fail2ban
```

## 12. ✅ FAIL2BAN 測試（注意，你把自己鎖在外面了！）

執行敲門

```bash
nc -z 39.236.83.109 12345
```

使用錯誤密碼嘗試 SSH 3 次

檢查禁令

```bash
sudo fail2ban-client status sshd
```

手動解禁：

```bash
sudo fail2ban-client set sshd unbanip X.X.X.X
```

## ⚠️ 注意：以下第 13 和 14 節涉及遞歸 DNS 和 DHCP 服務器，在將 SAMBA4 升級為 PDC 後必須丟棄！

## 13. ✅ 部署臨時遞歸 DNS 來為內部網絡提供服務

```bash
sudo xbps-install -y unbound
```

最低配置

```bash
sudo vim /etc/unbound/unbound.conf
```

內容

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

激活服務（運行單元）：

```bash
ln -s /etc/sv/unbound /var/service/
sv start unbound
```

## 14. ✅ 實施臨時 DHCP 服務器來為內部網絡提供服務

包安裝

```bash
sudo xbps-install -y dhcp
```

該軟件包安裝：

- dhcpd（服務器）
- Runit服務結構：
/etc/sv/dhcpd4
/etc/sv/dhcpd6

編輯文件並配置內部網絡的設置

```bash
sudo vim /etc/dhcpd.conf
```

內容

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

創建租賃文件：

```bash
sudo mkdir -p /var/lib/dhcp
sudo touch /var/lib/dhcp/dhcpd.leases
```

Runit服務創建

```bash
sudo vim /etc/sv/dhcpd4/conf
```

內容

```bash
OPTS="-4 -q -cf /etc/dhcpd.conf eth1"
```

解釋：

- -4 → IPv4
- -q → 靜默模式
- -cf → 正確的 dhcpd.conf 路徑
- eth1 → 接口 LAN

在runit中激活服務：

```bash
sudo ln -s /etc/sv/dhcpd4 /var/service/
```

啟動/重新啟動：

```bash
sudo sv restart dhcpd4
```

檢查狀態：

```bash
sudo sv status dhcpd4
```

預期結果：

```bash
run: dhcpd4: (pid 17652) 831s; run: log: (pid 15544) 1213s
```

檢查67端口監聽

```bash
UNCONN 0      0            0.0.0.0:67        0.0.0.0:*    users:(("dhcpd",pid=17652,fd=6))  
```

實時監控 DHCP

```bash
sudo tcpdump -ni eth1 port 67 or port 68
```

預期結果

```bash
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth1, link-type EN10MB (Ethernet), snapshot length 262144 bytes
```

用於直接調試（無需 runit）

```bash
sudo dhcpd -4 -d -cf /etc/dhcpd.conf eth1
```

這應該顯示

- DHCP發現
- DHCP優惠
- DHCP請求
- DHCP確認

重要文件

- /etc/dhcpd.conf → 主要配置
- /var/lib/dhcp/dhcpd.leases → 租約
- /etc/sv/dhcpd4/run → 腳本 runit
- /etc/sv/dhcpd4/conf → 服務參數
- /var/service/dhcpd4 → 服務處於活動狀態

調整 iptables 腳本以允許 LAN 上的 DHCP。在隱式 DROP 規則之前添加：

```bash
# ============================
# DHCP LAN
# ============================

iptables -A INPUT  -i $LAN -p udp --sport 67:68 --dport 67:68 -j ACCEPT
iptables -A OUTPUT -o $LAN -p udp --sport 67:68 --dport 67:68 -j ACCEPT
```

💡 DHCP 使用廣播 → 如果沒有廣播，客戶端將無法獲得 IP。

重新應用防火牆：

```bash
sudo /usr/local/bin/firewall
```

在 LAN VM 上測試

```bash
dhclient -v
```

在防火牆中，監控

```bash
sudo tail -f /var/log/messages
```

或者

```bash
sudo tcpdump -ni eth1 port 67 or port 68
```

## 15. 🎉 最終檢查清單

- 隱形SSH無需敲門
- 一次性敲擊器
- 訪問窗口短
- Fail2ban 主動身份驗證後
- 禁止無視敲門
- 功能性NAT
- 持久防火牆
- Proxmox 只能通過隧道訪問
- 最小遞歸 DNS（直到 PDC 進入）
- DHCP服務器

---

🎯 這就是大家！

👉 https://t.me/z3r0l135
👉 https://t.me/vcatafesta
