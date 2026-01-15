
# 在 Void Linux 服務器上運行 Samba4 的文件服務器；D

## 🎯 目標 – 通過從源代碼編譯 Samba4、AD 集成、ACL、服務以及文件服務器為網絡客戶端提供服務所需的整個堆棧，在 Void Linux (glibc) 上部署文件服務器。

## 🔧 使用 QEMU/Virtmanager 和 Proxmox 的網絡實驗室。調整教程以適合您自己的環境。

---

## 📡 本地網絡佈局

- 域名：EDUCATUX.EDU

- 主機名：文件服務器

- 防火牆 192.168.70.254 (DNS/GW)

- IP：192.168.70.251

## 安裝Void Linux

## 更改 Void 上的默認 shell

```bash
chsh -s /bin/bash
```

## 🧩 在Void上安裝編譯Samba4的依賴包

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

## 🖥️ 設置主機名

```bash
echo "fileserver" > /etc/hostname
```

## 🏠 /etc/hosts

```bash
vim /etc/hosts
```

## 內容：

```bash
127.0.0.1      localhost
127.0.1.1      fileserver.educatux.edu fileserver
192.168.70.251 fileserver.educatux.edu fileserver
```

## 🌐配置靜態IP

## 👉我們將使用標準的Void方法，/etc/dhcpcd.conf

```bash
vim /etc/dhcpcd.conf
```

## 添加IP、網關（路由器）和DNS（AD）：

```bash
interface eth0
static ip_address=192.168.70.251/24
static routers=192.168.70.254
static domain_name_servers=192.168.70.253
```

## 重新啟動網絡接口：

```bash
sv restart dhcpcd
```

## 🧭 設置 DNS 地址 - 指向 PDC

```bash
vim /etc/resolv.conf
```

## 內容：

```bash
domain educatux.edu
search educatux.edu
nameserver 192.168.70.253
```

## 鎖定resolv.conf

```bash
chattr +i /etc/resolv.conf
```

## 🔍 驗證網絡接口地址

```bash
ip -c addr
ip -br link
```

## 📥 下載並解壓Samba4源代碼

```bash
wget https://download.samba.org/pub/samba/samba-4.23.4.tar.gz
```

```bash
tar -xvzf samba-4.23.4.tar.gz
```

## 從源代碼編譯並安裝

```bash
cd samba-4.23.4
```

```bash
./configure --prefix=/opt/samba
```

```bash
make -j$(nproc) && make install
```

## 筆記：

- void 不會干擾，因為 Samba 被編譯到 /opt/samba 中。

- make -j 大大加快了編譯速度——不過，去喝杯咖啡吧。

- 安裝後，編譯後的Samba4沒有任何runit服務。

- 我們將手動創建服務。

## 📁 將 Samba4 添加到系統 PATH 並重新加載環境

```bash
echo 'export PATH=/opt/samba/bin:/opt/samba/sbin:$PATH' >> /etc/profile
```

```bash
source /etc/profile
```

## 測試 Samba4 PATH 插入操作系統

```bash
samba-tool -V
```

## 輸出：

```bash
4.23.4
```

## ⚠️警告：請勿在文件服務器上使用配置命令！

## 📝 創建 smb.conf 文件

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

## 創建日誌文件

```bash
mkdir /opt/samba/var
```

## 📂 創建共享路徑

```bash
sudo mkdir -p /srv/samba/public
sudo chown -R root:root /srv/samba/public
sudo chmod -R 0770 /srv/samba/public
```

## 重新加載 Samba4 配置

```bash
smbcontrol all reload-config
```

## 🕒 NTP/Chrony 服務器

## 域控制器必須是本地時間服務器，因為漂移 5 分鐘後，Kerberos 將不再對客戶端進行身份驗證。

## 安裝 Chrony

```bash
xbps-install -Syu chrony
```

## 編輯配置並允許內部網絡

```bash
vim /etc/chrony.conf
```

## 在時間服務器上設置域控制：

```bash
# Comment the external line
#pool pool.ntp.org iburst

# PDC Time Servers
server 192.168.70.253 iburst
```

## 在runit中啟用chronyd

```bash
ln -sf /etc/sv/chronyd/ /var/service/
```

## 重啟服務：

```bash
sv restart chronyd
```

## 驗證服務器：

```bash
chronyc sources -v
```

## 🔐 創建 Kerberos 文件 - 指向 PDC

```bash
vim /etc/krb5.conf
```

## 含有

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

## Kerberos測試

```bash
kinit Administrator
```

```bash
klist
```

## 得到的結果

```bash
Ticket cache: FILE:/tmp/krb5cc_0
Default principal: administrator@EDUCATUX.EDU

Valid starting       Expires              Service principal
11/12/2025 09:43:40  11/12/2025 19:43:40  krbtgt/EDUCATUX.EDU@EDUCATUX.EDU
	renew until 12/12/2025 09:43:36
```

## 🔗 將 Winbind 庫鏈接到系統

## 驗證 libdir 路徑：

```bash
/opt/samba/sbin/smbd -b | grep LIBDIR
```

## 預期的：

```bash
LIBDIR: /opt/samba/lib
```

## 創建庫鏈接（最好手動輸入）：

```bash
ln -sf /opt/samba/lib/libnss_winbind.so.2 /usr/lib/
```

```bash
ln -sf /usr/lib/libnss_winbind.so.2 /usr/lib/libnss_winbind.so
```

## 重新加載庫緩存：

```bash
ldconfig
```

## 更新 nsswitch 以進行 Kerberos 票證交換（添加 winbind）：

```bash
vim /etc/nsswitch.conf
```

```bash
passwd: files winbind
group:  files winbind
```

## 其餘部分保持不變

## 加入域

```bash
net ads join -U Administrator
```

## 得到的結果

```bash
Password for [EDUCATUX\Administrator]:
Using short domain name -- EDUCATUX
Joined 'VOIDFILES' to dns domain 'educatux.edu'
```

## 📦 為 smbd、winbindd 和可選的 nmbd 創建 RUNIT 服務

### Void Linux使用runit作為init系統，其原生記錄器是svlogd，包含在runit基礎包中。不需要額外的包。

## SMBD——服務和日誌記錄

## 創建服務和日誌目錄

```bash
mkdir -p /etc/sv/smbd/log
mkdir -p /var/log/smbd
```

## 創建/etc/sv/smbd/run

```bash
vim /etc/sv/smbd/run
```

## 內容

```bash
#!/bin/sh
exec 2>&1
exec /opt/samba/sbin/smbd --foreground --no-process-group
```

## 允許

```bash
chmod +x /etc/sv/smbd/run
```

## 創建/etc/sv/smbd/log/run

```bash
vim /etc/sv/smbd/log/run
```

## 內容

```bash
#!/bin/sh
exec svlogd -tt /var/log/smbd
```

## 允許

```bash
chmod +x /etc/sv/smbd/log/run
```

## 調試（可選）

```bash
/opt/samba/sbin/smbd -i
```

## 得到的結果

```bash
smbd version 4.23.4 started.
Copyright Andrew Tridgell and the Samba Team 1992-2025
daemon 'smbd' : Starting process ...
```

## WINBIDD — 服務和日誌記錄

## 創建服務和日誌目錄

```bash
mkdir -p /etc/sv/winbindd/log
mkdir -p /var/log/winbindd
```

## 創建 /etc/sv/winbindd/run

```bash
vim /etc/sv/winbindd/run
```

## 內容

```bash
#!/bin/sh
exec 2>&1
exec /opt/samba/sbin/winbindd --foreground --no-process-group
```

## 允許

```bash
chmod +x /etc/sv/winbindd/run
```

## 創建 /etc/sv/winbindd/log/run

```bash
vim /etc/sv/winbindd/log/run
```

## 內容

```bash
#!/bin/sh
exec 2>&1
exec /opt/samba/sbin/winbindd --foreground --no-process-group
```

## 允許

```bash
chmod +x /etc/sv/winbindd/log/run
```

## 調試（可選）

```bash
/opt/samba/sbin/winbindd -i
```

## 得到的結果

```bash
winbindd version 4.23.4 started.
Copyright Andrew Tridgell and the Samba Team 1992-2025
daemon 'winbindd' : Starting process ...
```

## NMBD — 服務和日誌記錄（可選）

### 僅當您的環境需要 NetBIOS/SMB1 瀏覽時才啟用。

## 創建服務和日誌目錄

```bash
mkdir -p /etc/sv/nmbd/log
mkdir -p /var/log/nmbd
```

## 創建/etc/sv/nmbd/run

```bash
vim /etc/sv/nmbd/run
```

## 內容

```bash
#!/bin/sh
exec 2>&1
exec /opt/samba/sbin/nmbd --foreground --no-process-group
```

## 允許

```bash
chmod +x /etc/sv/nmbd/run
```

## 創建/etc/sv/nmbd/log/run

```bash
vim /etc/sv/nmbd/log/run
```
 
## 內容

```bash
#!/bin/sh
exec svlogd -tt /var/log/nmbd
```

## 允許

```bash
chmod +x /etc/sv/nmbd/log/run
```

## 啟用服務

```bash
ln -sf /etc/sv/smbd /var/service/
ln -sf /etc/sv/winbindd /var/service/
```

## 可選 - 僅在使用 NetBIOS 時啟用：

```bash
ln -sf /etc/sv/nmbd /var/service/
```

## 驗證服務

```bash
sv status smbd winbindd nmbd
```

## 🧪 驗證集成

```bash
net ads testjoin
```

## 得到的結果

```bash
Join is OK
```

```bash
wbinfo -u
```

## 得到的結果

```bash
guest
krbtgt
administrator```

```bash
wbinfo-g
```

## Result obtained

```bash
企業只讀域控制器
受保護的用戶
域控制器
域來賓
只讀域控制器
架構管理員
域名更新代理
域管理員
組策略創建者所有者
RAS 和 IAS 服務器
域名管理員
允許的 rodc 密碼複製組
企業管理員
證書發布者
域用戶
拒絕 rodc 密碼複製組
域計算機
```

```bash
wbinfo --ping-dc
```

## Result obtained

```bash
檢查 NETLOGON 的域 [EDUCATUX] dc 連接到“voiddc.educatux.edu”成功
```

## ✅ FINAL SUMMARY

## 🎉 Congratulations — you have successfully deployed a fully functional File Server on Void Linux!

## 👉 REMEMBER: While Samba4 can be managed via CLI, it was designed to be managed via RSAT Remote Server Administration Tools, which can be installed on a Windows 10 machine without issues!

---

🎯 THAT'S ALL FOLKS!

👉 Contact: zerolies@disroot.org

👉 https://t.me/z3r0l135
