# 用于将 Linux Mint 加入域的 LightDM 配置

## 🎯 目标通过 NSS/PAM 集成使用 Winbind 将 Linux 加入 Samba4 域

---

## 先决条件

- Samba4 作为域控制器 (PDC)
- Linux 的 DNS 和时间与 PDC 一致 (192.168.70.253)
- 与服务器的连接

---

## 🛠️步骤

## 1.安装需要的包

```bash
sudo apt update && sudo apt install samba winbind libpam-winbind libnss-winbind krb5-user
```

## 2.配置/etc/krb5.conf

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

## 3.配置/etc/samba/smb.conf

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

## 4.配置/etc/nsswitch.conf

```bash
passwd:         compat winbind
group:          compat winbind
shadow:         compat
```

## 5. 加入域

```bash
sudo net ads join -U Administrator
```

## 6. 系统重启

```bash
sudo reboot
```

## 7. 检查

```bash
sudo net ads testjoin
wbinfo -u
wbinfo -g
getent passwd usuario
```

## 8.自动创建HOME目录


## 编辑 /etc/pam.d/common-session 并添加：

```bash
session required pam_mkhomedir.so skel=/etc/skel umask=0022
```

## 9. 重启服务

```bash
sudo systemctl restart smbd nmbd winbind
sudo systemctl enable winbind
```

## 10. 时间同步

```bash
sudo timedatectl set-ntp true
```

## 如果您是 Lightdm 用户，就像 Mint 的情况一样，请将其设置为使用网络用户登录，而不仅仅是本地用户。

## 🛠️ 一步一步：配置 LightDM 以接受域用户

## 1.编辑LightDM配置文件

```bash
sudo vim /etc/lightdm/lightdm.conf
```

## 编辑文件中的以下行：

```bash
[Seat:*]
greeter-show-manual-login=true
greeter-hide-users=true
allow-guest=false
```

## 说明：

- greeter-show-manual-login=true：允许您手动输入用户名。
- greeter-hide-users=true：隐藏本地用户列表（对于企业环境有用）。
- allow-guest=false：阻止访客登录（出于安全考虑）。

## 2. 确保 PAM 允许域用户

## 如果您使用 SSSD 或 Winbind，PAM 应该已经正确集成。但请确保 home 模块存在：

```bash
sudo vim /etc/pam.d/common-session
```

## 确认此行存在或添加它：

```bash
session required pam_mkhomedir.so skel=/etc/skel umask=0022
```

## 3. 重新启动LightDM

```bash
sudo timedatectl set-ntp true
```

## ⚠️ 对于基于 Ubuntu 的 Linux MInt，您可以禁用 systemd-resolved 来手动控制 DNS。

```bash
systemctl status systemd-resolved
```

```bash
systemctl stop systemd-resolved
systemctl disable systemd-resolved.service
sudo systemctl mask systemd-resolved
```

## 删除由 SYSTEMD-RESOLVED 创建的没有编辑权限的文件：

```bash
rm -f /etc/resolv.conf
```

## 创建具有编辑权限的新文件：

```bash
vim /etc/resolv.conf
```

```bash
domain educatux.edu
search educatux.edu.
nameserver 192.168.70.253
```

## 锁定文件以防止自动编辑

```bash
sudo chattr +i /etc/resolv.conf
```

## 服务重启

```bash
sudo systemctl restart NetworkManager
```

---

🎯 这就是大家！

👉联系方式：zerolies@disroot.org
👉 https://t.me/z3r0l135

