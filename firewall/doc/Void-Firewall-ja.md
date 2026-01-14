#  🧩 VOID LINUX チュートリアル — セキュリティ スキームの実装 – ラボラトリー ワークショップ

📌 IP Público のファイアウォール、Void Linux (glibc)、IPTables (レガシー)、NAT、ポート ノッキング、Fail2ban

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
  conntrack-tools\
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
chmod +x /etc/sv/firewall/run
ln -s /etc/sv/firewall /var/service/
sv status firewall
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
cat /proc/net/xt_recent/SSH_KNOCK
```

期待される結果

```bash
src=99.336.74.209 ttl: 61 last_seen: 4302299386 oldest_pkt: 7 4302292227, 4302293242, 4302294266, 4302295290, 4302296314, 4302297338, 4302299386
```

すべての障害をクリアしたい場合

```bash
echo clear > /proc/net/xt_recent/SSH_KNOCK
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
sudo vim .bashrc
```

コンテンツ

```bash
alias knock='nc -z 39.236.83.109 12345'
alias officinas='ssh -p 2222 supertux@39.236.83.109'
```

検証のためにファイルを再読み込みします

```bash
source .bashrc
```

11. ✅ FAIL2BAN – ノック後の保護

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

12. ✅ FAIL2BAN テスト (外部アクセス中はロックアウトされるので注意)

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

13. 🎉 チェックリスト最終版

- ノックなしの目に見えない SSH
- 使い捨てノック
- 短いアクセスウィンドウ
- Fail2ban アクティブな認証後
- ノック無視禁止
- 機能的NAT
- 永続的なファイアウォール
- Proxmox はトンネル経由でのみアクセス可能

---

🎯 以上です!
👉 チリ_REF_0_チリ
👉 チリ_REF_0_チリ
















































































