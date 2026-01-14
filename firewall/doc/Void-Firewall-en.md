#  🧩 VOID LINUX TUTORIAL — SECURITY SCHEME IMPLEMENTATION – LABORATORY WORKSHOPS

📌 Firewall com IP Público, Void Linux (glibc), IPTables (legacy), NAT, Port Knocking e Fail2ban

---

## ✅ 1. NETWORK TOPOLOGY

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

View from another angle

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

The firewall is the only host exposed to the Internet.

## ✅ 2. OBJECTIVES AND ASSUMPTIONS

- Deny default policy
- Active IPv4 routing
- Scanner never sees the door
- Firewall as the only point of entry
- No web dashboards published
- SSH protected by Port Knocking
- Brute-force control via Fail2ban
- Controlled NAT for the LAN
- Remote administration via SSH tunnel

## ✅ 3. UPDATE AND INSTALL NECESSARY PACKAGES

Update the system

```bash
sudo xbps-install -Syu
```

Install the packages

```bash
sudo xbps-install -y \
  vim \
  bash-completion \
  iptables \
  iproute2 \
  openssh \
  tcpdump \
  conntrack-tools\
  fail2ban
```

## ✅ 4. SSH CONFIGURATION

```bash
sudo vim /etc/ssh/sshd_config
```

Adjust the pointed lines

```bash
Port 2222
ListenAddress 0.0.0.0

PermitRootLogin yes        # TEMPORÁRIO (remover após hardening)
PasswordAuthentication yes
UsePAM no

SyslogFacility AUTH
LogLevel INFO
```

Fail2ban depends on log, guarantee the lines

```bash
SyslogFacility AUTH
LogLevel INFO
```

Confirm log generation

```bash
sudo tail -f /var/log/auth.log
```

## Service activation

```bash
sudo ln -s /etc/sv/sshd /var/service/
sudo sv start sshd
```

## After full deployment:

- Disable root login

- Use only key authentication

## ✅ 5. FIREWALL NETWORK SETUP

```bash
sudo vim /etc/dhcpcd.conf
```

Content

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

Apply

```bash
sudo sv restart dhcpcd
```

## ✅ 6. PORT KNOCKING – KERNEL SUPPORT

Load the required module

```bash
sudo modprobe xt_recent
```

Validate:

```bash
sudo lsmod | grep xt_recent
```

Expected result

```bash
xt_recent              24576  0
x_tables               65536  1 xt_recent
```

## ✅ 7. FIREWALL IPTABLES

Enable routing between Firewall network cards

```bash
sudo vim /etc/sysctl.conf
```

Content

```bash
net.ipv4.ip_forward=1
```

Apply without reboot:

```bash
sudo sysctl --system
```

Create the firewall script in /usr/local/bin

```bash
sudo vim /usr/local/bin/firewall
```

Content

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
# ANTISCAN
# ============================

iptables -A INPUT -p tcp --tcp-flags ALL NONE -j DROP
iptables -A INPUT -p tcp --tcp-flags ALL ALL -j DROP
iptables -A INPUT -p tcp --tcp-flags SYN,FIN SYN,FIN -j DROP

exit 0
```

Apply permission and execute

```bash
sudo chmod +x /usr/local/bin/firewall
sudo bash /usr/local/bin/firewall
```

## ✅ 8. FIREWALL PERSISTENCE IN RUNIT

Create the directory

```bash
sudo mkdir -p /etc/sv/firewall
```

Create the file

```bash
sudo vim /etc/sv/firewall/run
```

Content

```bash
#!/bin/sh
exec /usr/local/bin/firewall
```

Activate, run and validate status

```bash
chmod +x /etc/sv/firewall/run
ln -s /etc/sv/firewall /var/service/
sv status firewall
```

## ✅ 9. TESTING AND VALIDATION (HOT) OF PORT KNOCKING

Monitor knock on a terminal WITHOUT FIREWALL

```bash
sudo tcpdump -ni eth0 tcp port 12345
```

Send the knock BY NOTEBOOK via EXTERNAL access

```bash
sudo nc -z 39.236.83.109 12345
```

✔ SYN arrives
✔ It's DROPed
✔ Stay registered
✔ status is visible

Expected result in tcpdump

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

Important technical note

- The RST is sent via the TCP stack
- The package is registered by xt_recent
- Port does not respond as a service
- There is no banner or fingerprint

Validate IP registration

```bash
cat /proc/net/xt_recent/SSH_KNOCK
```

Expected result

```bash
src=99.336.74.209 ttl: 61 last_seen: 4302299386 oldest_pkt: 7 4302292227, 4302293242, 4302294266, 4302295290, 4302296314, 4302297338, 4302299386
```

IF you want to clear all knocks

```bash
echo clear > /proc/net/xt_recent/SSH_KNOCK
```

## ✅ 10. PERFORM EXTERNAL ADMINISTRATIVE ACCESS

Execute the knock

```bash
nc -z 39.236.83.109 12345
```

Within 15 seconds, access

```bash
ssh -p 2222 supertux@39.236.83.109
```

Recommended aliases

```bash
sudo vim .bashrc
```

Content

```bash
alias knock='nc -z 39.236.83.109 12345'
alias officinas='ssh -p 2222 supertux@39.236.83.109'
```

Reread the file for validation

```bash
source .bashrc
```

11. ✅ FAIL2BAN – POST-KNOCK PROTECTION

Create configuration file (Never edit jail.conf)

```bash
sudo vim /etc/fail2ban/jail.local
```

Content:

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

Runit activation

```bash
sudo ln -s /etc/sv/fail2ban /var/service/
sudo sv start fail2ban
sudo sv status fail2ban
```

12. ✅ FAIL2BAN TEST (ATTENTION you lock yourself out during external access)

Execute o knock

```bash
nc -z 39.236.83.109 12345
```

Try SSH with the wrong password 3 times

Check the ban

```bash
sudo fail2ban-client status sshd
```

Unban manually:

```bash
sudo fail2ban-client set sshd unbanip X.X.X.X
```

13. 🎉  CHECKLIST FINAL

- Invisible SSH without knock
- Single-use Knock
- Short access window
- Fail2ban active post-auth
- Ban ignore knock
- Functional NAT
- Persistent firewall
- Proxmox only accessible via tunnel

---

🎯 THAT'S ALL FOLKS!
👉 https://t.me/z3r0l135
👉 https://t.me/vcatafesta
















































































