
# Файловый сервер под управлением Samba4 на сервере Void Linux ;D

## 🎯 Цель — развернуть файловый сервер на Void Linux (glibc), скомпилировав Samba4 из исходного кода, интеграцию с AD, списки управления доступом, службы и весь стек, необходимый файловому серверу для обслуживания сетевых клиентов.

## 🔧 Сетевая лаборатория с QEMU/Virtmanager и Proxmox. Настройте руководство в соответствии с вашей средой.

---

## 📡 Схема локальной сети

- Домен: EDUCATUX.EDU

- Имя хоста: файловый сервер

- Межсетевой экран 192.168.70.254 (DNS/GW)

- IP: 192.168.70.251

## Установить Войд Линукс

## Изменить оболочку по умолчанию в Void

```bash
chsh -s /bin/bash
```

## 🧩 Установите пакеты зависимостей для компиляции Samba4 на Void.

```bash
xbps-install -S \
 net-tools rsync acl attr attr-devel autoconf automake libtool \
 binutils bison gcc make ccache chrpath curl \
 docbook-xml docbook-xsl flex gdb git htop \
 mit-krb5 mit-krb5-client mit-krb5-devel \
 libarchive-devel avahi avahi-libs libblkid-devel \
 libbsd-devel libcap-devel cups-devel dbus-devel glib-devel \
 gnutls-devel gpgme-devel icu-devel jansson-devel \
 lmdb lmdb-devel libldap-devel ncurses-devel pam-devel perl \
 perl-Text-ParseWords perl-JSON perl-Parse-Yapp \
 libpcap-devel popt-devel readline-devel \
 libtasn1 libtasn1-devel libunwind-devel python3 python3-devel \
 python3-dnspython python3-cryptography \
 python3-matplotlib python3-pexpect python3-pyasn1 \
 tree libuuid-devel wget xfsprogs-devel zlib-devel \
 bind ldns pkg-config vim
```

## 🖥️ Установить имя хоста

```bash
echo "fileserver" > /etc/hostname
```

## 🏠 /etc/hosts

```bash
vim /etc/hosts
```

## Содержание:

```bash
127.0.0.1      localhost
127.0.1.1      fileserver.educatux.edu fileserver
192.168.70.251 fileserver.educatux.edu fileserver
```

## 🌐 Настройка статического IP

## 👉 Мы будем использовать стандартный метод Void, /etc/dhcpcd.conf.

```bash
vim /etc/dhcpcd.conf
```

## Добавьте IP, шлюз (маршрутизатор) и DNS (AD):

```bash
interface eth0
static ip_address=192.168.70.251/24
static routers=192.168.70.254
static domain_name_servers=192.168.70.253
```

## Перезапустите сетевой интерфейс:

```bash
sv restart dhcpcd
```

## 🧭 Установить DNS-адрес — укажите PDC.

```bash
vim /etc/resolv.conf
```

## Содержание:

```bash
domain educatux.edu
search educatux.edu
nameserver 192.168.70.253
```

## Блокировка resolv.conf

```bash
chattr +i /etc/resolv.conf
```

## 🔍 Проверьте адрес сетевого интерфейса.

```bash
ip -c addr
ip -br link
```

## 📥 Загрузите и извлеките исходный код Samba4.

```bash
wget https://download.samba.org/pub/samba/samba-4.23.4.tar.gz
```

```bash
tar -xvzf samba-4.23.4.tar.gz
```

## Скомпилировать и установить из исходников

```bash
cd samba-4.23.4
```

```bash
./configure --prefix=/opt/samba
```

```bash
make -j$(nproc) && make install
```

## Примечания:

- Void не мешает, поскольку Samba скомпилирована в /opt/samba.

- make -j значительно ускоряет компиляцию, но всё равно идите выпейте кофе.

- После установки скомпилированная Samba4 не имеет никаких служб запуска.

- Мы создадим сервисы вручную.

## 📁 Добавьте Samba4 в системный PATH и перезагрузите среду.

```bash
echo 'export PATH=/opt/samba/bin:/opt/samba/sbin:$PATH' >> /etc/profile
```

```bash
source /etc/profile
```

## Тестирование вставки Samba4 PATH в ОС

```bash
samba-tool -V
```

## Выход:

```bash
4.23.4
```

## ⚠️ Внимание: НЕ используйте команду подготовки на файловом сервере!

## 📝 Создайте файл smb.conf.

```bash
vim /opt/samba/etc/smb.conf
```

```bash
[global]
   workgroup = EDUCATUX
   security = ads
   realm = EDUCATUX.EDU
   encrypt passwords = yes
   # point to the services, the active interfaces
   interfaces = eth0
   bind interfaces only = yes

   log file = /opt/samba/var/log.%m
   max log size = 50

   winbind use default domain = yes
   winbind enum users = yes
   winbind enum groups = yes
   winbind refresh tickets = yes

   idmap config * : backend = tdb
   idmap config * : range = 3000-7999
   idmap config EDUCATUX : backend = rid
   idmap config EDUCATUX : range = 10000-999999

   template shell = /bin/bash
   template homedir = /home/%U

[Public]
   path = /srv/samba/public
   browsable = yes
   writable = yes
   guest ok = no
   create mask = 0660
   directory mask = 0770
``` 

## Создайте файл журнала

```bash
mkdir /opt/samba/var
```

## 📂 Создайте путь обмена

```bash
sudo mkdir -p /srv/samba/public
sudo chown -R root:root /srv/samba/public
sudo chmod -R 0770 /srv/samba/public
```

## Перезагрузить конфигурацию Samba4.

```bash
smbcontrol all reload-config
```

## 🕒 Сервер NTP/Chrony

## Контроллер домена должен быть локальным сервером времени, поскольку после 5-минутного отклонения Kerberos больше не будет аутентифицировать клиентов.

## Установить Хрони

```bash
xbps-install -Syu chrony
```

## Отредактируйте конфигурацию и разрешите внутреннюю сеть

```bash
vim /etc/chrony.conf
```

## Установите контроль домена на серверах времени:

```bash
# Comment the external line
#pool pool.ntp.org iburst

# PDC Time Servers
server 192.168.70.253 iburst
```

## Включить chronyd в runit

```bash
ln -sf /etc/sv/chronyd/ /var/service/
```

## Перезапустить службу:

```bash
sv restart chronyd
```

## Проверьте серверы:

```bash
chronyc sources -v
```

## 🔐 Создайте файл Kerberos — укажите PDC.

```bash
vim /etc/krb5.conf
```

## содержащий

```bash
[libdefaults]
    default_realm = EDUCATUX.EDU
    dns_lookup_realm = true
    dns_lookup_kdc = true
    rdns = false
    forwardable = true
    proxiable = true

[realms]
    EDUCATUX.EDU = {
        kdc = 192.168.70.253
        admin_server = 192.168.70.253
        default_domain = educatux.edu
    }

[domain_realm]
    .educatux.edu = EDUCATUX.EDU
    educatux.edu = EDUCATUX.EDU
```

## Тест Кербероса

```bash
kinit Administrator
```

```bash
klist
```

## Результат получен

```bash
Ticket cache: FILE:/tmp/krb5cc_0
Default principal: administrator@EDUCATUX.EDU

Valid starting       Expires              Service principal
11/12/2025 09:43:40  11/12/2025 19:43:40  krbtgt/EDUCATUX.EDU@EDUCATUX.EDU
	renew until 12/12/2025 09:43:36
```

## 🔗 Свяжите библиотеки Winbind с системой

## Проверьте путь к libdir:

```bash
/opt/samba/sbin/smbd -b | grep LIBDIR
```

## Ожидал:

```bash
LIBDIR: /opt/samba/lib
```

## Создайте ссылки на библиотеки (предпочтительно вводить вручную):

```bash
ln -sf /opt/samba/lib/libnss_winbind.so.2 /usr/lib/
```

```bash
ln -sf /usr/lib/libnss_winbind.so.2 /usr/lib/libnss_winbind.so
```

## Перезагрузить кеш библиотеки:

```bash
ldconfig
```

## Обновите nsswitch для обмена билетами Kerberos (добавьте winbind):

```bash
vim /etc/nsswitch.conf
```

```bash
passwd: files winbind
group:  files winbind
```

## Остальное оставьте нетронутым

## Присоединяйтесь к домену

```bash
net ads join -U Administrator
```

## Результат получен

```bash
Password for [EDUCATUX\Administrator]:
Using short domain name -- EDUCATUX
Joined 'VOIDFILES' to dns domain 'educatux.edu'
```

## 📦 Создайте службы RUNIT для smbd, winbindd и, при необходимости, nmbd.

### Void Linux использует runit в качестве системы инициализации, а его собственный регистратор — svlogd, включенный в базовый пакет runit. Никаких дополнительных пакетов не требуется.

## SMBD — Сервис и ведение журнала

## Создание каталогов служб и журналов

```bash
mkdir -p /etc/sv/smbd/log
mkdir -p /var/log/smbd
```

## Создайте /etc/sv/smbd/run.

```bash
vim /etc/sv/smbd/run
```

## Содержание

```bash
#!/bin/sh
exec 2>&1
exec /opt/samba/sbin/smbd --foreground --no-process-group
```

## Разрешение

```bash
chmod +x /etc/sv/smbd/run
```

## Создайте /etc/sv/smbd/log/run.

```bash
vim /etc/sv/smbd/log/run
```

## Содержание

```bash
#!/bin/sh
exec svlogd -tt /var/log/smbd
```

## Разрешение

```bash
chmod +x /etc/sv/smbd/log/run
```

## Отладка (необязательно)

```bash
/opt/samba/sbin/smbd -i
```

## Результат получен

```bash
smbd version 4.23.4 started.
Copyright Andrew Tridgell and the Samba Team 1992-2025
daemon 'smbd' : Starting process ...
```

## WINBINDD — Сервис и ведение журнала

## Создание каталогов служб и журналов

```bash
mkdir -p /etc/sv/winbindd/log
mkdir -p /var/log/winbindd
```

## Создайте /etc/sv/winbindd/run.

```bash
vim /etc/sv/winbindd/run
```

## Содержание

```bash
#!/bin/sh
exec 2>&1
exec /opt/samba/sbin/winbindd --foreground --no-process-group
```

## Разрешение

```bash
chmod +x /etc/sv/winbindd/run
```

## Создайте /etc/sv/winbindd/log/run.

```bash
vim /etc/sv/winbindd/log/run
```

## Содержание

```bash
#!/bin/sh
exec 2>&1
exec /opt/samba/sbin/winbindd --foreground --no-process-group
```

## Разрешение

```bash
chmod +x /etc/sv/winbindd/log/run
```

## Отладка (необязательно)

```bash
/opt/samba/sbin/winbindd -i
```

## Результат получен

```bash
winbindd version 4.23.4 started.
Copyright Andrew Tridgell and the Samba Team 1992-2025
daemon 'winbindd' : Starting process ...
```

## NMBD — Сервис и журналирование (необязательно)

### Включайте только в том случае, если ваша среда требует просмотра NetBIOS/SMB1.

## Создание каталогов служб и журналов

```bash
mkdir -p /etc/sv/nmbd/log
mkdir -p /var/log/nmbd
```

## Создайте /etc/sv/nmbd/run.

```bash
vim /etc/sv/nmbd/run
```

## Содержание

```bash
#!/bin/sh
exec 2>&1
exec /opt/samba/sbin/nmbd --foreground --no-process-group
```

## Разрешение

```bash
chmod +x /etc/sv/nmbd/run
```

## Создайте /etc/sv/nmbd/log/run.

```bash
vim /etc/sv/nmbd/log/run
```
 
## Содержание

```bash
#!/bin/sh
exec svlogd -tt /var/log/nmbd
```

## Разрешение

```bash
chmod +x /etc/sv/nmbd/log/run
```

## Включить службы

```bash
ln -sf /etc/sv/smbd /var/service/
ln -sf /etc/sv/winbindd /var/service/
```

## Необязательно — включать только при использовании NetBIOS:

```bash
ln -sf /etc/sv/nmbd /var/service/
```

## Проверка услуг

```bash
sv status smbd winbindd nmbd
```

## 🧪 Подтвердить интеграцию

```bash
net ads testjoin
```

## Результат получен

```bash
Join is OK
```

```bash
wbinfo -u
```

## Результат получен

```bash
guest
krbtgt
administrator```

```bash
wbinfo -g
```

## Result obtained

```bash
корпоративные контроллеры домена только для чтения
защищенные пользователи
контроллеры домена
гости домена
контроллеры домена только для чтения
администраторы схемы
dnsupdateproxy
администраторы домена
владельцы создателей групповой политики
ras и ias серверы
DNSadmins
разрешенная группа репликации паролей Rodc
администраторы предприятия
издатели сертификатов
пользователи домена
группа репликации пароля Rodc отклонена
компьютеры домена
```

```bash
wbinfo --ping-dc
```

## Result obtained

```bash
проверка подключения NETLOGON для домена [EDUCATUX] постоянного тока к «voiddc.educatux.edu» прошла успешно
```

## ✅ FINAL SUMMARY

## 🎉 Congratulations — you have successfully deployed a fully functional File Server on Void Linux!

## 👉 REMEMBER: While Samba4 can be managed via CLI, it was designed to be managed via RSAT Remote Server Administration Tools, which can be installed on a Windows 10 machine without issues!

---

🎯 THAT'S ALL FOLKS!

👉 Contact: zerolies@disroot.org

👉 https://t.me/z3r0l135
