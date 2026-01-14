#  🧩 VOID LINUX TUTORIAL – SICHERHEITSSCHEMA-IMPLEMENTIERUNG – LABOR-WORKSHOPS

📌 Firewall mit öffentlicher IP, Void Linux (glibc), IPTables (Legacy), NAT, Port Knocking und Fail2ban

---

## ✅ 1. NETZWERKTOPOLOGIE

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

Blick aus einem anderen Blickwinkel

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

Die Firewall ist der einzige Host, der dem Internet ausgesetzt ist.

## ✅ 2. ZIELE UND ANNAHMEN

- Standardrichtlinie verweigern
- Aktives IPv4-Routing
- Der Scanner sieht die Tür nie
- Firewall als einziger Eintrittspunkt
- Keine Web-Dashboards veröffentlicht
- SSH geschützt durch Port Knocking
- Brute-Force-Kontrolle über Fail2ban
- Kontrolliertes NAT für das LAN
- Fernverwaltung über SSH-Tunnel

## ✅ 3. ERFORDERLICHE PAKETE AKTUALISIEREN UND INSTALLIEREN

Aktualisieren Sie das System

```bash
sudo xbps-install -Syu
```

Installieren Sie die Pakete

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

## ✅ 4. SSH-KONFIGURATION

```bash
sudo vim /etc/ssh/sshd_config
```

Passen Sie die spitzen Linien an

```bash
Port 2222
ListenAddress 0.0.0.0

PermitRootLogin yes        # TEMPORÁRIO (remover após hardening)
PasswordAuthentication yes
UsePAM no

SyslogFacility AUTH
LogLevel INFO
```

Fail2ban hängt vom Protokoll ab, garantiert die Leitungen

```bash
SyslogFacility AUTH
LogLevel INFO
```

Bestätigen Sie die Protokollerstellung

```bash
sudo tail -f /var/log/auth.log
```

## Dienstaktivierung

```bash
sudo ln -s /etc/sv/sshd /var/service/
sudo sv start sshd
```

## Nach vollständiger Bereitstellung:

- Deaktivieren Sie die Root-Anmeldung

- Verwenden Sie nur die Schlüsselauthentifizierung

## ✅ 5. FIREWALL-NETZWERK-EINRICHTUNG

```bash
sudo vim /etc/dhcpcd.conf
```

Inhalt

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

Anwenden

```bash
sudo sv restart dhcpcd
```

## ✅ 6. PORT KNOCKING – KERNEL-UNTERSTÜTZUNG

Laden Sie das erforderliche Modul

```bash
sudo modprobe xt_recent
```

Bestätigen:

```bash
sudo lsmod | grep xt_recent
```

Erwartetes Ergebnis

```bash
xt_recent              24576  0
x_tables               65536  1 xt_recent
```

## ✅ 7. FIREWALL IPTABLES

Aktivieren Sie das Routing zwischen Firewall-Netzwerkkarten

```bash
sudo vim /etc/sysctl.conf
```

Inhalt

```bash
net.ipv4.ip_forward=1
```

Ohne Neustart anwenden:

```bash
sudo sysctl --system
```

Erstellen Sie das Firewall-Skript in /usr/local/bin

```bash
sudo vim /usr/local/bin/firewall
```

Inhalt

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

Berechtigung anwenden und ausführen

```bash
sudo chmod +x /usr/local/bin/firewall
sudo bash /usr/local/bin/firewall
```

## ✅ 8. Firewall-Persistenz in Runit

Erstellen Sie das Verzeichnis

```bash
sudo mkdir -p /etc/sv/firewall
```

Erstellen Sie die Datei

```bash
sudo vim /etc/sv/firewall/run
```

Inhalt

```bash
#!/bin/sh
exec /usr/local/bin/firewall
```

Status aktivieren, ausführen und validieren

```bash
chmod +x /etc/sv/firewall/run
ln -s /etc/sv/firewall /var/service/
sv status firewall
```

## ✅ 9. TESTEN UND VALIDIEREN (HEISS) VON PORT KNOCKING

Überwachen Sie das Klopfen an einem Terminal OHNE FIREWALL

```bash
sudo tcpdump -ni eth0 tcp port 12345
```

Senden Sie den Klopf DURCH NOTIZBUCH über EXTERNEN Zugriff

```bash
sudo nc -z 39.236.83.109 12345
```

✔ SYN kommt
✔ Es ist GEFALLEN
✔ Bleiben Sie registriert
✔ Status ist sichtbar

Erwartetes Ergebnis in tcpdump

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

Wichtiger technischer Hinweis

- Der RST wird über den TCP-Stack gesendet
- Das Paket wird von xt_recent registriert
- Der Port antwortet nicht als Dienst
- Es gibt kein Banner oder Fingerabdruck

Validieren Sie die IP-Registrierung

```bash
cat /proc/net/xt_recent/SSH_KNOCK
```

Erwartetes Ergebnis

```bash
src=99.336.74.209 ttl: 61 last_seen: 4302299386 oldest_pkt: 7 4302292227, 4302293242, 4302294266, 4302295290, 4302296314, 4302297338, 4302299386
```

WENN Sie alle Stöße beseitigen möchten

```bash
echo clear > /proc/net/xt_recent/SSH_KNOCK
```

## ✅ 10. EXTERNEN ADMINISTRATIVEN ZUGRIFF DURCHFÜHREN

Führe den Klopf aus

```bash
nc -z 39.236.83.109 12345
```

Innerhalb von 15 Sekunden Zugriff

```bash
ssh -p 2222 supertux@39.236.83.109
```

Empfohlene Aliase

```bash
sudo vim .bashrc
```

Inhalt

```bash
alias knock='nc -z 39.236.83.109 12345'
alias officinas='ssh -p 2222 supertux@39.236.83.109'
```

Lesen Sie die Datei zur Validierung erneut

```bash
source .bashrc
```

11. ✅ FAIL2BAN – NACHKLOPFSCHUTZ

Konfigurationsdatei erstellen (niemals jail.conf bearbeiten)

```bash
sudo vim /etc/fail2ban/jail.local
```

Inhalt:

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

Runit-Aktivierung

```bash
sudo ln -s /etc/sv/fail2ban /var/service/
sudo sv start fail2ban
sudo sv status fail2ban
```

12. ✅ FAIL2BAN TEST (ACHTUNG Du sperrst Dich bei Fremdzugriff aus)

Ausführen oder klopfen

```bash
nc -z 39.236.83.109 12345
```

Versuchen Sie dreimal SSH mit dem falschen Passwort

Überprüfen Sie das Verbot

```bash
sudo fail2ban-client status sshd
```

Manuell entsperren:

```bash
sudo fail2ban-client set sshd unbanip X.X.X.X
```

13. 🎉 CHECKLISTE FINAL

- Unsichtbares SSH ohne Klopfen
- Einweg-Knock
- Kurzes Zugriffsfenster
- Fail2ban aktiv nach der Authentifizierung
- Verbot, Klopfen zu ignorieren
- Funktionelles NAT
- Permanente Firewall
- Proxmox nur über Tunnel erreichbar

---

🎯 DAS IST ALLES, LEUTE!
👉 https://t.me/z3r0l135
👉 https://t.me/vcatafesta
















































































