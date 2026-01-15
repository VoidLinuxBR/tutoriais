
# 在 Void Linux 服务器上运行 Samba4 的文件服务器；D

## 🎯 目标 – 通过从源代码编译 Samba4、AD 集成、ACL、服务以及文件服务器为网络客户端提供服务所需的整个堆栈，在 Void Linux (glibc) 上部署文件服务器。

## 🔧 使用 QEMU/Virtmanager 和 Proxmox 的网络实验室。调整教程以适合您自己的环境。

---

## 📡 本地网络布局

- 域名：EDUCATUX.EDU

- 主机名：文件服务器

- 防火墙 192.168.70.254 (DNS/GW)

- IP：192.168.70.251

## 安装Void Linux

## 更改 Void 上的默认 shell

```bash
chsh -s /bin/bash
```

## 🧩 在Void上安装编译Samba4的依赖包

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

## 🖥️ 设置主机名

```bash
echo "fileserver" > /etc/hostname
```

## 🏠 /etc/hosts

```bash
vim /etc/hosts
```

## 内容：

```bash
127.0.0.1      localhost
127.0.1.1      fileserver.educatux.edu fileserver
192.168.70.251 fileserver.educatux.edu fileserver
```

## 🌐配置静态IP

## 👉我们将使用标准的Void方法，/etc/dhcpcd.conf

```bash
vim /etc/dhcpcd.conf
```

## 添加IP、网关（路由器）和DNS（AD）：

```bash
interface eth0
static ip_address=192.168.70.251/24
static routers=192.168.70.254
static domain_name_servers=192.168.70.253
```

## 重新启动网络接口：

```bash
sv restart dhcpcd
```

## 🧭 设置 DNS 地址 - 指向 PDC

```bash
vim /etc/resolv.conf
```

## 内容：

```bash
domain educatux.edu
search educatux.edu
nameserver 192.168.70.253
```

## 锁定resolv.conf

```bash
chattr +i /etc/resolv.conf
```

## 🔍 验证网络接口地址

```bash
ip -c addr
ip -br link
```

## 📥 下载并解压Samba4源代码

```bash
wget https://download.samba.org/pub/samba/samba-4.23.4.tar.gz
```

```bash
tar -xvzf samba-4.23.4.tar.gz
```

## 从源代码编译并安装

```bash
cd samba-4.23.4
```

```bash
./configure --prefix=/opt/samba
```

```bash
make -j$(nproc) && make install
```

## 笔记：

- void 不会干扰，因为 Samba 被编译到 /opt/samba 中。

- make -j 大大加快了编译速度——不过，去喝杯咖啡吧。

- 安装后，编译后的Samba4没有任何runit服务。

- 我们将手动创建服务。

## 📁 将 Samba4 添加到系统 PATH 并重新加载环境

```bash
echo 'export PATH=/opt/samba/bin:/opt/samba/sbin:$PATH' >> /etc/profile
```

```bash
source /etc/profile
```

## 测试 Samba4 PATH 插入操作系统

```bash
samba-tool -V
```

## 输出：

```bash
4.23.4
```

## ⚠️警告：请勿在文件服务器上使用配置命令！

## 📝 创建 smb.conf 文件

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

## 创建日志文件

```bash
mkdir /opt/samba/var
```

## 📂 创建共享路径

```bash
sudo mkdir -p /srv/samba/public
sudo chown -R root:root /srv/samba/public
sudo chmod -R 0770 /srv/samba/public
```

## 重新加载 Samba4 配置

```bash
smbcontrol all reload-config
```

## 🕒 NTP/Chrony 服务器

## 域控制器必须是本地时间服务器，因为漂移 5 分钟后，Kerberos 将不再对客户端进行身份验证。

## 安装 Chrony

```bash
xbps-install -Syu chrony
```

## 编辑配置并允许内部网络

```bash
vim /etc/chrony.conf
```

## 在时间服务器上设置域控制：

```bash
# Comment the external line
#pool pool.ntp.org iburst

# PDC Time Servers
server 192.168.70.253 iburst
```

## 在runit中启用chronyd

```bash
ln -sf /etc/sv/chronyd/ /var/service/
```

## 重启服务：

```bash
sv restart chronyd
```

## 验证服务器：

```bash
chronyc sources -v
```

## 🔐 创建 Kerberos 文件 - 指向 PDC

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

## Kerberos测试

```bash
kinit Administrator
```

```bash
klist
```

## 得到的结果

```bash
Ticket cache: FILE:/tmp/krb5cc_0
Default principal: administrator@EDUCATUX.EDU

Valid starting       Expires              Service principal
11/12/2025 09:43:40  11/12/2025 19:43:40  krbtgt/EDUCATUX.EDU@EDUCATUX.EDU
	renew until 12/12/2025 09:43:36
```

## 🔗 将 Winbind 库链接到系统

## 验证 libdir 路径：

```bash
/opt/samba/sbin/smbd -b | grep LIBDIR
```

## 预期的：

```bash
LIBDIR: /opt/samba/lib
```

## 创建库链接（最好手动输入）：

```bash
ln -sf /opt/samba/lib/libnss_winbind.so.2 /usr/lib/
```

```bash
ln -sf /usr/lib/libnss_winbind.so.2 /usr/lib/libnss_winbind.so
```

## 重新加载库缓存：

```bash
ldconfig
```

## 更新 nsswitch 以进行 Kerberos 票证交换（添加 winbind）：

```bash
vim /etc/nsswitch.conf
```

```bash
passwd: files winbind
group:  files winbind
```

## 其余部分保持不变

## 加入域

```bash
net ads join -U Administrator
```

## 得到的结果

```bash
Password for [EDUCATUX\Administrator]:
Using short domain name -- EDUCATUX
Joined 'VOIDFILES' to dns domain 'educatux.edu'
```

## 📦 为 smbd、winbindd 和可选的 nmbd 创建 RUNIT 服务

### Void Linux使用runit作为init系统，其原生记录器是svlogd，包含在runit基础包中。不需要额外的包。

## SMBD——服务和日志记录

## 创建服务和日志目录

```bash
mkdir -p /etc/sv/smbd/log
mkdir -p /var/log/smbd
```

## 创建/etc/sv/smbd/run

```bash
vim /etc/sv/smbd/run
```

## 内容

```bash
#!/bin/sh
exec 2>&1
exec /opt/samba/sbin/smbd --foreground --no-process-group
```

## 允许

```bash
chmod +x /etc/sv/smbd/run
```

## 创建/etc/sv/smbd/log/run

```bash
vim /etc/sv/smbd/log/run
```

## 内容

```bash
#!/bin/sh
exec svlogd -tt /var/log/smbd
```

## 允许

```bash
chmod +x /etc/sv/smbd/log/run
```

## 调试（可选）

```bash
/opt/samba/sbin/smbd -i
```

## 得到的结果

```bash
smbd version 4.23.4 started.
Copyright Andrew Tridgell and the Samba Team 1992-2025
daemon 'smbd' : Starting process ...
```

## WINBIDD — 服务和日志记录

## 创建服务和日志目录

```bash
mkdir -p /etc/sv/winbindd/log
mkdir -p /var/log/winbindd
```

## 创建 /etc/sv/winbindd/run

```bash
vim /etc/sv/winbindd/run
```

## 内容

```bash
#!/bin/sh
exec 2>&1
exec /opt/samba/sbin/winbindd --foreground --no-process-group
```

## 允许

```bash
chmod +x /etc/sv/winbindd/run
```

## 创建 /etc/sv/winbindd/log/run

```bash
vim /etc/sv/winbindd/log/run
```

## 内容

```bash
#!/bin/sh
exec 2>&1
exec /opt/samba/sbin/winbindd --foreground --no-process-group
```

## 允许

```bash
chmod +x /etc/sv/winbindd/log/run
```

## 调试（可选）

```bash
/opt/samba/sbin/winbindd -i
```

## 得到的结果

```bash
winbindd version 4.23.4 started.
Copyright Andrew Tridgell and the Samba Team 1992-2025
daemon 'winbindd' : Starting process ...
```

## NMBD — 服务和日志记录（可选）

### 仅当您的环境需要 NetBIOS/SMB1 浏览时才启用。

## 创建服务和日志目录

```bash
mkdir -p /etc/sv/nmbd/log
mkdir -p /var/log/nmbd
```

## 创建/etc/sv/nmbd/run

```bash
vim /etc/sv/nmbd/run
```

## 内容

```bash
#!/bin/sh
exec 2>&1
exec /opt/samba/sbin/nmbd --foreground --no-process-group
```

## 允许

```bash
chmod +x /etc/sv/nmbd/run
```

## 创建/etc/sv/nmbd/log/run

```bash
vim /etc/sv/nmbd/log/run
```
 
## 内容

```bash
#!/bin/sh
exec svlogd -tt /var/log/nmbd
```

## 允许

```bash
chmod +x /etc/sv/nmbd/log/run
```

## 启用服务

```bash
ln -sf /etc/sv/smbd /var/service/
ln -sf /etc/sv/winbindd /var/service/
```

## 可选 - 仅在使用 NetBIOS 时启用：

```bash
ln -sf /etc/sv/nmbd /var/service/
```

## 验证服务

```bash
sv status smbd winbindd nmbd
```

## 🧪 验证集成

```bash
net ads testjoin
```

## 得到的结果

```bash
Join is OK
```

```bash
wbinfo -u
```

## 得到的结果

```bash
guest
krbtgt
administrator```

```bash
wbinfo-g
```

## Result obtained

```bash
企业只读域控制器
受保护的用户
域控制器
域来宾
只读域控制器
架构管理员
域名更新代理
域管理员
组策略创建者所有者
RAS 和 IAS 服务器
域名管理员
允许的 rodc 密码复制组
企业管理员
证书发布者
域用户
拒绝 rodc 密码复制组
域计算机
```

```bash
wbinfo --ping-dc
```

## Result obtained

```bash
检查 NETLOGON 的域 [EDUCATUX] dc 连接到“voiddc.educatux.edu”成功
```

## ✅ FINAL SUMMARY

## 🎉 Congratulations — you have successfully deployed a fully functional File Server on Void Linux!

## 👉 REMEMBER: While Samba4 can be managed via CLI, it was designed to be managed via RSAT Remote Server Administration Tools, which can be installed on a Windows 10 machine without issues!

---

🎯 THAT'S ALL FOLKS!

👉 Contact: zerolies@disroot.org

👉 https://t.me/z3r0l135
