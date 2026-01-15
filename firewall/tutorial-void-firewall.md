# 🧩 TUTORIAL VOID LINUX - IMPLANTAÇÃO DO FIREWALL - LABORATÓRIO EDUCATUX

📌 Firewall com IP Público, Void Linux (glibc), IPTables (legacy), NAT, Port Knocking, Fail2ban, DHCP Server e DNS recursivo

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
  conntrack-tools \
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
sudo chmod +x /etc/sv/firewall/run
sudo ln -s /etc/sv/firewall /var/service/
sudo sv status firewall
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
sudo cat /proc/net/xt_recent/SSH_KNOCK
```

Resultado esperado

```bash
src=99.336.74.209 ttl: 61 last_seen: 4302299386 oldest_pkt: 7 4302292227, 4302293242, 4302294266, 4302295290, 4302296314, 4302297338, 4302299386
```

SE quiser limpar todos os knocks

```bash
sudo echo clear > /proc/net/xt_recent/SSH_KNOCK
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
vim ~/.bashrc
```

Conteúdo

```bash
alias knock='nc -z 39.236.83.109 12345'
alias firewall='ssh -p 2222 supertux@39.236.83.109'
```

Releia o arquivo para validação

```bash
source ~/.bashrc
```

11. ✅ FAIL2BAN – PROTEÇÃO PÓS-KNOCK

Ajustes de logs para atender o fail2ban

```bash
sudo xbps-install -y socklog-void
sudo ln -s /etc/sv/socklog-unix /var/service/
sudo ln -s /etc/sv/nanoklogd /var/service/
sudo touch /var/log/auth.log
```

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

## 12. ✅ TESTE DO FAIL2BAN (ATENÇÃO, VC SE TRANCA PRA FORA!)

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

## ⚠️ ATENÇÃO: AS SEÇÕES 13 e 14 SEGUINTES, QUE TRATAM DO DNS RECURSIVO E DHCP SERVER, DEVERÃO SER DESCATADAS APÓS SUBIR O SAMBA4 COMO PDC!!

## 13. ✅ IMPLANTANDO UM DNS RECURSIVO TEMPORÁRIO PARA ATENDER A REDE INTERNA

```bash
sudo xbps-install -y unbound
```

Configuração mínima

```bash
sudo vim /etc/unbound/unbound.conf
```

Conteúdo

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

Ativar serviço (runit):

```bash
ln -s /etc/sv/unbound /var/service/
sv start unbound
```

## 14. ✅ IMPLEMENTAÇÃO DO DHCP SERVER TEMPORÁRIO PARA ATENDER A REDE INTERNA

Instalação do pacote

```bash
sudo xbps-install -y dhcp
```

Esse pacote instala:

- dhcpd (servidor)
- Estrutura de serviço runit:
    /etc/sv/dhcpd4
    /etc/sv/dhcpd6

Editar o arquivo e setar as configurações para a rede interna

```bash
sudo vim /etc/dhcpd.conf
```

Conteúdo

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

Criar o arquivo de leases:

```bash
sudo mkdir -p /var/lib/dhcp
sudo touch /var/lib/dhcp/dhcpd.leases
```

Criação do serviço do runit

```bash
sudo vim /etc/sv/dhcpd4/conf
```

Conteúdo

```bash
OPTS="-4 -q -cf /etc/dhcpd.conf eth1"
```

Explicação:

- -4              → IPv4
- -q              → modo silencioso
- -cf             → caminho correto do dhcpd.conf
- eth1            → interface LAN

Ativar o serviço no runit:

```bash
sudo ln -s /etc/sv/dhcpd4 /var/service/
```

Iniciar/reiniciar:

```bash
sudo sv restart dhcpd4
```

Verificar status:

```bash
sudo sv status dhcpd4
```

Resultado esperado:

```bash
run: dhcpd4: (pid 17652) 831s; run: log: (pid 15544) 1213s
```

Verificar escuta da porta 67

```bash
UNCONN 0      0            0.0.0.0:67        0.0.0.0:*    users:(("dhcpd",pid=17652,fd=6))  
```

Monitorar DHCP em tempo real

```bash
sudo tcpdump -ni eth1 port 67 or port 68
```

Resultado esperado

```bash
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth1, link-type EN10MB (Ethernet), snapshot length 262144 bytes
```

Para debug direto (sem runit)

```bash
sudo dhcpd -4 -d -cf /etc/dhcpd.conf eth1
```

Isso deve mostrar

- DHCPDISCOVER
- DHCPOFFER
- DHCPREQUEST
- DHCPACK

Arquivos importantes

- /etc/dhcpd.conf                 → Configuração principal
- /var/lib/dhcp/dhcpd.leases      → Leases
- /etc/sv/dhcpd4/run              → Script runit
- /etc/sv/dhcpd4/conf             → Parâmetros do serviço
- /var/service/dhcpd4             → Serviço ativo

Ajuste no script do iptables para permitir DHCP na LAN. Adicione ANTES das regras DROP implícitas:

```bash
# ============================
# DHCP LAN
# ============================

iptables -A INPUT  -i $LAN -p udp --sport 67:68 --dport 67:68 -j ACCEPT
iptables -A OUTPUT -o $LAN -p udp --sport 67:68 --dport 67:68 -j ACCEPT
```

💡 DHCP usa broadcast → sem isso, cliente não pega IP.

Reaplique o firewall:

```bash
sudo /usr/local/bin/firewall
```

Testes em uma VM da LAN

```bash
dhclient -v
```

No firewall, acompanhar

```bash
sudo tail -f /var/log/messages
```

Ou

```bash
sudo tcpdump -ni eth1 port 67 or port 68
```

## 15. 🎉  CHECKLIST FINAL

- SSH invisível sem knock
- Knock de uso único
- Janela curta de acesso
- Fail2ban ativo pós-auth
- Ban ignora knock
- NAT funcional
- Firewall persistente
- Proxmox acessível apenas via túnel
- DNS recursivo mínimo (Até entrar o PDC)
- DHCP Server

---

🎯 THAT'S ALL FOLKS!

👉 https://t.me/z3r0l135
👉 https://t.me/vcatafesta
