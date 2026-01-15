# 🧩 TUTORIAL VOID LINUX - IMPLEMENTACIÓN FIREWALL - LABORATORIO EDUCATUX

📌 Firewall con IP Pública, Void Linux (glibc), IPTables (legacy), NAT, Port Knocking, Fail2ban, Servidor DHCP y DNS recursivo

---

## ✅ 1. TOPOLOGÍA DE RED

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

Vista desde otro ángulo

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

El firewall es el único host expuesto a Internet.

## ✅ 2. OBJETIVOS Y SUPUESTOS

- Denegar política predeterminada
- Enrutamiento IPv4 activo
- El escáner nunca ve la puerta
- Firewall como único punto de entrada
- No se han publicado paneles web
- SSH protegido por Port Knocking
- Control de fuerza bruta a través de Fail2ban
- NAT controlada para la LAN
- Administración remota a través de túnel SSH

## ✅ 3. ACTUALIZAR E INSTALAR LOS PAQUETES NECESARIOS

Actualizar el sistema

```bash
sudo xbps-install -Syu
```

Instalar los paquetes

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

## ✅ 4. CONFIGURACIÓN SSH

```bash
sudo vim /etc/ssh/sshd_config
```

Ajustar las líneas puntiagudas

```bash
Port 2222
ListenAddress 0.0.0.0

PermitRootLogin yes        # TEMPORÁRIO (remover após hardening)
PasswordAuthentication yes
UsePAM no

SyslogFacility AUTH
LogLevel INFO
```

Fail2ban depende del registro, garantiza las líneas

```bash
SyslogFacility AUTH
LogLevel INFO
```

Confirmar generación de registro

```bash
sudo tail -f /var/log/auth.log
```

## Activación del servicio

```bash
sudo ln -s /etc/sv/sshd /var/service/
sudo sv start sshd
```

## Después de la implementación completa:

- Deshabilitar el inicio de sesión raíz

- Utilice solo autenticación de clave

## ✅ 5. CONFIGURACIÓN DE LA RED FIREWALL

```bash
sudo vim /etc/dhcpcd.conf
```

Contenido

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

## ✅ 6. LLAMADO DE PUERTOS – SOPORTE DEL NÚCLEO

Cargue el módulo requerido

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

## ✅ 7. IPTABLES DE CORTAFUEGOS

Habilitar el enrutamiento entre tarjetas de red Firewall

```bash
sudo vim /etc/sysctl.conf
```

Contenido

```bash
net.ipv4.ip_forward=1
```

Aplicar sin reiniciar:

```bash
sudo sysctl --system
```

Cree el script del firewall en /usr/local/bin

```bash
sudo vim /usr/local/bin/firewall
```

Contenido

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

Aplicar permiso y ejecutar

```bash
sudo chmod +x /usr/local/bin/firewall
sudo bash /usr/local/bin/firewall
```

## ✅ 8. PERSISTENCIA DEL FIREWALL EN RUNIT

Crear el directorio

```bash
sudo mkdir -p /etc/sv/firewall
```

Crea el archivo

```bash
sudo vim /etc/sv/firewall/run
```

Contenido

```bash
#!/bin/sh
exec /usr/local/bin/firewall
```

Activar, ejecutar y validar estado

```bash
sudo chmod +x /etc/sv/firewall/run
sudo ln -s /etc/sv/firewall /var/service/
sudo sv status firewall
```

## ✅ 9. PRUEBAS Y VALIDACIÓN (EN CALIENTE) DEL GOLPE DE PUERTOS

Monitor de golpe en un terminal SIN FIREWALL

```bash
sudo tcpdump -ni eth0 tcp port 12345
```

Enviar el golpe POR CUADERNO mediante acceso EXTERNO

```bash
sudo nc -z 39.236.83.109 12345
```

✔ Llega SYN
✔ Está CAÍDO
✔ Manténgase registrado
✔ el estado es visible

Resultado esperado en tcpdump

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

Nota técnica importante

- El RST se envía a través de la pila TCP.
- El paquete está registrado por xt_recent
- El puerto no responde como un servicio.
- No hay pancarta ni huella digital.

Validar registro de IP

```bash
sudo cat /proc/net/xt_recent/SSH_KNOCK
```

Resultado esperado

```bash
src=99.336.74.209 ttl: 61 last_seen: 4302299386 oldest_pkt: 7 4302292227, 4302293242, 4302294266, 4302295290, 4302296314, 4302297338, 4302299386
```

SI quieres borrar todos los golpes

```bash
sudo echo clear > /proc/net/xt_recent/SSH_KNOCK
```

## ✅ 10. REALIZAR ACCESO ADMINISTRATIVO EXTERNO

ejecutar el golpe

```bash
nc -z 39.236.83.109 12345
```

En 15 segundos, acceda

```bash
ssh -p 2222 supertux@39.236.83.109
```

Alias recomendados

```bash
vim ~/.bashrc
```

Contenido

```bash
alias knock='nc -z 39.236.83.109 12345'
alias firewall='ssh -p 2222 supertux@39.236.83.109'
```

Vuelva a leer el archivo para su validación.

```bash
source ~/.bashrc
```

11. ✅ FAIL2BAN – PROTECCIÓN POST-GOLPE

Ajustes de registro para cumplir con fail2ban

```bash
sudo xbps-install -y socklog-void
sudo ln -s /etc/sv/socklog-unix /var/service/
sudo ln -s /etc/sv/nanoklogd /var/service/
sudo touch /var/log/auth.log
```

Crear archivo de configuración (Nunca editar jail.conf)

```bash
sudo vim /etc/fail2ban/jail.local
```

Contenido:

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

Activación de ejecución

```bash
sudo ln -s /etc/sv/fail2ban /var/service/
sudo sv start fail2ban
sudo sv status fail2ban
```

## 12. ✅ PRUEBA FAIL2BAN (¡ATENCIÓN, TE BLOQUEAS!)

Ejecutar o tocar

```bash
nc -z 39.236.83.109 12345
```

Pruebe SSH con la contraseña incorrecta 3 veces

comprobar la prohibición

```bash
sudo fail2ban-client status sshd
```

Desbanear manualmente:

```bash
sudo fail2ban-client set sshd unbanip X.X.X.X
```

## ⚠️ ATENCIÓN: ¡¡LAS SIGUIENTES SECCIONES 13 y 14, QUE TRATAN CON DNS RECURSIVO Y SERVIDOR DHCP, DEBEN DESCARTARSE DESPUÉS DE ACTUALIZAR SAMBA4 COMO PDC!!

## 13. ✅ IMPLEMENTAR UN DNS RECURSIVO TEMPORAL PARA SERVIR LA RED INTERNA

```bash
sudo xbps-install -y unbound
```

Configuración mínima

```bash
sudo vim /etc/unbound/unbound.conf
```

Contenido

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

Activar servicio (runit):

```bash
ln -s /etc/sv/unbound /var/service/
sv start unbound
```

## 14. ✅ IMPLEMENTACIÓN DE SERVIDOR DHCP TEMPORAL PARA SERVIR LA RED INTERNA

Instalación del paquete

```bash
sudo xbps-install -y dhcp
```

Este paquete instala:

- dhcpd (servidor)
- Estructura del servicio Runit:
/etc/sv/dhcpd4
/etc/sv/dhcpd6

Edite el archivo y configure los ajustes para la red interna

```bash
sudo vim /etc/dhcpd.conf
```

Contenido

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

Cree el archivo de arrendamiento:

```bash
sudo mkdir -p /var/lib/dhcp
sudo touch /var/lib/dhcp/dhcpd.leases
```

Creación de servicios Runit

```bash
sudo vim /etc/sv/dhcpd4/conf
```

Contenido

```bash
OPTS="-4 -q -cf /etc/dhcpd.conf eth1"
```

Explicación:

- -4 → IPv4
- -q → modo silencioso
- -cf → ruta correcta de dhcpd.conf
- eth1 → interfaz LAN

Activa el servicio en runit:

```bash
sudo ln -s /etc/sv/dhcpd4 /var/service/
```

Iniciar/Reiniciar:

```bash
sudo sv restart dhcpd4
```

Verificar estado:

```bash
sudo sv status dhcpd4
```

Resultado esperado:

```bash
run: dhcpd4: (pid 17652) 831s; run: log: (pid 15544) 1213s
```

Verifique el puerto 67 escuchando

```bash
UNCONN 0      0            0.0.0.0:67        0.0.0.0:*    users:(("dhcpd",pid=17652,fd=6))  
```

Supervise DHCP en tiempo real

```bash
sudo tcpdump -ni eth1 port 67 or port 68
```

Resultado esperado

```bash
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth1, link-type EN10MB (Ethernet), snapshot length 262144 bytes
```

Para depuración directa (sin runit)

```bash
sudo dhcpd -4 -d -cf /etc/dhcpd.conf eth1
```

Esto debería mostrar

- DHCPDESCUBRIR
- DHCPOFFER
- DHCPREQUEST
- DHCPACK

Archivos importantes

- /etc/dhcpd.conf → Configuración principal
- /var/lib/dhcp/dhcpd.leases → Arrendamientos
- /etc/sv/dhcpd4/run → Ejecución del script
- /etc/sv/dhcpd4/conf → Parámetros de servicio
- /var/service/dhcpd4 → Servicio activo

Ajuste el script iptables para permitir DHCP en la LAN. Agregue ANTES de las reglas DROP implícitas:

```bash
# ============================
# DHCP LAN
# ============================

iptables -A INPUT  -i $LAN -p udp --sport 67:68 --dport 67:68 -j ACCEPT
iptables -A OUTPUT -o $LAN -p udp --sport 67:68 --dport 67:68 -j ACCEPT
```

💡 DHCP usa transmisión → sin esto, el cliente no obtiene una IP.

Vuelva a aplicar el firewall:

```bash
sudo /usr/local/bin/firewall
```

Prueba en una VM LAN

```bash
dhclient -v
```

En el firewall, monitoree

```bash
sudo tail -f /var/log/messages
```

O

```bash
sudo tcpdump -ni eth1 port 67 or port 68
```

## 15. 🎉 LISTA DE VERIFICACIÓN FINAL

- SSH invisible sin golpes
- Golpe de un solo uso
- Ventana de acceso corto
- Fail2ban post-autenticación activa
- Prohibir ignorar el golpe
- NAT funcional
- Cortafuegos persistente
- A Proxmox solo se puede acceder a través de un túnel
- DNS recursivo mínimo (Hasta que entre PDC)
- Servidor DHCP

---

🎯 ¡ESO ES TODO AMIGOS!

👉https://t.me/z3r0l135
👉https://t.me/vcatafesta
