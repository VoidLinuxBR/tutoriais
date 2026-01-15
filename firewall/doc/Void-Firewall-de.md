# 🧩 VOID LINUX TUTORIAL – FIREWALL-IMPLEMENTIERUNG – LABOR-WORKSHOPS

📌 Firewall mit öffentlicher IP, Void Linux (glibc), IPTables (Legacy), NAT, Port Knocking, Fail2ban, DHCP-Server und rekursivem DNS

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
  conntrack-tools \
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
sudo chmod +x /etc/sv/firewall/run
sudo ln -s /etc/sv/firewall /var/service/
sudo sv status firewall
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
sudo cat /proc/net/xt_recent/SSH_KNOCK
```

Erwartetes Ergebnis

```bash
src=99.336.74.209 ttl: 61 last_seen: 4302299386 oldest_pkt: 7 4302292227, 4302293242, 4302294266, 4302295290, 4302296314, 4302297338, 4302299386
```

WENN Sie alle Stöße beseitigen möchten

```bash
sudo echo clear > /proc/net/xt_recent/SSH_KNOCK
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
vim ~/.bashrc
```

Inhalt

```bash
alias knock='nc -z 39.236.83.109 12345'
alias officinas='ssh -p 2222 supertux@39.236.83.109'
```

Lesen Sie die Datei zur Validierung erneut

```bash
source ~/.bashrc
```

11. ✅ FAIL2BAN – NACHKLOPFSCHUTZ

Protokollanpassungen zur Einhaltung von fail2ban

```bash
sudo xbps-install -y socklog-void
sudo ln -s /etc/sv/socklog-unix /var/service/
sudo ln -s /etc/sv/nanoklogd /var/service/
sudo touch /var/log/auth.log
```

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

## 12. ✅ FAIL2BAN-TEST (ACHTUNG, SIE SPERREN SICH AUS!)

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

## ⚠️ ACHTUNG: DIE FOLGENDEN ABSCHNITTE 13 und 14, DIE SICH MIT REKURSIVEM DNS UND DHCP-SERVER BEHANDELN, MÜSSEN NACH DEM UPGRADE VON SAMBA4 ALS PDC ENTFERNT WERDEN!!

## 13. ✅ BEREITSTELLEN EINES TEMPORÄREN REKURSIVEN DNS, UM DAS INTERNE NETZWERK ZU BEDIENEN

```bash
sudo xbps-install -y unbound
```

Mindestkonfiguration

```bash
sudo vim /etc/unbound/unbound.conf
```

Inhalt

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

Dienst aktivieren (runit):

```bash
ln -s /etc/sv/unbound /var/service/
sv start unbound
```

## 14. ✅ IMPLEMENTIERUNG EINES TEMPORÄREN DHCP-SERVERS ZUM BEDIENEN DES INTERNEN NETZWERKS

Paketinstallation

```bash
sudo xbps-install -y dhcp
```

Dieses Paket installiert:

- dhcpd (Server)
- Runit-Dienststruktur:
/etc/sv/dhcpd4
/etc/sv/dhcpd6

Bearbeiten Sie die Datei und konfigurieren Sie die Einstellungen für das interne Netzwerk

```bash
sudo vim /etc/dhcpd.conf
```

Inhalt

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

Erstellen Sie die Mietvertragsdatei:

```bash
sudo mkdir -p /var/lib/dhcp
sudo touch /var/lib/dhcp/dhcpd.leases
```

Erstellung des Runit-Dienstes

```bash
sudo vim /etc/sv/dhcpd4/conf
```

Inhalt

```bash
OPTS="-4 -q -cf /etc/dhcpd.conf eth1"
```

Erläuterung:

- -4 → IPv4
- -q → Silent-Modus
- -cf → korrekten dhcpd.conf-Pfad
- eth1 → Schnittstelle LAN

Aktivieren Sie den Dienst in runit:

```bash
sudo ln -s /etc/sv/dhcpd4 /var/service/
```

Start/Neustart:

```bash
sudo sv restart dhcpd4
```

Status prüfen:

```bash
sudo sv status dhcpd4
```

Erwartetes Ergebnis:

```bash
run: dhcpd4: (pid 17652) 831s; run: log: (pid 15544) 1213s
```

Überprüfen Sie, ob Port 67 überwacht wird

```bash
UNCONN 0      0            0.0.0.0:67        0.0.0.0:*    users:(("dhcpd",pid=17652,fd=6))  
```

Überwachen Sie DHCP in Echtzeit

```bash
sudo tcpdump -ni eth1 port 67 or port 68
```

Erwartetes Ergebnis

```bash
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth1, link-type EN10MB (Ethernet), snapshot length 262144 bytes
```

Zum direkten Debuggen (ohne Runit)

```bash
sudo dhcpd -4 -d -cf /etc/dhcpd.conf eth1
```

Das sollte sich zeigen

- DHCPDISCOVER
- DHCPANGEBOT
- DHCPREQUEST
- DHCPACK

Wichtige Dateien

- /etc/dhcpd.conf → Hauptkonfiguration
- /var/lib/dhcp/dhcpd.leases → Leasingverträge
- /etc/sv/dhcpd4/run → Skript runit
- /etc/sv/dhcpd4/conf → Dienstparameter
- /var/service/dhcpd4 → Dienst aktiv

Passen Sie das iptables-Skript an, um DHCP im LAN zuzulassen. Fügen Sie VOR den impliziten DROP-Regeln Folgendes hinzu:

```bash
# ============================
# DHCP LAN
# ============================

iptables -A INPUT  -i $LAN -p udp --sport 67:68 --dport 67:68 -j ACCEPT
iptables -A OUTPUT -o $LAN -p udp --sport 67:68 --dport 67:68 -j ACCEPT
```

💡 DHCP nutzt Broadcast → ohne dies erhält der Client keine IP.

Wenden Sie die Firewall erneut an:

```bash
sudo /usr/local/bin/firewall
```

Testen auf einer LAN-VM

```bash
dhclient -v
```

In der Firewall überwachen

```bash
sudo tail -f /var/log/messages
```

Oder

```bash
sudo tcpdump -ni eth1 port 67 or port 68
```

## 15. 🎉 CHECKLISTE FINALE

- Unsichtbares SSH ohne Klopfen
- Einweg-Knock
- Kurzes Zugriffsfenster
- Fail2ban aktiv nach der Authentifizierung
- Verbot, Klopfen zu ignorieren
- Funktionelles NAT
- Permanente Firewall
- Proxmox nur über Tunnel erreichbar
- Minimales rekursives DNS (bis PDC eintritt)
- DHCP-Server

---

🎯 DAS IST ALLES, LEUTE!

👉 https://t.me/z3r0l135
👉 https://t.me/vcatafesta
