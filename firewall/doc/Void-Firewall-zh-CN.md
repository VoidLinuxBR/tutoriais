# 🧩 VOID LINUX 教程 - 防火墙实现 - 实验室研讨会

📌 具有公共 IP、Void Linux (glibc)、IPTables（旧版）、NAT、端口敲门、Fail2ban、DHCP 服务器和递归 DNS 的防火墙

---

## ✅ 1. 网络拓扑

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

从另一个角度看

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

防火墙是唯一暴露于互联网的主机。

## ✅ 2. 目标和假设

- 拒绝默认策略
- 主动 IPv4 路由
- 扫描仪永远看不到门
- 防火墙作为唯一的入口点
- 没有发布网络仪表板
- 受端口敲门保护的 SSH
- 通过 Fail2ban 进行暴力控制
- LAN 的受控 NAT
- 通过 SSH 隧道进行远程管理

## ✅ 3.更新并安装必要的软件包

更新系统

```bash
sudo xbps-install -Syu
```

安装软件包

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

调整尖线

```bash
Port 2222
ListenAddress 0.0.0.0

PermitRootLogin yes        # TEMPORÁRIO (remover após hardening)
PasswordAuthentication yes
UsePAM no

SyslogFacility AUTH
LogLevel INFO
```

Fail2ban依赖日志，保证线路

```bash
SyslogFacility AUTH
LogLevel INFO
```

确认日志生成

```bash
sudo tail -f /var/log/auth.log
```

## 服务激活

```bash
sudo ln -s /etc/sv/sshd /var/service/
sudo sv start sshd
```

## 全面部署后：

- 禁用 root 登录

- 仅使用密钥身份验证

## ✅ 5. 防火墙网络设置

```bash
sudo vim /etc/dhcpcd.conf
```

内容

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

申请

```bash
sudo sv restart dhcpcd
```

## ✅ 6. 端口敲击 – 内核支持

加载所需模块

```bash
sudo modprobe xt_recent
```

证实：

```bash
sudo lsmod | grep xt_recent
```

预期结果

```bash
xt_recent              24576  0
x_tables               65536  1 xt_recent
```

## ✅ 7. 防火墙 IPTables

启用防火墙网卡之间的路由

```bash
sudo vim /etc/sysctl.conf
```

内容

```bash
net.ipv4.ip_forward=1
```

无需重启即可应用：

```bash
sudo sysctl --system
```

在 /usr/local/bin 中创建防火墙脚本

```bash
sudo vim /usr/local/bin/firewall
```

内容

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

申请权限并执行

```bash
sudo chmod +x /usr/local/bin/firewall
sudo bash /usr/local/bin/firewall
```

## ✅ 8. RUNIT 中的防火墙持久性

创建目录

```bash
sudo mkdir -p /etc/sv/firewall
```

创建文件

```bash
sudo vim /etc/sv/firewall/run
```

内容

```bash
#!/bin/sh
exec /usr/local/bin/firewall
```

激活、运行和验证状态

```bash
sudo chmod +x /etc/sv/firewall/run
sudo ln -s /etc/sv/firewall /var/service/
sudo sv status firewall
```

## ✅ 9. 端口敲击的测试和验证（热）

无需防火墙即可监控终端敲击

```bash
sudo tcpdump -ni eth0 tcp port 12345
```

通过外部访问通过笔记本发送敲门声

```bash
sudo nc -z 39.236.83.109 12345
```

✔ SYN 到达
✔ 已删除
✔ 保持注册状态
✔ 状态可见

tcpdump 中的预期结果

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

重要技术说明

- RST 通过 TCP 堆栈发送
- 该包由 xt_recent 注册
- 端口不作为服务响应
- 没有横幅或指纹

验证IP注册

```bash
sudo cat /proc/net/xt_recent/SSH_KNOCK
```

预期结果

```bash
src=99.336.74.209 ttl: 61 last_seen: 4302299386 oldest_pkt: 7 4302292227, 4302293242, 4302294266, 4302295290, 4302296314, 4302297338, 4302299386
```

如果你想清除所有的敲门声

```bash
sudo echo clear > /proc/net/xt_recent/SSH_KNOCK
```

## ✅ 10. 执行外部管理访问

执行敲击

```bash
nc -z 39.236.83.109 12345
```

15秒内，访问

```bash
ssh -p 2222 supertux@39.236.83.109
```

推荐别名

```bash
vim ~/.bashrc
```

内容

```bash
alias knock='nc -z 39.236.83.109 12345'
alias officinas='ssh -p 2222 supertux@39.236.83.109'
```

重新读取文件进行验证

```bash
source ~/.bashrc
```

11. ✅ FAIL2BAN – 爆震后保护

日志调整以符合fail2ban

```bash
sudo xbps-install -y socklog-void
sudo ln -s /etc/sv/socklog-unix /var/service/
sudo ln -s /etc/sv/nanoklogd /var/service/
sudo touch /var/log/auth.log
```

创建配置文件（切勿编辑jail.conf）

```bash
sudo vim /etc/fail2ban/jail.local
```

内容：

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

运行单元激活

```bash
sudo ln -s /etc/sv/fail2ban /var/service/
sudo sv start fail2ban
sudo sv status fail2ban
```

## 12. ✅ FAIL2BAN 测试（注意，你把自己锁在外面了！）

执行敲门

```bash
nc -z 39.236.83.109 12345
```

使用错误密码尝试 SSH 3 次

检查禁令

```bash
sudo fail2ban-client status sshd
```

手动解禁：

```bash
sudo fail2ban-client set sshd unbanip X.X.X.X
```

## ⚠️ 注意：以下第 13 和 14 节涉及递归 DNS 和 DHCP 服务器，在将 SAMBA4 升级为 PDC 后必须丢弃！

## 13. ✅ 部署临时递归 DNS 来为内部网络提供服务

```bash
sudo xbps-install -y unbound
```

最低配置

```bash
sudo vim /etc/unbound/unbound.conf
```

内容

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

激活服务（运行单元）：

```bash
ln -s /etc/sv/unbound /var/service/
sv start unbound
```

## 14. ✅ 实施临时 DHCP 服务器来为内部网络提供服务

包安装

```bash
sudo xbps-install -y dhcp
```

该软件包安装：

- dhcpd（服务器）
- Runit服务结构：
/etc/sv/dhcpd4
/etc/sv/dhcpd6

编辑文件并配置内部网络的设置

```bash
sudo vim /etc/dhcpd.conf
```

内容

```bash
authoritative;

default-lease-time 600;
max-lease-time 7200;

option domain-name "officinas.edu";
option domain-name-servers 192.168.70.254;

subnet 192.168.70.0 netmask 255.255.255.0 {

  range 192.168.70.100 192.168.70.200;

  option routers 192.168.70.254;
  option subnet-mask 255.255.255.0;
  option broadcast-address 192.168.70.255;

  option domain-name-servers 192.168.70.254;
}
```

创建租赁文件：

```bash
sudo mkdir -p /var/lib/dhcp
sudo touch /var/lib/dhcp/dhcpd.leases
```

Runit服务创建

```bash
sudo vim /etc/sv/dhcpd4/conf
```

内容

```bash
OPTS="-4 -q -cf /etc/dhcpd.conf eth1"
```

解释：

- -4 → IPv4
- -q → 静默模式
- -cf → 正确的 dhcpd.conf 路径
- eth1 → 接口 LAN

在runit中激活服务：

```bash
sudo ln -s /etc/sv/dhcpd4 /var/service/
```

启动/重新启动：

```bash
sudo sv restart dhcpd4
```

检查状态：

```bash
sudo sv status dhcpd4
```

预期结果：

```bash
run: dhcpd4: (pid 17652) 831s; run: log: (pid 15544) 1213s
```

检查67端口监听

```bash
UNCONN 0      0            0.0.0.0:67        0.0.0.0:*    users:(("dhcpd",pid=17652,fd=6))  
```

实时监控 DHCP

```bash
sudo tcpdump -ni eth1 port 67 or port 68
```

预期结果

```bash
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth1, link-type EN10MB (Ethernet), snapshot length 262144 bytes
```

用于直接调试（无需 runit）

```bash
sudo dhcpd -4 -d -cf /etc/dhcpd.conf eth1
```

这应该显示

- DHCP发现
- DHCP优惠
- DHCP请求
- DHCP确认

重要文件

- /etc/dhcpd.conf → 主要配置
- /var/lib/dhcp/dhcpd.leases → 租约
- /etc/sv/dhcpd4/run → 脚本 runit
- /etc/sv/dhcpd4/conf → 服务参数
- /var/service/dhcpd4 → 服务处于活动状态

调整 iptables 脚本以允许 LAN 上的 DHCP。在隐式 DROP 规则之前添加：

```bash
# ============================
# DHCP LAN
# ============================

iptables -A INPUT  -i $LAN -p udp --sport 67:68 --dport 67:68 -j ACCEPT
iptables -A OUTPUT -o $LAN -p udp --sport 67:68 --dport 67:68 -j ACCEPT
```

💡 DHCP 使用广播 → 如果没有广播，客户端将无法获得 IP。

重新应用防火墙：

```bash
sudo /usr/local/bin/firewall
```

在 LAN VM 上测试

```bash
dhclient -v
```

在防火墙中，监控

```bash
sudo tail -f /var/log/messages
```

或者

```bash
sudo tcpdump -ni eth1 port 67 or port 68
```

## 15. 🎉 最终检查清单

- 隐形SSH无需敲门
- 一次性敲击器
- 访问窗口短
- Fail2ban 主动身份验证后
- 禁止无视敲门
- 功能性NAT
- 持久防火墙
- Proxmox 只能通过隧道访问
- 最小递归 DNS（直到 PDC 进入）
- DHCP服务器

---

🎯 这就是大家！

👉 https://t.me/z3r0l135
👉 https://t.me/vcatafesta
