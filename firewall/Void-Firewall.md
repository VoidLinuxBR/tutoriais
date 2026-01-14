#  🧩 TUTORIAL VOID LINUX — IMPLANTAÇÃO DO ESQUEMA DE SEGURANÇA – LABORATÓRIO OFFICINAS

📌 Firewall com IP Público, Void Linux (glibc), IPTables (legacy), NAT, Port Knocking e Fail2ban

---

## ✅ 1. TOPOLOGIA DA REDE

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

Vista de outro ângulo

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

O firewall é o único host exposto à Internet.

## ✅ 2. OBJETIVOS E PREMISSAS

- Política default deny
- Roteamento IPv4 ativo
- Scanner nunca vê a porta
- Firewall como único ponto de entrada
- Nenhum painel web publicado
- SSH protegido por Port Knocking
- Controle de brute-force via Fail2ban
- NAT controlado para a LAN
- Administração remota via túnel SSH

## ✅ 3. ATUALIZAÇÃO E INSTALAÇÃO DOS PACOTES NECESSÁRIOS

Atualize o sistema

```bash
sudo xbps-install -Syu
```

Instale os pacotes

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

## ✅ 4. CONFIGURAÇÃO DO SSH

```bash
sudo vim /etc/ssh/sshd_config
```

Ajustes as linhas apontadas

```bash
Port 2222
ListenAddress 0.0.0.0

PermitRootLogin yes        # TEMPORÁRIO (remover após hardening)
PasswordAuthentication yes
UsePAM no

SyslogFacility AUTH
LogLevel INFO
```

Fail2ban depende de log, garanta as linhas

```bash
SyslogFacility AUTH
LogLevel INFO
```

Confirme geração de log

```bash
sudo tail -f /var/log/auth.log
```

## Ativação do serviço

```bash
sudo ln -s /etc/sv/sshd /var/service/
sudo sv start sshd
```

## Após a implantação completa:

- Desabilitar login de root

- Usar apenas autenticação por chave

## ✅ 5. CONFIGURAÇÃO DE REDE DO FIREWALL

```bash
sudo vim /etc/dhcpcd.conf
```

Conteúdo

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

Aplicar

```bash
sudo sv restart dhcpcd
```

## ✅ 6. PORT KNOCKING – SUPORTE EM KERNEL

Carregar o módulo necessário

```bash
sudo modprobe xt_recent
```

Validar:

```bash
sudo lsmod | grep xt_recent
```

Resultado esperado

```bash
xt_recent              24576  0
x_tables               65536  1 xt_recent
```

## ✅ 7. FIREWALL IPTABLES

Habilite o roteamento entre as placas de rede do Firewall

```bash
sudo vim /etc/sysctl.conf
```

Conteúdo

```bash
net.ipv4.ip_forward=1
```

Aplicar sem reboot:

```bash
sudo sysctl --system
```

Criar o script do firewall em /usr/local/bin

```bash
sudo vim /usr/local/bin/firewall
```

Conteúdo

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

Aplicar permissão e executar

```bash
sudo chmod +x /usr/local/bin/firewall
sudo bash /usr/local/bin/firewall
```

## ✅ 8. PERSISTÊNCIA DO FIREWALL NO RUNIT

Cria o diretório

```bash
sudo mkdir -p /etc/sv/firewall
```

Cria o arquivo 

```bash
sudo vim /etc/sv/firewall/run
```

Conteúdo

```bash
#!/bin/sh
exec /usr/local/bin/firewall
```

Ativar, rodar e validar status

```bash
chmod +x /etc/sv/firewall/run
ln -s /etc/sv/firewall /var/service/
sv status firewall
```

## ✅ 9. TESTE E VALIDAÇÃO (Á QUENTE) DO PORT KNOCKING

Monitorar o knock em um terminal NO FIREWALL

```bash
sudo tcpdump -ni eth0 tcp port 12345
```

Enviar o knock PELO NOTEBOOK por acesso EXTERNO

```bash
sudo nc -z 39.236.83.109 12345
```

✔ o SYN chega
✔ É DROPado
✔ Fica registrado
✔ o estado está visível

Resultado esperado no tcpdump

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

Observação técnica importante

- O RST é enviado pelo stack TCP
- O pacote é registrado pelo xt_recent
- A porta não responde como serviço
- Não há banner nem fingerprint

Validar o registro do IP

```bash
cat /proc/net/xt_recent/SSH_KNOCK
```

Resultado esperado

```bash
src=99.336.74.209 ttl: 61 last_seen: 4302299386 oldest_pkt: 7 4302292227, 4302293242, 4302294266, 4302295290, 4302296314, 4302297338, 4302299386
```

SE quiser limpar todos os knocks

```bash
echo clear > /proc/net/xt_recent/SSH_KNOCK
```

## ✅ 10. REALIZAR O ACESSO ADMINISTRATIVO EXTERNO

Executar o knock

```bash
nc -z 39.236.83.109 12345
```

Dentro de 15 segundos fazer o acesso

```bash
ssh -p 2222 supertux@39.236.83.109
```

Aliases recomendados

```bash
sudo vim .bashrc
```

Conteúdo

```bash
alias knock='nc -z 39.236.83.109 12345'
alias officinas='ssh -p 2222 supertux@39.236.83.109'
```

Releia o arquivo para validação

```bash
source .bashrc
```

11. ✅ FAIL2BAN – PROTEÇÃO PÓS-KNOCK

Cria o arquivo de configuração (Nunca edite o jail.conf)

```bash
sudo vim /etc/fail2ban/jail.local
```

Conteúdo:

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

Ativação no runit

```bash
sudo ln -s /etc/sv/fail2ban /var/service/
sudo sv start fail2ban
sudo sv status fail2ban
```

12. ✅ TESTE DO FAIL2BAN (ATENÇÃO vc se tranca pra fora no acesso externo)

Execute o knock

```bash
nc -z 39.236.83.109 12345
```

Tente SSH errando a senha 3 vezes

Verifique o ban

```bash
sudo fail2ban-client status sshd
```

Desbanir manualmente:

```bash
sudo fail2ban-client set sshd unbanip X.X.X.X
```

13. 🎉  CHECKLIST FINAL

- SSH invisível sem knock
- Knock de uso único
- Janela curta de acesso
- Fail2ban ativo pós-auth
- Ban ignora knock
- NAT funcional
- Firewall persistente
- Proxmox acessível apenas via túnel

---

🎯 THAT'S ALL FOLKS!
👉 https://t.me/z3r0l135
👉 https://t.me/vcatafesta
















































































