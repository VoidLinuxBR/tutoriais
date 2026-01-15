# 用於將 Linux Mint 加入域的 LightDM 配置

## 🎯 目標通過 NSS/PAM 集成使用 Winbind 將 Linux 加入 Samba4 域

---

## 先決條件

- Samba4 作為域控制器 (PDC)
- Linux 的 DNS 和時間與 PDC 一致 (192.168.70.253)
- 與服務器的連接

---

## 🛠️步驟

## 1.安裝需要的包

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

## 6. 系統重啟

```bash
sudo reboot
```

## 7. 檢查

```bash
sudo net ads testjoin
wbinfo -u
wbinfo -g
getent passwd usuario
```

## 8.自動創建HOME目錄


## 編輯 /etc/pam.d/common-session 並添加：

```bash
session required pam_mkhomedir.so skel=/etc/skel umask=0022
```

## 9. 重啟服務

```bash
sudo systemctl restart smbd nmbd winbind
sudo systemctl enable winbind
```

## 10. 時間同步

```bash
sudo timedatectl set-ntp true
```

## 如果您是 Lightdm 用戶，就像 Mint 的情況一樣，請將其設置為使用網絡用戶登錄，而不僅僅是本地用戶。

## 🛠️ 一步一步：配置 LightDM 以接受域用戶

## 1.編輯LightDM配置文件

```bash
sudo vim /etc/lightdm/lightdm.conf
```

## 編輯文件中的以下行：

```bash
[Seat:*]
greeter-show-manual-login=true
greeter-hide-users=true
allow-guest=false
```

## 說明：

- greeter-show-manual-login=true：允許您手動輸入用戶名。
- greeter-hide-users=true：隱藏本地用戶列表（對於企業環境有用）。
- allow-guest=false：阻止訪客登錄（出於安全考慮）。

## 2. 確保 PAM 允許域用戶

## 如果您使用 SSSD 或 Winbind，PAM 應該已經正確集成。但請確保 home 模塊存在：

```bash
sudo vim /etc/pam.d/common-session
```

## 確認此行存在或添加它：

```bash
session required pam_mkhomedir.so skel=/etc/skel umask=0022
```

## 3. 重新啟動LightDM

```bash
sudo timedatectl set-ntp true
```

## ⚠️ 對於基於 Ubuntu 的 Linux MInt，您可以禁用 systemd-resolved 來手動控制 DNS。

```bash
systemctl status systemd-resolved
```

```bash
systemctl stop systemd-resolved
systemctl disable systemd-resolved.service
sudo systemctl mask systemd-resolved
```

## 刪除由 SYSTEMD-RESOLVED 創建的沒有編輯權限的文件：

```bash
rm -f /etc/resolv.conf
```

## 創建具有編輯權限的新文件：

```bash
vim /etc/resolv.conf
```

```bash
domain educatux.edu
search educatux.edu.
nameserver 192.168.70.253
```

## 鎖定文件以防止自動編輯

```bash
sudo chattr +i /etc/resolv.conf
```

## 服務重啟

```bash
sudo systemctl restart NetworkManager
```

---

🎯 這就是大家！

👉聯繫方式：zerolies@disroot.org
👉 https://t.me/z3r0l135

