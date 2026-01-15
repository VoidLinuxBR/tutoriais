
# Void Linux サーバー上で Samba4 を実行しているファイル サーバー ;D

## 🎯 目標 – ソース、AD 統合、ACL、サービス、およびファイル サーバーがネットワーク クライアントにサービスを提供するために必要なスタック全体から Samba4 をコンパイルすることにより、Void Linux (glibc) にファイル サーバーをデプロイします。

## 🔧 QEMU/Virtmanager および Proxmox を使用したネットワーキング ラボ。自分の環境に合わせてチュートリアルを調整してください。

---

## 📡 ローカルネットワークのレイアウト

- ドメイン: EDUCATUX.EDU

- ホスト名: ファイルサーバー

- ファイアウォール 192.168.70.254 (DNS/GW)

- IP: 192.168.70.251

## Void Linux をインストールする

## Void のデフォルトのシェルを変更する

```bash
chsh -s /bin/bash
```

## 🧩 Void で Samba4 をコンパイルするための依存関係パッケージをインストールする

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

## 🖥️ ホスト名を設定する

```bash
echo "fileserver" > /etc/hostname
```

## 🏠 /etc/hosts

```bash
vim /etc/hosts
```

## コンテンツ：

```bash
127.0.0.1      localhost
127.0.1.1      fileserver.educatux.edu fileserver
192.168.70.251 fileserver.educatux.edu fileserver
```

## 🌐 静的 IP を構成する

## 👉 標準の Void メソッド、/etc/dhcpcd.conf を使用します。

```bash
vim /etc/dhcpcd.conf
```

## IP、ゲートウェイ (ルーター)、および DNS (AD) を追加します。

```bash
interface eth0
static ip_address=192.168.70.251/24
static routers=192.168.70.254
static domain_name_servers=192.168.70.253
```

## ネットワークインターフェースを再起動します。

```bash
sv restart dhcpcd
```

## 🧭 DNS アドレスを設定 - PDC をポイントします

```bash
vim /etc/resolv.conf
```

## コンテンツ：

```bash
domain educatux.edu
search educatux.edu
nameserver 192.168.70.253
```

## resolv.conf をロックする

```bash
chattr +i /etc/resolv.conf
```

## 🔍 ネットワークインターフェースアドレスを検証する

```bash
ip -c addr
ip -br link
```

## 📥 Samba4 ソースコードをダウンロードして抽出する

```bash
wget https://download.samba.org/pub/samba/samba-4.23.4.tar.gz
```

```bash
tar -xvzf samba-4.23.4.tar.gz
```

## ソースからコンパイルしてインストールする

```bash
cd samba-4.23.4
```

```bash
./configure --prefix=/opt/samba
```

```bash
make -j$(nproc) && make install
```

## 注:

- Samba は /opt/samba にコンパイルされるため、Void は干渉しません。

- make -j を使用すると、コンパイルが大幅に高速化されます。それでも、コーヒーを飲みに行きましょう。

- インストール後、コンパイルされた Samba4 には runit サービスがありません。

- サービスは手動で作成します。

## 📁 Samba4をシステムPATHに追加し、環境をリロードします

```bash
echo 'export PATH=/opt/samba/bin:/opt/samba/sbin:$PATH' >> /etc/profile
```

```bash
source /etc/profile
```

## OS への Samba4 PATH の挿入をテストする

```bash
samba-tool -V
```

## 出力：

```bash
4.23.4
```

## ⚠️ 警告: ファイル サーバーでプロビジョニング コマンドを使用しないでください。

## 📝 smb.conf ファイルを作成する

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

## ログファイルを作成する

```bash
mkdir /opt/samba/var
```

## 📂 共有パスを作成する

```bash
sudo mkdir -p /srv/samba/public
sudo chown -R root:root /srv/samba/public
sudo chmod -R 0770 /srv/samba/public
```

## Samba4 設定をリロードする

```bash
smbcontrol all reload-config
```

## 🕒 NTP / Chronyサーバー

## 5 分のずれでは Kerberos がクライアントを認証しなくなるため、ドメイン コントローラーはローカル タイム サーバーである必要があります。

## Chronyをインストールする

```bash
xbps-install -Syu chrony
```

## 設定を編集して内部ネットワークを許可します

```bash
vim /etc/chrony.conf
```

## タイムサーバーでドメイン制御を設定します。

```bash
# Comment the external line
#pool pool.ntp.org iburst

# PDC Time Servers
server 192.168.70.253 iburst
```

## runit で chronyd を有効にする

```bash
ln -sf /etc/sv/chronyd/ /var/service/
```

## サービスを再起動します:

```bash
sv restart chronyd
```

## サーバーを検証します。

```bash
chronyc sources -v
```

## 🔐 Kerberos ファイルを作成します - PDC を指定します

```bash
vim /etc/krb5.conf
```

## 含む

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

## ケルベロステスト

```bash
kinit Administrator
```

```bash
klist
```

## 得られた結果

```bash
Ticket cache: FILE:/tmp/krb5cc_0
Default principal: administrator@EDUCATUX.EDU

Valid starting       Expires              Service principal
11/12/2025 09:43:40  11/12/2025 19:43:40  krbtgt/EDUCATUX.EDU@EDUCATUX.EDU
	renew until 12/12/2025 09:43:36
```

## 🔗 Winbind ライブラリをシステムにリンクする

## libdir パスを検証します。

```bash
/opt/samba/sbin/smbd -b | grep LIBDIR
```

## 期待される：

```bash
LIBDIR: /opt/samba/lib
```

## ライブラリ リンクを作成します (手動で入力することをお勧めします)。

```bash
ln -sf /opt/samba/lib/libnss_winbind.so.2 /usr/lib/
```

```bash
ln -sf /usr/lib/libnss_winbind.so.2 /usr/lib/libnss_winbind.so
```

## ライブラリ キャッシュをリロードします。

```bash
ldconfig
```

## Kerberos チケット交換用に nsswitch を更新します (winbind を追加):

```bash
vim /etc/nsswitch.conf
```

```bash
passwd: files winbind
group:  files winbind
```

## 残りはそのままにしておきます

## ドメインに参加する

```bash
net ads join -U Administrator
```

## 得られた結果

```bash
Password for [EDUCATUX\Administrator]:
Using short domain name -- EDUCATUX
Joined 'VOIDFILES' to dns domain 'educatux.edu'
```

## 📦 smbd、winbindd、およびオプションで nmbd の RUNIT サービスを作成します

### Void Linux は init システムとして runit を使用し、そのネイティブ ロガーは runit 基本パッケージに含まれる svlogd です。追加のパッケージは必要ありません。

## SMBD — サービスとロギング

## サービスとログのディレクトリを作成する

```bash
mkdir -p /etc/sv/smbd/log
mkdir -p /var/log/smbd
```

## /etc/sv/smbd/run を作成します

```bash
vim /etc/sv/smbd/run
```

## コンテンツ

```bash
#!/bin/sh
exec 2>&1
exec /opt/samba/sbin/smbd --foreground --no-process-group
```

## 許可

```bash
chmod +x /etc/sv/smbd/run
```

## /etc/sv/smbd/log/run を作成します

```bash
vim /etc/sv/smbd/log/run
```

## コンテンツ

```bash
#!/bin/sh
exec svlogd -tt /var/log/smbd
```

## 許可

```bash
chmod +x /etc/sv/smbd/log/run
```

## デバッグ (オプション)

```bash
/opt/samba/sbin/smbd -i
```

## 得られた結果

```bash
smbd version 4.23.4 started.
Copyright Andrew Tridgell and the Samba Team 1992-2025
daemon 'smbd' : Starting process ...
```

## WINBINDD — サービスとロギング

## サービスとログのディレクトリを作成する

```bash
mkdir -p /etc/sv/winbindd/log
mkdir -p /var/log/winbindd
```

## /etc/sv/winbindd/run を作成します

```bash
vim /etc/sv/winbindd/run
```

## コンテンツ

```bash
#!/bin/sh
exec 2>&1
exec /opt/samba/sbin/winbindd --foreground --no-process-group
```

## 許可

```bash
chmod +x /etc/sv/winbindd/run
```

## /etc/sv/winbindd/log/run を作成します。

```bash
vim /etc/sv/winbindd/log/run
```

## コンテンツ

```bash
#!/bin/sh
exec 2>&1
exec /opt/samba/sbin/winbindd --foreground --no-process-group
```

## 許可

```bash
chmod +x /etc/sv/winbindd/log/run
```

## デバッグ (オプション)

```bash
/opt/samba/sbin/winbindd -i
```

## 得られた結果

```bash
winbindd version 4.23.4 started.
Copyright Andrew Tridgell and the Samba Team 1992-2025
daemon 'winbindd' : Starting process ...
```

## NMBD — サービスとロギング (オプション)

### お使いの環境で NetBIOS/SMB1 の参照が必要な場合にのみ有効にします。

## サービスとログのディレクトリを作成する

```bash
mkdir -p /etc/sv/nmbd/log
mkdir -p /var/log/nmbd
```

## /etc/sv/nmbd/run を作成します

```bash
vim /etc/sv/nmbd/run
```

## コンテンツ

```bash
#!/bin/sh
exec 2>&1
exec /opt/samba/sbin/nmbd --foreground --no-process-group
```

## 許可

```bash
chmod +x /etc/sv/nmbd/run
```

## /etc/sv/nmbd/log/run を作成します

```bash
vim /etc/sv/nmbd/log/run
```
 
## コンテンツ

```bash
#!/bin/sh
exec svlogd -tt /var/log/nmbd
```

## 許可

```bash
chmod +x /etc/sv/nmbd/log/run
```

## サービスを有効にする

```bash
ln -sf /etc/sv/smbd /var/service/
ln -sf /etc/sv/winbindd /var/service/
```

## オプション - NetBIOS を使用する場合にのみ有効にします。

```bash
ln -sf /etc/sv/nmbd /var/service/
```

## サービスの検証

```bash
sv status smbd winbindd nmbd
```

## 🧪 統合を検証する

```bash
net ads testjoin
```

## 得られた結果

```bash
Join is OK
```

```bash
wbinfo -u
```

## 得られた結果

```bash
guest
krbtgt
administrator```

```bash
wbinfo -g
```

## Result obtained

```bash
エンタープライズ読み取り専用ドメイン コントローラー
保護されたユーザー
ドメインコントローラー
ドメインゲスト
読み取り専用ドメイン コントローラー
スキーマ管理者
dnsupdateproxy
ドメイン管理者
グループポリシー作成者の所有者
ras および ias サーバー
dnsadmins
許可された Rodc パスワード レプリケーション グループ
エンタープライズ管理者
証明書発行者
ドメインユーザー
Rodc パスワード複製グループが拒否されました
ドメインコンピュータ
```

```bash
wbinfo --ping-dc
```

## Result obtained

```bash
Domain[EDUCATUX] の NETLOGON をチェックして、「voiddc.educatux.edu」への DC 接続が成功しました。
```

## ✅ FINAL SUMMARY

## 🎉 Congratulations — you have successfully deployed a fully functional File Server on Void Linux!

## 👉 REMEMBER: While Samba4 can be managed via CLI, it was designed to be managed via RSAT Remote Server Administration Tools, which can be installed on a Windows 10 machine without issues!

---

🎯 THAT'S ALL FOLKS!

👉 Contact: zerolies@disroot.org

👉 https://t.me/z3r0l135
