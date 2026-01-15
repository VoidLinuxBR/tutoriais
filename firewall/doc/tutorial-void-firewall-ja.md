# 🧩 VOID LINUX チュートリアル - ファイアウォールの実装 - EDUCATUX LABORATORY

📌 パブリック IP を備えたファイアウォール、Void Linux (glibc)、IPTables (レガシー)、NAT、ポート ノッキング、Fail2ban、DHCP サーバー、再帰的 DNS

---

## ✅ 1. ネットワークトポロジ

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

別の角度から見る

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

ファイアウォールは、インターネットに公開される唯一のホストです。

## ✅ 2. 目的と前提

- デフォルトポリシーを拒否する
- アクティブなIPv4ルーティング
- スキャナーはドアを決して認識しません
- 唯一のエントリポイントとしてのファイアウォール
- Web ダッシュボードは公開されていません
- ポートノッキングで保護されたSSH
- Fail2banによるブルートフォース制御
- LAN 用の制御された NAT
- SSHトンネル経由のリモート管理

## ✅ 3. 必要なパッケージを更新してインストールする

システムをアップデートする

```bash
sudo xbps-install -Syu
```

パッケージをインストールする

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

## ✅ 4. SSH 設定

```bash
sudo vim /etc/ssh/sshd_config
```

尖った線を調整する

```bash
Port 2222
ListenAddress 0.0.0.0

PermitRootLogin yes        # TEMPORÁRIO (remover após hardening)
PasswordAuthentication yes
UsePAM no

SyslogFacility AUTH
LogLevel INFO
```

Fail2ban はログに依存し、ラインを保証します

```bash
SyslogFacility AUTH
LogLevel INFO
```

ログの生成を確認する

```bash
sudo tail -f /var/log/auth.log
```

## サービスのアクティベーション

```bash
sudo ln -s /etc/sv/sshd /var/service/
sudo sv start sshd
```

## 完全な展開後:

- rootログインを無効にする

- キー認証のみを使用する

## ✅ 5. ファイアウォールネットワークのセットアップ

```bash
sudo vim /etc/dhcpcd.conf
```

コンテンツ

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

適用する

```bash
sudo sv restart dhcpcd
```

## ✅ 6. ポートノッキング – カーネルサポート

必要なモジュールをロードする

```bash
sudo modprobe xt_recent
```

検証:

```bash
sudo lsmod | grep xt_recent
```

期待される結果

```bash
xt_recent              24576  0
x_tables               65536  1 xt_recent
```

## ✅ 7. ファイアウォール IP テーブル

ファイアウォールネットワークカード間のルーティングを有効にする

```bash
sudo vim /etc/sysctl.conf
```

コンテンツ

```bash
net.ipv4.ip_forward=1
```

再起動せずに適用します。

```bash
sudo sysctl --system
```

/usr/local/bin にファイアウォール スクリプトを作成します。

```bash
sudo vim /usr/local/bin/firewall
```

コンテンツ

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

権限を適用して実行する

```bash
sudo chmod +x /usr/local/bin/firewall
sudo bash /usr/local/bin/firewall
```

## ✅ 8. 実行中のファイアウォールの永続性

ディレクトリを作成する

```bash
sudo mkdir -p /etc/sv/firewall
```

ファイルを作成する

```bash
sudo vim /etc/sv/firewall/run
```

コンテンツ

```bash
#!/bin/sh
exec /usr/local/bin/firewall
```

アクティブ化、実行、ステータスの検証

```bash
sudo chmod +x /etc/sv/firewall/run
sudo ln -s /etc/sv/firewall /var/service/
sudo sv status firewall
```

## ✅ 9. ポートノッキングのテストと検証 (ホット)

ファイアウォールなしで端末のノックを監視する

```bash
sudo tcpdump -ni eth0 tcp port 12345
```

外部アクセス経由でノートブックでノックを送信

```bash
sudo nc -z 39.236.83.109 12345
```

✔ SYNが到着
✔ 削除されました
✔ 登録を続ける
✔ステータスが見える

tcpdump で期待される結果

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

重要な技術上の注意事項

- RST は TCP スタック経由で送信されます
- パッケージは xt_recent によって登録されます
- ポートがサービスとして応答しない
- バナーや指紋はありません

IP登録の検証

```bash
sudo cat /proc/net/xt_recent/SSH_KNOCK
```

期待される結果

```bash
src=99.336.74.209 ttl: 61 last_seen: 4302299386 oldest_pkt: 7 4302292227, 4302293242, 4302294266, 4302295290, 4302296314, 4302297338, 4302299386
```

すべての障害をクリアしたい場合

```bash
sudo echo clear > /proc/net/xt_recent/SSH_KNOCK
```

## ✅ 10. 外部管理アクセスの実行

ノックを実行する

```bash
nc -z 39.236.83.109 12345
```

15秒以内にアクセスしてください

```bash
ssh -p 2222 supertux@39.236.83.109
```

推奨されるエイリアス

```bash
vim ~/.bashrc
```

コンテンツ

```bash
alias knock='nc -z 39.236.83.109 12345'
alias firewall='ssh -p 2222 supertux@39.236.83.109'
```

検証のためにファイルを再読み込みします

```bash
source ~/.bashrc
```

11. ✅ FAIL2BAN – ノック後の保護

フェイル2バンに準拠するためのログ調整

```bash
sudo xbps-install -y socklog-void
sudo ln -s /etc/sv/socklog-unix /var/service/
sudo ln -s /etc/sv/nanoklogd /var/service/
sudo touch /var/log/auth.log
```

設定ファイルを作成します (jail.conf は編集しないでください)

```bash
sudo vim /etc/fail2ban/jail.local
```

コンテンツ：

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

Runitのアクティベーション

```bash
sudo ln -s /etc/sv/fail2ban /var/service/
sudo sv start fail2ban
sudo sv status fail2ban
```

## 12. ✅ 2BAN テストに失敗する (注意、自分自身をロックアウトしてしまいます!)

オノックを実行する

```bash
nc -z 39.236.83.109 12345
```

間違ったパスワードで SSH を 3 回試行してください

禁止事項を確認してください

```bash
sudo fail2ban-client status sshd
```

手動で禁止を解除する:

```bash
sudo fail2ban-client set sshd unbanip X.X.X.X
```

## ⚠️ 注意: 再帰 DNS と DHCP サーバーを扱う次のセクション 13 と 14 は、SAMBA4 を PDC としてアップグレードした後は破棄する必要があります。

## 13. ✅ 内部ネットワークにサービスを提供するための一時的な再帰 DNS の展開

```bash
sudo xbps-install -y unbound
```

最小構成

```bash
sudo vim /etc/unbound/unbound.conf
```

コンテンツ

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

サービス (runit) をアクティブ化します。

```bash
ln -s /etc/sv/unbound /var/service/
sv start unbound
```

## 14. ✅ 内部ネットワークにサービスを提供するための一時的な DHCP サーバーの実装

パッケージのインストール

```bash
sudo xbps-install -y dhcp
```

このパッケージは以下をインストールします:

- dhcpd (サーバー)
- Runit サービスの構造:
/etc/sv/dhcpd4
/etc/sv/dhcpd6

ファイルを編集し、内部ネットワークの設定を構成します

```bash
sudo vim /etc/dhcpd.conf
```

コンテンツ

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

リース ファイルを作成します。

```bash
sudo mkdir -p /var/lib/dhcp
sudo touch /var/lib/dhcp/dhcpd.leases
```

Runitサービスの作成

```bash
sudo vim /etc/sv/dhcpd4/conf
```

コンテンツ

```bash
OPTS="-4 -q -cf /etc/dhcpd.conf eth1"
```

説明：

- -4 → IPv4
- -q → サイレントモード
- -cf → 正しい dhcpd.conf パス
- eth1 → インターフェースLAN

runit でサービスをアクティブ化します。

```bash
sudo ln -s /etc/sv/dhcpd4 /var/service/
```

開始/再起動:

```bash
sudo sv restart dhcpd4
```

ステータスを確認します:

```bash
sudo sv status dhcpd4
```

期待される結果:

```bash
run: dhcpd4: (pid 17652) 831s; run: log: (pid 15544) 1213s
```

ポート67のリスニングを確認してください

```bash
UNCONN 0      0            0.0.0.0:67        0.0.0.0:*    users:(("dhcpd",pid=17652,fd=6))  
```

DHCPをリアルタイムで監視する

```bash
sudo tcpdump -ni eth1 port 67 or port 68
```

期待される結果

```bash
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth1, link-type EN10MB (Ethernet), snapshot length 262144 bytes
```

直接デバッグ用 (runit なし)

```bash
sudo dhcpd -4 -d -cf /etc/dhcpd.conf eth1
```

これで表示されるはずです

- DHCPディスカバー
- DHCPOFFER
- DHCPリクエスト
- DHCPACK

重要なファイル

- /etc/dhcpd.conf → 主な設定
- /var/lib/dhcp/dhcpd.leases → リース
- /etc/sv/dhcpd4/run → スクリプト runit
- /etc/sv/dhcpd4/conf → サービスパラメータ
- /var/service/dhcpd4 → サービスがアクティブです

LAN 上で DHCP を許可するように iptables スクリプトを調整します。暗黙的な DROP ルールの前に追加します。

```bash
# ============================
# DHCP LAN
# ============================

iptables -A INPUT  -i $LAN -p udp --sport 67:68 --dport 67:68 -j ACCEPT
iptables -A OUTPUT -o $LAN -p udp --sport 67:68 --dport 67:68 -j ACCEPT
```

💡 DHCP はブロードキャストを使用します → これがないと、クライアントは IP を取得できません。

ファイアウォールを再適用します。

```bash
sudo /usr/local/bin/firewall
```

LAN VM でのテスト

```bash
dhclient -v
```

ファイアウォール内で監視

```bash
sudo tail -f /var/log/messages
```

または

```bash
sudo tcpdump -ni eth1 port 67 or port 68
```

## 15. 🎉 最終チェックリスト

- ノックなしの目に見えない SSH
- 使い捨てノック
- 短いアクセスウィンドウ
- Fail2ban アクティブな認証後
- ノック無視禁止
- 機能的NAT
- 永続的なファイアウォール
- Proxmox はトンネル経由でのみアクセス可能
- 最小限の再帰 DNS (PDC が入るまで)
- DHCPサーバー

---

🎯 以上です!

👉 チリ_REF_0_チリ
👉 チリ_REF_0_チリ
