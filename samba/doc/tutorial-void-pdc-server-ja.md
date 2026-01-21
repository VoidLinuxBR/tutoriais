# Void Linux サーバーで Samba4 を実行しているプライマリ ドメイン コントローラー (Active Directory) ;D

## 🎯 目的 - Void Linux (glibc) にプライマリ ドメイン コントローラーをアップロードし、ソース コードから Samba4 をコンパイルし、内部 DNS、Kerberos、AD 統合、ACL、サービス、およびネットワーク クライアントの制御に必要なスタック全体を構成します。

### 🔧 もちろん、チュートリアルをあなたの現実に合わせてください!

## 📡 ローカルのレイアウト

- ドメイン: EDUCATUX.EDU
- ホスト名: pdc01
- ファイアウォール 192.168.70.254 (DNS/GW)
- IP: 192.168.70.253

---

## Void Linux のデフォルトのインストールについては、このチュートリアルでは説明しません。

## インストール後に Void のデフォルトのシェルを変更します。

```bash
chsh -s /bin/bash
```

## 🧩 Void で Samba4 をコンパイルするための依存関係パッケージをインストールする

```bash
xbps-install -S \
 net-tools ldns bind-utils rsync acl attr attr-devel autoconf automake libtool \
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

## ⚠️ 注意: コンパイルされた Samba4 には、デフォルトで組み込み (内部 KDC) である Heimdal Kerberos コードが含まれていますが、Kerberos クライアントは含まれていません。この場合、リポジトリは MIT からバイナリ パッケージを提供します。これは、ドメイン コントローラ上でコンパイルされたデフォルトの kerberos heimdal に問題や干渉を与えることなくインストールできます。パッケージは次のとおりです: mit-krb5 mit-krb5-client mit-krb5-devel。ただし、いかなる状況でも、リポジトリから krb5-server バイナリ パッケージをインストールしてはなりません。Samba4 内部の Heimdal kerberos と競合するサービスが発生する可能性があります。

## MIT-krb5 の顧客が提供するサービスは次のとおりです。

```bash
/usr/bin/kinit
/usr/bin/klist
/usr/bin/kvno
/usr/bin/kdestroy
```

## 🖥️ Setar ホスト名

```bash
echo "pdc01" > /etc/hostname
```

## 🏠 /etc/hosts

```bash
vim /etc/hosts
```

## コンテンツ：

```bash
127.0.0.1      localhost
127.0.1.1      pdc01.educatux.edu pdc01
192.168.70.253 pdc01.educatux.edu pdc01
```

## 🌐 固定IPを構成する

### 👉 Void のデフォルトのメソッド /etc/dhcpcd.conf を使用します。

```bash
vim /etc/dhcpcd.conf
```

## IP、ゲートウェイ、DNS を追加します:## 🎯 目的

```bash
interface eth0
static ip_address=192.168.70.253/24
static routers=192.168.70.254
static domain_name_servers=192.168.70.254
```

## ネットワークインターフェースを再起動します。

```bash
sv restart dhcpcd
```

## 🌐 プロビジョニングの前に一時的な DNS (ルーター) を設定します

```bash
echo "nameserver 192.168.70.254" > /etc/resolv.conf
```

## resolv.conf 構成をロックする

```bash
chattr +i /etc/resolv.conf
```

## 🔍 ネットワークインターフェースに割り当てられたアドレスを検証する

```bash
ip -c addr
```

```bash
ip -br link
```

## 📥 Samba4 ソースコードをダウンロードして解凍します。

```bash
wget https://download.samba.org/pub/samba/samba-4.23.4.tar.gz
```

```bash
tar -xvzf samba-4.23.4.tar.gz
```

## ソースコードをコンパイルしてインストールする

```bash
cd samba-4.23.4
```

```bash
./configure --prefix=/opt/samba
```

```bash
make -j$(nproc) && make install
```

## コメント：

- Samba は /opt/samba でコンパイルされるため、Void はインストールを妨げません。
- Make -j を使用すると、コンパイルが大幅に高速化されます。とにかく、コーヒーを飲みに行きましょう。
- インストール後、コンパイルされた Samba4 には runit 内にサービスが作成されません。
- サービスは手動で作成します。

## 📁 Samba4 をシステム PATH に追加し、環境を再読み込みします

```bash
echo 'export PATH=/opt/samba/bin:/opt/samba/sbin:$PATH' >> /etc/profile
```

```bash
source /etc/profile
```

## オペレーティング システムへの Samba4 PATH の挿入をテストします。

```bash
samba-tool -V
```

## 結果：

```bash
4.23.4
```

🏰 SAMBA4 ドメインのプロビジョニング (PDC 自体の作成)

```bash
samba-tool domain provision \
 --realm=educatux.edu \
 --domain=EDUCATUX \
 --use-rfc2307 \
 --dns-backend=SAMBA_INTERNAL \
 --server-role=dc \
 --adminpass='P@ssw0rd' \
 --option="ad dc functional level = 2016" \
 --function-level=2016
```

### Samba4 は以下を作成します:

```bash
/opt/samba/etc/smb.conf
/opt/samba/private/*
/opt/samba/var/locks/sysvol
```

## Samba4 を要約すると:

- AD フォレスト、プライマリ DC、内部 DNS、アカウント DB を作成します。
- ドメイン、レルム、2016 機能レベル、および管理者パスワードを定義します。
- Void はネイティブ Samba をインストールしないため、競合は発生しません。
- この後、DNS は PDC 自体になり、/etc/resolv.conf を 127.0.0.1 に調整する必要があります。

## ⚙️ Active Directory 2016 の機能レベルを検証する

```bash
samba-tool domain level show
```

## 結果：

```bash
Domain and forest function level for domain 'DC=educatux,DC=edu'
Forest function level: (Windows) 2016
Domain function level: (Windows) 2016
Lowest function level of a DC: (Windows) 2016
```

## 🧪 サービスを作成する前に AD DC サービスを手動でテストします

```bash
/opt/samba/sbin/samba -i -M single
```

* -i → 前景
* -M シングル → 単一プロセス モデル (runit 制御外でデーモン フォークをトリガーしません)

## すべて問題なければ、次のように表示されます。

```bash
Completed DNS update check OK
Completed SPN update check OK
Registered EDUCATUX<1c> ...
```

## CTRL+C を押して終了します

## 📦 起動時に AD をアップロードするための samba-ad-dc RUNIT サービスを作成します

## ⚠️この部分はとても重要です。既存のサーバーを再調整する場合は、古い残りを削除してください。

```bash
sv stop samba-ad-dc 2>/dev/null
rm -rf /var/service/samba-ad-dc
rm -rf /var/service/a-ad-dc
rm -rf /etc/sv/samba-ad-dc
rm -rf /etc/sv/a-ad-dc
rm -rf /etc/sv/*/supervise
rm -rf /var/service/*/supervise
```

## 次に、システム起動時に runit をロードできるように、ログを含む samba-ad-dc サービスと権限を作成しましょう。

## 何よりもまずサービス構造を作成する

```bash
mkdir -p /etc/sv/samba-ad-dc/log
mkdir -p /var/log/samba-ad-dc
```

## 実行サービスを作成する

```bash
cat > /etc/sv/samba-ad-dc/run << 'EOF'
#!/bin/sh
exec 2>&1
exec /opt/samba/sbin/samba -i -M single --debuglevel=3
EOF
```

## サービスの実行権限を設定する

```bash
chmod +x /etc/sv/samba-ad-dc/run
```

## ログファイルを作成する

```bash
cat > /etc/sv/samba-ad-dc/log/run << 'EOF'
#!/bin/sh
exec svlogd -tt /var/log/samba-ad-dc
EOF
```

## ログ/実行権限を設定する

```bash
chmod +x /etc/sv/samba-ad-dc/log/run
```

## 起動時に samba-ad-dc サービスを実行できるようにします。

```bash
ln -sf /etc/sv/samba-ad-dc/ /var/service/
```

## 実行中かどうかを検証する

```bash
sv status samba-ad-dc
```

## 次のようなものが表示されるはずです。

```bash
run: samba-ad-dc: (pid 28032) 4s; run: log: (pid 28031) 4s
```

```bash
samba-tool processes
```

## 受け取った結果:

```bash
 Service:                          PID
--------------------------------------
cldap_server                      1012
dnssrv                            1012
dnsupdate                         1012
dreplsrv                          1012
ft_scanner                        1012
kccsrv                            1012
kdc_server                        1012
ldap_server                       1012
nbt_server                        1012
notify-daemon                     1016
rpc_server                        1012
samba                             1012
winbind_server                    1019
```

## ログをオンラインで検証します。

```bash
tail -f /var/log/samba-ad-dc/current
```

## 正しい出力は次のようになります。

```bash
2025-11-27_04:14:23.73604 Completed DNS update check OK
2025-11-27_04:14:25.35809 Registered pdc01<00> with 192.168.70.253 on interface 192.168.70.255
2025-11-27_04:14:25.35814 Registered pdc01<03> with 192.168.70.253 on interface 192.168.70.255
2025-11-27_04:14:25.35815 Registered pdc01<20> with 192.168.70.253 on interface 192.168.70.255
2025-11-27_04:14:25.35941 Registered EDUCATUX<1b> with 192.168.70.253 on interface 192.168.70.255
2025-11-27_04:14:25.35942 Registered EDUCATUX<1c> with 192.168.70.253 on interface 192.168.70.255
2025-11-27_04:14:25.35944 Registered EDUCATUX<00> with 192.168.70.253 on interface 192.168.70.255
2025-11-27_04:14:36.71381 Calling samba_kcc script
2025-11-27_04:14:37.31554 samba_runcmd_io_handler: Child /opt/samba/sbin/samba_kcc exited 0
2025-11-27_04:14:37.31557 Completed samba_kcc OK
```

## 🕒 NTP / Chronyサーバー

## 5 分の差があると Kerberos がクライアントを認証しなくなるため、ドメイン コントローラーはローカル ネットワークのタイム サーバーである必要があります。

## Chrony サーバー パッケージをインストールする

```bash
xbps-install -Syu chrony
```

## サーバー ファイルを編集し、時刻同期リポジトリを置き換え、内部ネットワーク クエリを解放します。

```bash
vim /etc/chrony.conf
```

### ブラジルのパブリックタイムサーバーを指摘する

```bash
# Comentar a linha do Servidor externo
#pool pool.ntp.org iburst (AQUI)

# Apontar Servidores de tempo BR
server 0.br.pool.ntp.org iburst
server 1.br.pool.ntp.org iburst
server 2.br.pool.ntp.org iburst
server 3.br.pool.ntp.org iburst

# Permitir sincronização da rede interna
allow 192.168.70.0/24
```

## chronyd サービスを RUNIT start に追加します。

```bash
ln -sf /etc/sv/chronyd/ /var/service/
```

## TimeServer を再起動します。

```bash
sv restart chronyd
```

## サーバーを検証します。サーバーはクエリ中に周期的かつランダムです。

```bash
chronyc sources -v
```

## 🔐 Kerberos ファイルを作成する

```bash
vim /etc/krb5.conf
```

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

## 🧭 プロビジョニング後に /etc/resolv.conf のロックを解除してリセットし、PDC 自体をポイントします

```bash
chattr -i /etc/resolv.conf
```

```bash
vim /etc/resolv.conf
```

## コンテンツ：

```bash
domain educatux.edu
search educatux.edu
nameserver 127.0.0.1
```

## ファイルを再度ロックします。

```bash
chattr +i /etc/resolv.conf
```

## 🔗 システム上の Winbind ライブラリをリンクする

## libdir パスを検証します。

```bash
/opt/samba/sbin/smbd -b | grep LIBDIR
```

## 期待される出力:

```bash
LIBDIR: /opt/samba/lib
```

## ライブラリ間のリンクを作成します。ここではコピーして貼り付けるのではなく、手動で入力することをお勧めします。

```bash
ln -sf /opt/samba/lib/libnss_winbind.so.2 /usr/lib/
```

```bash
ln -sf /usr/lib/libnss_winbind.so.2 /usr/lib/libnss_winbind.so
```

## 新しいリンクされたライブラリを使用して構成を再読み込みします

```bash
ldconfig
```

## Kerberos チケット交換の有効性を検証し、nsswhitch の 2 行 (passwd と group) に winbind を追加します。

```bash
vim /etc/nsswitch.conf
```

```bash
passwd: files winbind
group:  files winbind
```

### ファイルの残りの部分はそのまま残ります

## 📝 プロビジョニングによって自動的に作成された smb.conf を検証する

```bash
cat /opt/samba/etc/smb.conf
```

```
# Global parameters
[global]
        ad dc functional level = 2016
        dns forwarder = 192.168.70.254
        netbios name = pdc01
        realm = EDUCATUX.EDU
        server role = active directory domain controller
        workgroup = EDUCATUX
        idmap_ldb:use rfc2307 = yes
        # aponte aos serviços, as interfaces ativas
        interfaces = eth0 
        bind interfaces only = yes

[sysvol]
        path = /opt/samba/var/locks/sysvol
        read only = No

[netlogon]
        path = /opt/samba/var/locks/sysvol/educatux.edu/scripts
        read only = No
```

## 🔍 次に、DNS、SMB、Winbind、Kerberos などの重要な PDC サービスを検証します。

```bash
ps aux | grep samba
```

## 受け取った結果:

```bash
root     28030  0.0  0.0   2392  1388 ?        Ss   01:14   0:00 runsv samba-ad-dc
root     28031  0.0  0.0   2540  1376 ?        S    01:14   0:00 svlogd -tt /var/log/samba-ad-dc
root     28032  0.1  3.3 129656 66884 ?        S    01:14   0:04 samba: root process  .
root     28033  0.0  1.6 129152 33728 ?        S    01:14   0:00 samba: tfork waiter process(28034)
root     28034  0.0  3.3 133112 67156 ?        Ss   01:14   0:00 /opt/samba/sbin/smbd -D --option=server role check:inhibit=yes --foreground
root     28038  0.0  1.6 129152 33432 ?        S    01:14   0:00 samba: tfork waiter process(28039)
root     28039  0.0  3.1 127588 63240 ?        Ss   01:14   0:00 /opt/samba/sbin/winbindd -D --option=server role check:inhibit=yes --foreground
root     28180  0.0  0.1   6696  2556 pts/0    S+   02:10   0:00 grep samba
```

```bash
samba-tool user show administrator
```

## 受け取った結果:

```bash
dn: CN=Administrator,CN=Users,DC=educatux,DC=edu
objectClass: top
objectClass: person
objectClass: organizationalPerson
objectClass: user
cn: Administrator
description: Built-in account for administering the computer/domain
instanceType: 4
whenCreated: 20251127040618.0Z
uSNCreated: 3889
name: Administrator
objectGUID: 732e3aed-f232-427d-9377-5bf7bc79cd8e
userAccountControl: 512
badPwdCount: 0
codePage: 0
countryCode: 0
badPasswordTime: 0
lastLogoff: 0
pwdLastSet: 134086899781242602
primaryGroupID: 513
objectSid: S-1-5-21-294413610-3908852046-3961109876-500
adminCount: 1
accountExpires: 9223372036854775807
sAMAccountName: Administrator
sAMAccountType: 805306368
objectCategory: CN=Person,CN=Schema,CN=Configuration,DC=educatux,DC=edu
isCriticalSystemObject: TRUE
memberOf: CN=Domain Admins,CN=Users,DC=educatux,DC=edu
memberOf: CN=Schema Admins,CN=Users,DC=educatux,DC=edu
memberOf: CN=Enterprise Admins,CN=Users,DC=educatux,DC=edu
memberOf: CN=Group Policy Creator Owners,CN=Users,DC=educatux,DC=edu
memberOf: CN=Administrators,CN=Builtin,DC=educatux,DC=edu
lastLogonTimestamp: 134086916533352620
whenChanged: 20251127043413.0Z
uSNChanged: 4307
lastLogon: 134086917409338150
logonCount: 5
distinguishedName: CN=Administrator,CN=Users,DC=educatux,DC=edu
```

```bash
wbinfo -u
```

## 受け取った結果:

```bash
EDUCATUX\administrator
EDUCATUX\guest
EDUCATUX\krbtgt
```

```bash
wbinfo -g
```

## 受け取った結果:

```bash
EDUCATUX\administrator
EDUCATUX\guest
EDUCATUX\krbtgt
[root@pdc01 samba-4.23.4]# wbinfo -g
EDUCATUX\cert publishers
EDUCATUX\ras and ias servers
EDUCATUX\allowed rodc password replication group
EDUCATUX\denied rodc password replication group
EDUCATUX\dnsadmins
EDUCATUX\enterprise read-only domain controllers
EDUCATUX\domain admins
EDUCATUX\domain users
EDUCATUX\domain guests
EDUCATUX\domain computers
EDUCATUX\domain controllers
EDUCATUX\schema admins
EDUCATUX\enterprise admins
EDUCATUX\group policy creator owners
EDUCATUX\read-only domain controllers
EDUCATUX\protected users
EDUCATUX\dnsupdateproxy
```

```bash
getent group "Domain Admins"
```

## 受け取った結果:

```bash
EDUCATUX\domain admins:x:3000004:
```

```bash
smbclient -L localhost -U Administrator
```

## 受け取った結果:

```bash
Password for [EDUCATUX\Administrator]:

        Sharename       Type      Comment
        ---------       ----      -------
        sysvol          Disk
        netlogon        Disk
        IPC$            IPC       IPC Service (Samba 4.23.4)
SMB1 disabled -- no workgroup available
```

```bash
samba-tool dns zonelist localhost -U administrator
```

## 受け取った結果:

```bash
Password for [EDUCATUX\administrator]:
  2 zone(s) found

  pszZoneName               : educatux.edu
  Flags                            : DNS_RPC_ZONE_DSINTEGRATED DNS_RPC_ZONE_UPDATE_SECURE
  ZoneType                      : DNS_ZONE_TYPE_PRIMARY
  Version                         : 50
  dwDpFlags                    : DNS_DP_AUTOCREATED DNS_DP_DOMAIN_DEFAULT DNS_DP_ENLISTED
  pszDpFqdn                    : DomainDnsZones.educatux.edu

  pszZoneName               : _msdcs.educatux.edu
  Flags                            : DNS_RPC_ZONE_DSINTEGRATED DNS_RPC_ZONE_UPDATE_SECURE
  ZoneType                      : DNS_ZONE_TYPE_PRIMARY
  Version                         : 50
  dwDpFlags                    : DNS_DP_AUTOCREATED DNS_DP_FOREST_DEFAULT DNS_DP_ENLISTED
  pszDpFqdn                    : ForestDnsZones.educatux.edu
```

```bash
samba-tool user show administrator
```

## 受け取った結果:

```bash
dn: CN=Administrator,CN=Users,DC=educatux,DC=edu
objectClass: top
objectClass: person
objectClass: organizationalPerson
objectClass: user
cn: Administrator
description: Built-in account for administering the computer/domain
instanceType: 4
whenCreated: 20251127040618.0Z
uSNCreated: 3889
name: Administrator
objectGUID: 732e3aed-f232-427d-9377-5bf7bc79cd8e
userAccountControl: 512
badPwdCount: 0
codePage: 0
countryCode: 0
badPasswordTime: 0
lastLogoff: 0
pwdLastSet: 134086899781242602
primaryGroupID: 513
objectSid: S-1-5-21-294413610-3908852046-3961109876-500
adminCount: 1
accountExpires: 9223372036854775807
sAMAccountName: Administrator
sAMAccountType: 805306368
objectCategory: CN=Person,CN=Schema,CN=Configuration,DC=educatux,DC=edu
isCriticalSystemObject: TRUE
memberOf: CN=Domain Admins,CN=Users,DC=educatux,DC=edu
memberOf: CN=Schema Admins,CN=Users,DC=educatux,DC=edu
memberOf: CN=Enterprise Admins,CN=Users,DC=educatux,DC=edu
memberOf: CN=Group Policy Creator Owners,CN=Users,DC=educatux,DC=edu
memberOf: CN=Administrators,CN=Builtin,DC=educatux,DC=edu
lastLogonTimestamp: 134086916533352620
whenChanged: 20251127043413.0Z
uSNChanged: 4307
lastLogon: 134086917409338150
logonCount: 5
distinguishedName: CN=Administrator,CN=Users,DC=educatux,DC=edu
```

## 🔐 ドメイン ユーザーのパスワードの複雑さを無効にする (ラボ テストを容易にする - 運用環境では安全ではありません!)

```bash
samba-tool domain passwordsettings set --complexity=off
samba-tool domain passwordsettings set --history-length=0
samba-tool domain passwordsettings set --min-pwd-length=0
samba-tool domain passwordsettings set --min-pwd-age=0
samba-tool user setexpiry Administrator --noexpiry
```

## Samba4の設定を再読み込みする

```bash
smbcontrol all reload-config
```

## 🧪 Kerberos チケット交換を検証する

```bash
kinit Administrator@EDUCATUX.EDU
```

```bash
klist
```

## 受け取った結果:

```bash
Ticket cache: FILE:/tmp/krb5cc_0
Default principal: Administrator@EDUCATUX.EDU

Valid starting       Expires              Service principal
27/11/2025 02:22:52  27/11/2025 12:22:52  krbtgt/EDUCATUX.EDU@EDUCATUX.EDU
        renew until 28/11/2025 02:22:47
```

```bash
samba-tool dns query pdc01.educatux.edu educatux.edu @ A -U Administrator
```

## 受け取った結果:

```bash
Password for [EDUCATUX\Administrator]:

  Name=, Records=1, Children=0
    A: 192.168.70.253 (flags=600000f0, serial=1, ttl=900)
  Name=_msdcs, Records=0, Children=0
  Name=_sites, Records=0, Children=1
  Name=_tcp, Records=0, Children=4
  Name=_udp, Records=0, Children=2
  Name=DomainDnsZones, Records=0, Children=2
  Name=ForestDnsZones, Records=0, Children=2
  Name=pdc01, Records=1, Children=0
    A: 192.168.70.253 (flags=f0, serial=1, ttl=900)
```

```bash
drill google.com @192.168.70.253
```

## 得られた結果:

```bash
;; ->>HEADER<<- opcode: QUERY, rcode: NOERROR, id: 50285
;; flags: qr rd ra ; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0
;; QUESTION SECTION:
;; google.com.  IN      A

;; ANSWER SECTION:
google.com.     300     IN      A       172.217.30.142

;; AUTHORITY SECTION:

;; ADDITIONAL SECTION:

;; Query time: 224 msec
;; EDNS: version 0; flags: ; udp: 1232
;; SERVER: 192.168.70.253
;; WHEN: Thu Nov 27 02:30:42 2025
;; MSG SIZE  rcvd: 55
```

```bash
samba_dnsupdate --verbose
```

```bash
IPs: ['192.168.70.253']
Looking for DNS entry A pdc01.educatux.edu 192.168.70.253 as pdc01.educatux.edu.
Looking for DNS entry CNAME a9126dd4-c5ad-46b4-b91b-6ae91313e3b8._msdcs.educatux.edu pdc01.educatux.edu as a9126dd4-c5ad-46b4-b91b-6ae91313e3b8._msdcs.educatux.edu.
Looking for DNS entry NS educatux.edu pdc01.educatux.edu as educatux.edu.
Looking for DNS entry NS _msdcs.educatux.edu pdc01.educatux.edu as _msdcs.educatux.edu.
Looking for DNS entry A educatux.edu 192.168.70.253 as educatux.edu.
Looking for DNS entry SRV _ldap._tcp.educatux.edu pdc01.educatux.edu 389 as _ldap._tcp.educatux.edu.
Checking 0 100 389 pdc01.educatux.edu. against SRV _ldap._tcp.educatux.edu pdc01.educatux.edu 389
Looking for DNS entry SRV _ldap._tcp.dc._msdcs.educatux.edu pdc01.educatux.edu 389 as _ldap._tcp.dc._msdcs.educatux.edu.
Checking 0 100 389 pdc01.educatux.edu. against SRV _ldap._tcp.dc._msdcs.educatux.edu pdc01.educatux.edu 389
Looking for DNS entry SRV _ldap._tcp.f5cccdab-a9d9-4b1f-9344-d2affb3c9855.domains._msdcs.educatux.edu pdc01.educatux.edu 389 as _ldap._tcp.f5cccdab-a9d9-4b1f-9344-d2affb3c9855.domains._msdcs.educatux.edu.
Checking 0 100 389 pdc01.educatux.edu. against SRV _ldap._tcp.f5cccdab-a9d9-4b1f-9344-d2affb3c9855.domains._msdcs.educatux.edu pdc01.educatux.edu 389
Looking for DNS entry SRV _kerberos._tcp.educatux.edu pdc01.educatux.edu 88 as _kerberos._tcp.educatux.edu.
Checking 0 100 88 pdc01.educatux.edu. against SRV _kerberos._tcp.educatux.edu pdc01.educatux.edu 88
Looking for DNS entry SRV _kerberos._udp.educatux.edu pdc01.educatux.edu 88 as _kerberos._udp.educatux.edu.
Checking 0 100 88 pdc01.educatux.edu. against SRV _kerberos._udp.educatux.edu pdc01.educatux.edu 88
Looking for DNS entry SRV _kerberos._tcp.dc._msdcs.educatux.edu pdc01.educatux.edu 88 as _kerberos._tcp.dc._msdcs.educatux.edu.
Checking 0 100 88 pdc01.educatux.edu. against SRV _kerberos._tcp.dc._msdcs.educatux.edu pdc01.educatux.edu 88
Looking for DNS entry SRV _kpasswd._tcp.educatux.edu pdc01.educatux.edu 464 as _kpasswd._tcp.educatux.edu.
Checking 0 100 464 pdc01.educatux.edu. against SRV _kpasswd._tcp.educatux.edu pdc01.educatux.edu 464
Looking for DNS entry SRV _kpasswd._udp.educatux.edu pdc01.educatux.edu 464 as _kpasswd._udp.educatux.edu.
Checking 0 100 464 pdc01.educatux.edu. against SRV _kpasswd._udp.educatux.edu pdc01.educatux.edu 464
Looking for DNS entry SRV _ldap._tcp.Default-First-Site-Name._sites.educatux.edu pdc01.educatux.edu 389 as _ldap._tcp.Default-First-Site-Name._sites.educatux.edu.
Checking 0 100 389 pdc01.educatux.edu. against SRV _ldap._tcp.Default-First-Site-Name._sites.educatux.edu pdc01.educatux.edu 389
Looking for DNS entry SRV _ldap._tcp.Default-First-Site-Name._sites.dc._msdcs.educatux.edu pdc01.educatux.edu 389 as _ldap._tcp.Default-First-Site-Name._sites.dc._msdcs.educatux.edu.
Checking 0 100 389 pdc01.educatux.edu. against SRV _ldap._tcp.Default-First-Site-Name._sites.dc._msdcs.educatux.edu pdc01.educatux.edu 389
Looking for DNS entry SRV _kerberos._tcp.Default-First-Site-Name._sites.educatux.edu pdc01.educatux.edu 88 as _kerberos._tcp.Default-First-Site-Name._sites.educatux.edu.
Checking 0 100 88 pdc01.educatux.edu. against SRV _kerberos._tcp.Default-First-Site-Name._sites.educatux.edu pdc01.educatux.edu 88
Looking for DNS entry SRV _kerberos._tcp.Default-First-Site-Name._sites.dc._msdcs.educatux.edu pdc01.educatux.edu 88 as _kerberos._tcp.Default-First-Site-Name._sites.dc._msdcs.educatux.edu.
Checking 0 100 88 pdc01.educatux.edu. against SRV _kerberos._tcp.Default-First-Site-Name._sites.dc._msdcs.educatux.edu pdc01.educatux.edu 88
Looking for DNS entry SRV _ldap._tcp.pdc._msdcs.educatux.edu pdc01.educatux.edu 389 as _ldap._tcp.pdc._msdcs.educatux.edu.
Checking 0 100 389 pdc01.educatux.edu. against SRV _ldap._tcp.pdc._msdcs.educatux.edu pdc01.educatux.edu 389
Looking for DNS entry A gc._msdcs.educatux.edu 192.168.70.253 as gc._msdcs.educatux.edu.
Looking for DNS entry SRV _gc._tcp.educatux.edu pdc01.educatux.edu 3268 as _gc._tcp.educatux.edu.
Checking 0 100 3268 pdc01.educatux.edu. against SRV _gc._tcp.educatux.edu pdc01.educatux.edu 3268
Looking for DNS entry SRV _ldap._tcp.gc._msdcs.educatux.edu pdc01.educatux.edu 3268 as _ldap._tcp.gc._msdcs.educatux.edu.
Checking 0 100 3268 pdc01.educatux.edu. against SRV _ldap._tcp.gc._msdcs.educatux.edu pdc01.educatux.edu 3268
Looking for DNS entry SRV _gc._tcp.Default-First-Site-Name._sites.educatux.edu pdc01.educatux.edu 3268 as _gc._tcp.Default-First-Site-Name._sites.educatux.edu.
Checking 0 100 3268 pdc01.educatux.edu. against SRV _gc._tcp.Default-First-Site-Name._sites.educatux.edu pdc01.educatux.edu 3268
Looking for DNS entry SRV _ldap._tcp.Default-First-Site-Name._sites.gc._msdcs.educatux.edu pdc01.educatux.edu 3268 as _ldap._tcp.Default-First-Site-Name._sites.gc._msdcs.educatux.edu.
Checking 0 100 3268 pdc01.educatux.edu. against SRV _ldap._tcp.Default-First-Site-Name._sites.gc._msdcs.educatux.edu pdc01.educatux.edu 3268
Looking for DNS entry A DomainDnsZones.educatux.edu 192.168.70.253 as DomainDnsZones.educatux.edu.
Looking for DNS entry SRV _ldap._tcp.DomainDnsZones.educatux.edu pdc01.educatux.edu 389 as _ldap._tcp.DomainDnsZones.educatux.edu.
Checking 0 100 389 pdc01.educatux.edu. against SRV _ldap._tcp.DomainDnsZones.educatux.edu pdc01.educatux.edu 389
Looking for DNS entry SRV _ldap._tcp.Default-First-Site-Name._sites.DomainDnsZones.educatux.edu pdc01.educatux.edu 389 as _ldap._tcp.Default-First-Site-Name._sites.DomainDnsZones.educatux.edu.
Checking 0 100 389 pdc01.educatux.edu. against SRV _ldap._tcp.Default-First-Site-Name._sites.DomainDnsZones.educatux.edu pdc01.educatux.edu 389
Looking for DNS entry A ForestDnsZones.educatux.edu 192.168.70.253 as ForestDnsZones.educatux.edu.
Looking for DNS entry SRV _ldap._tcp.ForestDnsZones.educatux.edu pdc01.educatux.edu 389 as _ldap._tcp.ForestDnsZones.educatux.edu.
Checking 0 100 389 pdc01.educatux.edu. against SRV _ldap._tcp.ForestDnsZones.educatux.edu pdc01.educatux.edu 389
Looking for DNS entry SRV _ldap._tcp.Default-First-Site-Name._sites.ForestDnsZones.educatux.edu pdc01.educatux.edu 389 as _ldap._tcp.Default-First-Site-Name._sites.ForestDnsZones.educatux.edu.
Checking 0 100 389 pdc01.educatux.edu. against SRV _ldap._tcp.Default-First-Site-Name._sites.ForestDnsZones.educatux.edu pdc01.educatux.edu 389
No DNS updates needed
```

### ✅ 最終概要

## 🎉 おめでとうございます — これで、完全に機能する 2016 AD ドメインが Void Linux 上にセットアップされました。

### 👉 覚えておいてください: Samba4 はコマンド ラインで管理できるにもかかわらず、リモート サーバー管理ツール - RSAT で管理できるように設計されており、Windows 10 マシンに問題なくインストールできます。

## 次のことができるようになりました。

- Windows マシンをドメインに参加させる
- GPO を使用する
- テストレプリケーション (DC2 作成時)
- RSAT 経由でユーザー/グループを作成する
- sysvol レプリケーションを構成します (Rsync または新しい samba-gpupdate を使用)
- DNSフォワーダーを追加する
- DFSを有効にする
- メンバーファイルサーバーの作成
- 等

---

🎯 以上です!

👉連絡先: zerolies@disroot.org
👉 チリ_REF_0_チリ
