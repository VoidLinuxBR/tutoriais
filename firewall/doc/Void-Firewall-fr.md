#  🧩 TUTORIEL VOID LINUX — MISE EN ŒUVRE DU SYSTÈME DE SÉCURITÉ – ATELIERS DE LABORATOIRE

📌 Pare-feu avec IP Público, Void Linux (glibc), IPTables (legacy), NAT, Port Knocking et Fail2ban

---

## ✅ 1. TOPOLOGIE DU RÉSEAU

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

Vue sous un autre angle

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

Le pare-feu est le seul hôte exposé à Internet.

## ✅ 2. OBJECTIFS ET HYPOTHÈSES

- Refuser la stratégie par défaut
- Routage IPv4 actif
- Le scanner ne voit jamais la porte
- Le pare-feu comme seul point d'entrée
- Aucun tableau de bord Web publié
- SSH protégé par Port Knocking
- Contrôle par force brute via Fail2ban
- NAT contrôlé pour le LAN
- Administration à distance via tunnel SSH

## ✅ 3. METTRE À JOUR ET INSTALLER LES FORFAITS NÉCESSAIRES

Mettre à jour le système

```bash
sudo xbps-install -Syu
```

Installer les paquets

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

## ✅ 4. CONFIGURATION SSH

```bash
sudo vim /etc/ssh/sshd_config
```

Ajustez les lignes pointues

```bash
Port 2222
ListenAddress 0.0.0.0

PermitRootLogin yes        # TEMPORÁRIO (remover após hardening)
PasswordAuthentication yes
UsePAM no

SyslogFacility AUTH
LogLevel INFO
```

Fail2ban dépend du journal, garantissez les lignes

```bash
SyslogFacility AUTH
LogLevel INFO
```

Confirmer la génération du journal

```bash
sudo tail -f /var/log/auth.log
```

## Activation des services

```bash
sudo ln -s /etc/sv/sshd /var/service/
sudo sv start sshd
```

## Après le déploiement complet :

- Désactiver la connexion root

- Utiliser uniquement l'authentification par clé

## ✅ 5. CONFIGURATION DU RÉSEAU PARE-FEU

```bash
sudo vim /etc/dhcpcd.conf
```

Contenu

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

Appliquer

```bash
sudo sv restart dhcpcd
```

## ✅ 6. COUP DE PORT – SUPPORT DU NOYAU

Charger le module requis

```bash
sudo modprobe xt_recent
```

Valider:

```bash
sudo lsmod | grep xt_recent
```

Résultat attendu

```bash
xt_recent              24576  0
x_tables               65536  1 xt_recent
```

## ✅ 7. PARE-FEU IPTABLES

Activer le routage entre les cartes réseau du pare-feu

```bash
sudo vim /etc/sysctl.conf
```

Contenu

```bash
net.ipv4.ip_forward=1
```

Appliquer sans redémarrage :

```bash
sudo sysctl --system
```

Créez le script de pare-feu dans /usr/local/bin

```bash
sudo vim /usr/local/bin/firewall
```

Contenu

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

Appliquer l'autorisation et exécuter

```bash
sudo chmod +x /usr/local/bin/firewall
sudo bash /usr/local/bin/firewall
```

## ✅ 8. PERSISTANCE DU PARE-FEU EN RUNIT

Créer le répertoire

```bash
sudo mkdir -p /etc/sv/firewall
```

Créer le fichier

```bash
sudo vim /etc/sv/firewall/run
```

Contenu

```bash
#!/bin/sh
exec /usr/local/bin/firewall
```

Activer, exécuter et valider le statut

```bash
chmod +x /etc/sv/firewall/run
ln -s /etc/sv/firewall /var/service/
sv status firewall
```

## ✅ 9. TESTS ET VALIDATION (À CHAUD) DU PORT KNOCKING

Surveiller frapper sur un terminal SANS PARE-FEU

```bash
sudo tcpdump -ni eth0 tcp port 12345
```

Envoyez le coup PAR CARNET via un accès EXTERNE

```bash
sudo nc -z 39.236.83.109 12345
```

✔ SYN arrive
✔ C'est abandonné
✔ Restez inscrit
✔ le statut est visible

Résultat attendu dans tcpdump

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

Note technique importante

- Le RST est envoyé via la pile TCP
- Le package est enregistré par xt_recent
- Le port ne répond pas en tant que service
- Il n'y a pas de bannière ni d'empreinte digitale

Valider l'enregistrement IP

```bash
cat /proc/net/xt_recent/SSH_KNOCK
```

Résultat attendu

```bash
src=99.336.74.209 ttl: 61 last_seen: 4302299386 oldest_pkt: 7 4302292227, 4302293242, 4302294266, 4302295290, 4302296314, 4302297338, 4302299386
```

SI vous voulez effacer tous les coups

```bash
echo clear > /proc/net/xt_recent/SSH_KNOCK
```

## ✅ 10. EFFECTUER UN ACCÈS ADMINISTRATIF EXTERNE

Exécuter le coup

```bash
nc -z 39.236.83.109 12345
```

En 15 secondes, accédez

```bash
ssh -p 2222 supertux@39.236.83.109
```

Alias recommandés

```bash
sudo vim .bashrc
```

Contenu

```bash
alias knock='nc -z 39.236.83.109 12345'
alias officinas='ssh -p 2222 supertux@39.236.83.109'
```

Relisez le fichier pour validation

```bash
source .bashrc
```

11. ✅ FAIL2BAN – PROTECTION POST-COUP

Créer un fichier de configuration (ne modifiez jamais jail.conf)

```bash
sudo vim /etc/fail2ban/jail.local
```

Contenu:

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

Activation de l'unité d'exécution

```bash
sudo ln -s /etc/sv/fail2ban /var/service/
sudo sv start fail2ban
sudo sv status fail2ban
```

12. ✅ TEST FAIL2BAN (ATTENTION vous vous verrouillez lors d'un accès externe)

Exécuter ou frapper

```bash
nc -z 39.236.83.109 12345
```

Essayez SSH avec le mauvais mot de passe 3 fois

Vérifiez l'interdiction

```bash
sudo fail2ban-client status sshd
```

Annuler le ban manuellement :

```bash
sudo fail2ban-client set sshd unbanip X.X.X.X
```

13. 🎉 LISTE DE CONTRÔLE FINALE

- SSH invisible sans frapper
- Coup à usage unique
- Fenêtre d'accès courte
- Post-authentification active Fail2ban
- Interdire d'ignorer frapper
- NAT fonctionnel
- Pare-feu persistant
- Proxmox uniquement accessible via tunnel

---

🎯 C'EST TOUS LES GENS !
👉 https://t.me/z3r0l135
👉 https://t.me/vcatafesta
















































































