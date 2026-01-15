# Linux Mint をドメインに参加させるための LightDM 構成

## 🎯 目的 NSS/PAM 統合を通じて Winbind を使用して Linux を Samba4 ドメインに参加させる

---

## 前提条件

- ドメイン コントローラー (PDC) としての Samba4
- DNS を使用し、PDC と時刻調整された Linux (192.168.70.253)
- サーバーへの接続

---

## 🛠️ 歩数

## 1. 必要なパッケージをインストールする

```bash
sudo apt update && sudo apt install samba winbind libpam-winbind libnss-winbind krb5-user
```

## 2. /etc/krb5.conf を設定する

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

## 3. /etc/samba/smb.conf を設定します。

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

## 4. /etc/nsswitch.conf を構成する

```bash
passwd:         compat winbind
group:          compat winbind
shadow:         compat
```

## 5. ドメインに参加する

```bash
sudo net ads join -U Administrator
```

## 6. システムの再起動

```bash
sudo reboot
```

## 7. チェック

```bash
sudo net ads testjoin
wbinfo -u
wbinfo -g
getent passwd usuario
```

## 8. HOMEディレクトリを自動作成する


## /etc/pam.d/common-session を編集し、以下を追加します。

```bash
session required pam_mkhomedir.so skel=/etc/skel umask=0022
```

## 9. サービスの再起動

```bash
sudo systemctl restart smbd nmbd winbind
sudo systemctl enable winbind
```

## 10. 時刻同期

```bash
sudo timedatectl set-ntp true
```

## Mint の場合と同様に、Lightdm ユーザーの場合は、単なるローカル ユーザーではなくネットワーク ユーザーでログインするように設定します。

## 🛠️ ステップバイステップ: ドメイン ユーザーを受け入れるように LightDM を構成する

## 1. LightDM設定ファイルを編集する

```bash
sudo vim /etc/lightdm/lightdm.conf
```

## ファイル内の次の行を編集します。

```bash
[Seat:*]
greeter-show-manual-login=true
greeter-hide-users=true
allow-guest=false
```

## 説明:

- greeter-show-manual-login=true: ユーザー名を手動で入力できます。
- greeter-hide-users=true: ユーザーのローカル リストを非表示にします (企業環境に役立ちます)。
- allow-guest=false: ゲストのログインを禁止します (セキュリティのため)。

## 2. PAM がドメイン ユーザーを許可していることを確認します。

## SSSD または Winbind を使用した場合、PAM はすでに正しく統合されているはずです。ただし、ホーム モジュールが存在することを確認してください。

```bash
sudo vim /etc/pam.d/common-session
```

## この行が存在することを確認するか、追加します。

```bash
session required pam_mkhomedir.so skel=/etc/skel umask=0022
```

## 3.LightDMを再起動します

```bash
sudo timedatectl set-ntp true
```

## ⚠️ Ubuntu ベースの Linux Mint の場合、systemd-resolved を無効にして DNS を手動で制御できます。

```bash
systemctl status systemd-resolved
```

```bash
systemctl stop systemd-resolved
systemctl disable systemd-resolved.service
sudo systemctl mask systemd-resolved
```

## システム解決によって作成された編集権限なしのファイルの削除:

```bash
rm -f /etc/resolv.conf
```

## 編集権限のある新しいファイルを作成します。

```bash
vim /etc/resolv.conf
```

```bash
domain educatux.edu
search educatux.edu.
nameserver 192.168.70.253
```

## 自動編集に対してファイルをロックする

```bash
sudo chattr +i /etc/resolv.conf
```

## サービスの再開

```bash
sudo systemctl restart NetworkManager
```

---

🎯 以上です!

👉連絡先: zerolies@disroot.org
👉 チリ_REF_0_チリ

