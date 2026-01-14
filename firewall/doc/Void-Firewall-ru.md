#  🧩 VOID LINUX TUTORIAL — РЕАЛИЗАЦИЯ СХЕМЫ БЕЗОПАСНОСТИ — ЛАБОРАТОРНЫЕ СЕМИНАРЫ

📌 Брандмауэр с IP Público, Void Linux (glibc), IPTables (устаревшая версия), NAT, Port Knocking и Fail2ban

---

## ✅ 1. ТОПОЛОГИЯ СЕТИ

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

Взгляд под другим углом

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

Брандмауэр — единственный хост, подключенный к Интернету.

## ✅ 2. ЦЕЛИ И ПРЕДПОЛОЖЕНИЯ

- Запретить политику по умолчанию
- Активная маршрутизация IPv4
- Сканер никогда не видит дверь
- Брандмауэр как единственная точка входа
- Веб-панели не опубликованы
- SSH защищен с помощью Port Knocking
- Контроль брутфорса через Fail2ban
- Контролируемый NAT для локальной сети
- Удаленное администрирование через SSH-туннель

## ✅ 3. ОБНОВИТЬ И УСТАНОВИТЬ НЕОБХОДИМЫЕ ПАКЕТЫ

Обновите систему

```bash
sudo xbps-install -Syu
```

Установите пакеты

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

## ✅ 4. КОНФИГУРАЦИЯ SSH

```bash
sudo vim /etc/ssh/sshd_config
```

Отрегулируйте заостренные линии

```bash
Port 2222
ListenAddress 0.0.0.0

PermitRootLogin yes        # TEMPORÁRIO (remover após hardening)
PasswordAuthentication yes
UsePAM no

SyslogFacility AUTH
LogLevel INFO
```

Fail2ban зависит от журнала, гарантируйте линии

```bash
SyslogFacility AUTH
LogLevel INFO
```

Подтвердить создание журнала

```bash
sudo tail -f /var/log/auth.log
```

## Активация услуги

```bash
sudo ln -s /etc/sv/sshd /var/service/
sudo sv start sshd
```

## После полного развертывания:

- Отключить root-вход

- Используйте только ключевую аутентификацию

## ✅ 5. НАСТРОЙКА СЕТИ БРАНДМАУЭРА

```bash
sudo vim /etc/dhcpcd.conf
```

Содержание

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

Применять

```bash
sudo sv restart dhcpcd
```

## ✅ 6. УДАЛЕНИЕ ПОРТОВ – ПОДДЕРЖКА ЯДРА

Загрузите необходимый модуль

```bash
sudo modprobe xt_recent
```

Подтвердить:

```bash
sudo lsmod | grep xt_recent
```

Ожидаемый результат

```bash
xt_recent              24576  0
x_tables               65536  1 xt_recent
```

## ✅ 7. IPTABLES БРАНДМАУЭРА

Включить маршрутизацию между сетевыми картами брандмауэра

```bash
sudo vim /etc/sysctl.conf
```

Содержание

```bash
net.ipv4.ip_forward=1
```

Применить без перезагрузки:

```bash
sudo sysctl --system
```

Создайте скрипт брандмауэра в /usr/local/bin.

```bash
sudo vim /usr/local/bin/firewall
```

Содержание

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

Применить разрешение и выполнить

```bash
sudo chmod +x /usr/local/bin/firewall
sudo bash /usr/local/bin/firewall
```

## ✅ 8. СОХРАНЕНИЕ БРАНДМАУРА В RUNIT

Создать каталог

```bash
sudo mkdir -p /etc/sv/firewall
```

Создать файл

```bash
sudo vim /etc/sv/firewall/run
```

Содержание

```bash
#!/bin/sh
exec /usr/local/bin/firewall
```

Активируйте, запустите и проверьте статус

```bash
chmod +x /etc/sv/firewall/run
ln -s /etc/sv/firewall /var/service/
sv status firewall
```

## ✅ 9. ТЕСТИРОВАНИЕ И ВАЛИДАЦИЯ (ГОРЯЧАЯ) СТОНКИ ПОРТОВ

Монитор стучит в терминал БЕЗ ФИРМЭРАЛА

```bash
sudo tcpdump -ni eth0 tcp port 12345
```

Отправить стук ЧЕРЕЗ НОУТБУК через ВНЕШНИЙ доступ

```bash
sudo nc -z 39.236.83.109 12345
```

✔ SYN прибывает
✔ Оно выпало
✔ Оставайтесь зарегистрированными
✔ статус виден

Ожидаемый результат в tcpdump

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

Важное техническое примечание

- RST отправляется через стек TCP.
- Пакет зарегистрирован xt_recent
- Порт не отвечает как услуга
- Нет баннера или отпечатка пальца

Подтвердить регистрацию IP

```bash
cat /proc/net/xt_recent/SSH_KNOCK
```

Ожидаемый результат

```bash
src=99.336.74.209 ttl: 61 last_seen: 4302299386 oldest_pkt: 7 4302292227, 4302293242, 4302294266, 4302295290, 4302296314, 4302297338, 4302299386
```

ЕСЛИ вы хотите устранить все удары

```bash
echo clear > /proc/net/xt_recent/SSH_KNOCK
```

## ✅ 10. ОСУЩЕСТВЛЯТЬ ВНЕШНИЙ АДМИНИСТРАТИВНЫЙ ДОСТУП

Выполнить стук

```bash
nc -z 39.236.83.109 12345
```

В течение 15 секунд получите доступ

```bash
ssh -p 2222 supertux@39.236.83.109
```

Рекомендуемые псевдонимы

```bash
sudo vim .bashrc
```

Содержание

```bash
alias knock='nc -z 39.236.83.109 12345'
alias officinas='ssh -p 2222 supertux@39.236.83.109'
```

Перечитайте файл для проверки.

```bash
source .bashrc
```

11. ✅ FAIL2BAN – ЗАЩИТА ПОСЛЕ УДАРНОСТИ

Создайте файл конфигурации (никогда не редактируйте Jail.conf)

```bash
sudo vim /etc/fail2ban/jail.local
```

Содержание:

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

Активация рунита

```bash
sudo ln -s /etc/sv/fail2ban /var/service/
sudo sv start fail2ban
sudo sv status fail2ban
```

12. ✅ ТЕСТ FAIL2BAN (ВНИМАНИЕ вы блокируетесь при внешнем доступе)

Выполнить или постучать

```bash
nc -z 39.236.83.109 12345
```

Попробуйте SSH с неправильным паролем 3 раза.

Проверьте бан

```bash
sudo fail2ban-client status sshd
```

Разблокировать вручную:

```bash
sudo fail2ban-client set sshd unbanip X.X.X.X
```

13. 🎉 ОКОНЧАТЕЛЬНЫЙ СПИСОК

- Невидимый SSH без стука
- Одноразовый стук
- Короткое окно доступа
- Активная пост-аутентификация Fail2ban
- Забанить игнорировать стук
- Функциональный НАТ
- Постоянный брандмауэр
- Proxmox доступен только через туннель

---

🎯ВОТ ВСЕ, ЛЮДИ!
👉 https://t.me/z3r0l135
👉 https://t.me/vcatafesta
















































































