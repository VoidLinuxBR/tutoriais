# Конфигурация LightDM для присоединения Linux Mint к домену

## 🎯 Цель Присоединить Linux к домену Samba4 с помощью Winbind посредством интеграции NSS/PAM.

---

## Предварительные условия

- Samba4 в качестве контроллера домена (PDC)
- Linux с DNS и временем, согласованным с PDC (192.168.70.253)
- Подключение к серверу

---

## 🛠️ Шаги

## 1. Установите необходимые пакеты.

```bash
sudo apt update && sudo apt install samba winbind libpam-winbind libnss-winbind krb5-user
```

## 2. Настройте /etc/krb5.conf.

```bash
[libdefaults]
    default_realm = EDUCATUX.EDU
    dns_lookup_realm = false
    dns_lookup_kdc = true
    kdc_timesync = 1
    ccache_type = 4
    forwardable = true
    proxiable = true
    rdns = false
    fcc-mit-ticketflags = true

[realms]
    EDUCATUX.EDU = {
        kdc = 192.168.70.253
        admin_server = 192.168.70.253
        default_domain = educatux.edu
    }

[domain_realm]
    .officinas.edu = EDUCATUX.EDU
    educatux.edu = EDUCATUX.EDU
```

## 3. Настройте /etc/samba/smb.conf.

```bash
[global]
   workgroup = EDUCATUX
   security = ads
   realm = EDUCATUX.EDU
   server role = member server
   interfaces = lo eth0
   bind interfaces only = yes


   winbind use default domain = true
   winbind enum users = yes
   winbind enum groups = yes
   winbind refresh tickets = yes
   winbind offline logon = yes

   idmap config * : backend = tdb
   idmap config * : range = 10000-19999

   idmap config EDUCATUX : backend = rid
   idmap config EDUCATUX : range = 20000-999999

   template shell = /bin/bash
   template homedir = /home/%U
```

## 4. Настройте /etc/nsswitch.conf.

```bash
passwd:         compat winbind
group:          compat winbind
shadow:         compat
```

## 5. Присоединитесь к домену

```bash
sudo net ads join -U Administrator
```

## 6. Перезагрузка системы.

```bash
sudo reboot
```

## 7. Чеки

```bash
sudo net ads testjoin
wbinfo -u
wbinfo -g
getent passwd usuario
```

## 8. Автоматически создавать ГЛАВНЫЕ каталоги


## Отредактируйте /etc/pam.d/common-session и добавьте:

```bash
session required pam_mkhomedir.so skel=/etc/skel umask=0022
```

## 9. Перезапустите службы.

```bash
sudo systemctl restart smbd nmbd winbind
sudo systemctl enable winbind
```

## 10. Синхронизация времени

```bash
sudo timedatectl set-ntp true
```

## ЕСЛИ вы являетесь пользователем Lightdm, как в случае с Mint, настройте его для входа в систему с сетевым пользователем, а не просто с локальным пользователем.

## 🛠️ Шаг за шагом: настройте LightDM для приема пользователей домена.

## 1. Отредактируйте файл конфигурации LightDM.

```bash
sudo vim /etc/lightdm/lightdm.conf
```

## Отредактируйте следующие строки в файле:

```bash
[Seat:*]
greeter-show-manual-login=true
greeter-hide-users=true
allow-guest=false
```

## Пояснения:

- Greeter-show-manual-login=true: позволяет ввести имя пользователя вручную.
- Greeter-hide-users=true: скрывает локальный список пользователей (полезно для корпоративных сред).
- allow-guest=false: запрещает гостевой вход (в целях безопасности).

## 2. Убедитесь, что PAM разрешает пользователям домена

## Если вы использовали SSSD или Winbind, PAM уже должен быть правильно интегрирован. Но убедитесь, что домашний модуль присутствует:

```bash
sudo vim /etc/pam.d/common-session
```

## Подтвердите наличие этой строки ИЛИ добавьте ее:

```bash
session required pam_mkhomedir.so skel=/etc/skel umask=0022
```

## 3. Перезапустите LightDM.

```bash
sudo timedatectl set-ntp true
```

## ⚠️ Для Linux Mint на базе Ubuntu вы можете отключить systemd-resolved для управления DNS вручную.

```bash
systemctl status systemd-resolved
```

```bash
systemctl stop systemd-resolved
systemctl disable systemd-resolved.service
sudo systemctl mask systemd-resolved
```

## УДАЛЕНИЕ ФАЙЛА БЕЗ РАЗРЕШЕНИЯ НА РЕДАКТИРОВАНИЕ, СОЗДАННОГО СИСТЕМОЙ-RESOLVED:

```bash
rm -f /etc/resolv.conf
```

## Создание нового файла с разрешением на редактирование:

```bash
vim /etc/resolv.conf
```

```bash
domain educatux.edu
search educatux.edu.
nameserver 192.168.70.253
```

## Блокировка файла от автоматического редактирования

```bash
sudo chattr +i /etc/resolv.conf
```

## Перезапуск службы

```bash
sudo systemctl restart NetworkManager
```

---

🎯ВОТ ВСЕ, ЛЮДИ!

👉 Контакт: Zerolies@disroot.org
👉 https://t.me/z3r0l135

