#  🧩 TUTORIAL VOID LINUX — IMPLEMENTAZIONE DELLO SCHEMA DI SICUREZZA – WORKSHOP IN LABORATORIO

📌 Firewall con IP pubblico, Void Linux (glibc), IPTables (legacy), NAT, Port Knocking e Fail2ban

---

## ✅ 1. TOPOLOGIA DELLA RETE

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

Vista da un'altra angolazione

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

Il firewall è l'unico host esposto a Internet.

## ✅ 2. OBIETTIVI E ASSUNZIONI

- Nega la politica predefinita
- Routing IPv4 attivo
- Lo scanner non vede mai la porta
- Firewall come unico punto di accesso
- Nessun dashboard web pubblicato
- SSH protetto da Port Knocking
- Controllo della forza bruta tramite Fail2ban
- NAT controllato per la LAN
- Amministrazione remota tramite tunnel SSH

## ✅ 3. AGGIORNA E INSTALLA I PACCHETTI NECESSARI

Aggiorna il sistema

```bash
sudo xbps-install -Syu
```

Installa i pacchetti

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

## ✅ 4. CONFIGURAZIONE SSH

```bash
sudo vim /etc/ssh/sshd_config
```

Regola le linee appuntite

```bash
Port 2222
ListenAddress 0.0.0.0

PermitRootLogin yes        # TEMPORÁRIO (remover após hardening)
PasswordAuthentication yes
UsePAM no

SyslogFacility AUTH
LogLevel INFO
```

Fail2ban dipende dal log, garantisce le linee

```bash
SyslogFacility AUTH
LogLevel INFO
```

Conferma la generazione del registro

```bash
sudo tail -f /var/log/auth.log
```

## Attivazione del servizio

```bash
sudo ln -s /etc/sv/sshd /var/service/
sudo sv start sshd
```

## Dopo la distribuzione completa:

- Disabilita l'accesso root

- Utilizzare solo l'autenticazione con chiave

## ✅ 5. CONFIGURAZIONE DELLA RETE FIREWALL

```bash
sudo vim /etc/dhcpcd.conf
```

Contenuto

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

Fare domanda a

```bash
sudo sv restart dhcpcd
```

## ✅ 6. BATTUTA DEL PORTO – SUPPORTO DEL NOCCIOLO

Caricare il modulo richiesto

```bash
sudo modprobe xt_recent
```

Convalidare:

```bash
sudo lsmod | grep xt_recent
```

Risultato atteso

```bash
xt_recent              24576  0
x_tables               65536  1 xt_recent
```

## ✅ 7. IPTABLES FIREWALL

Abilita il routing tra le schede di rete Firewall

```bash
sudo vim /etc/sysctl.conf
```

Contenuto

```bash
net.ipv4.ip_forward=1
```

Applica senza riavviare:

```bash
sudo sysctl --system
```

Crea lo script del firewall in /usr/local/bin

```bash
sudo vim /usr/local/bin/firewall
```

Contenuto

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

Applicare l'autorizzazione ed eseguire

```bash
sudo chmod +x /usr/local/bin/firewall
sudo bash /usr/local/bin/firewall
```

## ✅ 8. PERSISTENZA DEL FIREWALL IN RUNIT

Crea la directory

```bash
sudo mkdir -p /etc/sv/firewall
```

Crea il file

```bash
sudo vim /etc/sv/firewall/run
```

Contenuto

```bash
#!/bin/sh
exec /usr/local/bin/firewall
```

Attiva, esegui e convalida lo stato

```bash
chmod +x /etc/sv/firewall/run
ln -s /etc/sv/firewall /var/service/
sv status firewall
```

## ✅ 9. TEST E VALIDAZIONE (A CALDO) DEI PORTO

Monitorare il knock su un terminale SENZA FIREWALL

```bash
sudo tcpdump -ni eth0 tcp port 12345
```

Invia la bussata TRAMITE NOTEBOOK tramite accesso ESTERNO

```bash
sudo nc -z 39.236.83.109 12345
```

✔ Arriva il SYN
✔ È caduto
✔ Rimani registrato
✔ lo stato è visibile

Risultato previsto in tcpdump

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

Nota tecnica importante

- L'RST viene inviato tramite lo stack TCP
- Il pacchetto è registrato da xt_recent
- La porta non risponde come servizio
- Non c'è banner o impronta digitale

Convalidare la registrazione IP

```bash
cat /proc/net/xt_recent/SSH_KNOCK
```

Risultato atteso

```bash
src=99.336.74.209 ttl: 61 last_seen: 4302299386 oldest_pkt: 7 4302292227, 4302293242, 4302294266, 4302295290, 4302296314, 4302297338, 4302299386
```

SE vuoi eliminare tutti i colpi

```bash
echo clear > /proc/net/xt_recent/SSH_KNOCK
```

## ✅ 10. EFFETTUARE L'ACCESSO AMMINISTRATIVO ESTERNO

Esegui il colpo

```bash
nc -z 39.236.83.109 12345
```

Entro 15 secondi, accesso

```bash
ssh -p 2222 supertux@39.236.83.109
```

Alias consigliati

```bash
sudo vim .bashrc
```

Contenuto

```bash
alias knock='nc -z 39.236.83.109 12345'
alias officinas='ssh -p 2222 supertux@39.236.83.109'
```

Rileggere il file per la convalida

```bash
source .bashrc
```

11. ✅ FAIL2BAN – PROTEZIONE POST-KNOCK

Crea file di configurazione (non modificare mai jail.conf)

```bash
sudo vim /etc/fail2ban/jail.local
```

Contenuto:

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

Attivazione di Runit

```bash
sudo ln -s /etc/sv/fail2ban /var/service/
sudo sv start fail2ban
sudo sv status fail2ban
```

12. ✅ TEST FAIL2BAN (ATTENZIONE ti chiudi fuori durante l'accesso esterno)

Esegui o bussa

```bash
nc -z 39.236.83.109 12345
```

Prova SSH con la password sbagliata 3 volte

Controlla il divieto

```bash
sudo fail2ban-client status sshd
```

Sblocca manualmente:

```bash
sudo fail2ban-client set sshd unbanip X.X.X.X
```

13. 🎉 CHECKLIST FINALE

- SSH invisibile senza bussare
- Bussare monouso
- Finestra di accesso breve
- Fail2ban attivo dopo l'autenticazione
- Divieto di ignorare bussare
- NAT funzionale
- Firewall persistente
- Proxmox accessibile solo tramite tunnel

---

🎯 E' TUTTO RAGAZZE!
👉 https://t.me/z3r0l135
👉 https://t.me/vcatafesta
















































































