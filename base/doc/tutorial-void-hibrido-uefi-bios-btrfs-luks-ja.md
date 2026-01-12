# 🧩 VOID LINUX チュートリアル — EXT4、XFS、JFS または BTRFS (サブボリューム)、LUKS、休止状態、ZRAM を使用したハイブリッド インストール (UEFI + BIOS)
### 改訂および検証されたバージョン — 正しいパーティショニング + ユニバーサル ブート

このガイドでは、古いマシン、新しいマシン、問題のあるマシンなど、あらゆるタイプのマシンを起動できる完全 **ハイブリッド** Void Linux をインストールします。

- 💾 **最新の UEFI** (通常の入力とフォールバックを使用)
- 🧮 **BIOS/レガシー** (完全な互換性)
- 🧰 **BIOS ブート付き GPT (EF02)** — 古いハードウェアを最大限にサポート
- 🚀 **サブボリュームを含む Btrfs** (オプション)、既製のスナップショット
- 🔐 **LUKS1 は GRUB と完全互換**
- 🌙 **スワップファイルによる実際の休止状態**
- 🧊 **パフォーマンスを考慮した ZRAM 構成**
- 🧱 **EXT4、XFS、JFS、BTRFS の完全サポート**
- 💡 **Initramfs/GRUB は自動的に構成されます (LUKS + 再開)**

📌 **妥協、GRUB の再インストール、時間を無駄にする必要はありません。**
📌 **NVRAM が消去されたマシンでもブートが保証されます (BOOTX64.EFI フォールバック)。**

---

# ▶️ 1. Bootar または Live ISO

提案: 優れた互換性を得るには、glibc バージョンを使用してください。
- ISO を次からダウンロードします。
```
https://repo-default.voidlinux.org/live/current/void-live-x86_64-20250202-base.iso
```
- または、次の場所で最新バージョンを探します。
```
https://voidlinux.org/download/
```

1. root としてログインします。
```bash
ログイン: root
パスワード: voidlinux
```

2. シェルを *sh* から *bash* に切り替えます。
*dash/sh* **多くのスクリプトで使用されるいくつかの機能は実装されていません**。
```bash
バッシュ
```

3. キーボード レイアウトを **ABNT2** に変更し、アクセントと記号が正しくマッピングされるようにします。
```bash
ロードキー br-abnt2
```

4. ターミナルに貼り付けます (オプション) — 色、user@host:path、および最後のコマンドのステータス (✔/✘) をプロンプトに表示します。便利で美しい。
```bash
エクスポート PS1='\[\e[1;32m\]\u\[\e[1;33m\]@\[\e[1;36m\]\h\[\e[1;31m\]:\w \
$([[ $? -eq 0 ]] && echo -e "\e[1;32m✔" || echo -e "\e[1;31m✘$?") \
\[\e[0m\]\$ '
```

# ▶️ 2. インターネットに接続する
- **Wi-Fi** の場合 *(ケーブル接続の場合は、この手順をスキップしてください)*:
```bash
wpa_パスフレーズ「SSID」「パスワード」 > wifi.conf
wpa_supplicant -B -i wlan0 -c wifi.conf
dhcpcd wlan0
```

1. 接続をテストします。
```bash
ping -c3 8.8.8.8
ping -c3 リポデフォルト.voidlinux.org
```

2. 必要なパッケージをインストールします。
⚠️ **重要:**
```bash
xbps-install -Sy xbps parted jfsutils xfsprogs nano zstd xz bash-completion
```
---

# ▶️ 3. ディスクを特定する
1. 使用可能なディスクをリストし、デバイス名をメモします (例: `/dev/sda`、`/dev/vda`、`/dev/nvme0n1`)。
```bash
fdisk -l | grep -E '^(ディスク|ディスク) '
```

# ▶️ 4. チュートリアルで使用する変数を定義します。
⚠️ **重要:**

1. 使用する **前** にデバイスを定義します。
> 1. **チュートリアルでは `/dev/sda` (通常) または `/dev/nvme0n1` (nvme) を想定します。
> 2. ディスクに応じて **調整** (**1 つ** または **別の** モデルを選択してください)

**通常** ディスクの場合 (例: /dev/sda)
```bash
エクスポート DEVICE=/dev/sda
エクスポート DEV_BIOS=${DEVICE}1
エクスポート DEV_EFI=${DEVICE}2
エクスポート DEV_ROOT=${DEVICE}3
エクスポート DEV_LUKS=/dev/mapper/cryptroot
```
**NVMe** ディスク (例: /dev/nvme0n1) の場合、パーティションのサフィックス (`p`) が変更されます。
```bash
エクスポート DEVICE=/dev/nvme0n1
エクスポート DEV_BIOS=${DEVICE}p1
エクスポート DEV_EFI=${DEVICE}p2
エクスポート DEV_RAIZ=${DEVICE}p3
エクスポート DEV_LUKS=/dev/mapper/cryptroot
```

> 📌 **注:**
> デバイス → ディスク全体
DEV_BIOS → BIOS ブート パーティション (1 ～ 2 MiB、FS なし、マウントされません)
DEV_EFI → EFI パーティション (FAT32)
DEV_ROOT → ルート パーティション (通常または LUKS)
DEV_LUKS → LUKS マッピング (/dev/mapper/cryptroot)

- 👉 ここで椎間板の構造を定義します。ガイド内の他のすべての内容は、これらの変数に従うだけです。
- 🔎なぜこれが必要なのでしょうか?
最初にすべてを宣言すると、次のプロセスでタイプミスが防止されるためです。

2. **KEYMAP** と **TIMEZONE** を定義します (必要に応じて変更します)。
```bash
エクスポート KEYMAP=br-abnt2
```
```bash
エクスポート TIMEZONE=アメリカ/サンパウロ
```

---

# ▶️ 5. ディスクのパーティション分割
> - BIOS パーティションは **必ず**最初に指定してください。これにより、古いマザーボード、問題のあるブートローダー、ディスクの最初の領域にブート コードが必要な BIOS との互換性が向上します。
> - ESP は問題なく後から来ることができます — UEFI は順位を気にしません。

### 理想的で正しい順序:

- 1️⃣ BIOS ブート (EF02)
- 2️⃣ ESP (EFI システム、FAT32)
- 3️⃣ Btrfs/Ext4/Xfs/Jfs (ルート)

### parted を使用したパーティション (自動)
> ここでは **DEVICE** がすでに定義されているため、「魔法の」変数はありません。
```
winefs -a "${DEVICE}"
分割 --script "${DEVICE}" -- \
mklabel gpt \
mkpart プライマリ 1MiB 2MiB 名 1 BIOS セット \ 上の 1 bios_grub
mkpart プライマリ fat32 2MiB 514MiB 名前 2 EFI セット 2 esp on \
mkpart プライマリ 514MiB 100% 名前 3 ROOT \
最適な整列チェック 1 \
最適な整列チェック 2 \
最適な整列チェック 1
parted --script "${DEVICE}" -- print
```
> - パーティション 1 → BIOS ブート (bios_grub、FS なし、マウントなし)
> - パーティション 2 → EFI (FAT32)
> - パーティション 3 → ROOT (LUKS の有無にかかわらず、後で EXT4/XFS/JFS/BTRFS でフォーマットします)
> - FS の拘束を避けるために、FS を正確に指定せずに mkpart プライマリ 514MiB を 100% 使用しました。後で FS を選択します。
---

# ▶️ 6. インストールモードを選択します (NORMAL または LUKS)
⚠️ **重要:**
> 以下の 2 つのブロックのうち 1 つだけを選択してください。
両方のステップを実行することはできません**。

1. 通常のインストール **(LUKS なし)**
```bash
エクスポート DISK="${DEV_RAIZ}"
```
- DISKを実デバイス/dev/sda3に設定します。

2. **LUKS を使用したインストール** (暗号化されたルート)
```
# LUKS1 (GRUB 互換) のルート パーティションのみを暗号化します。ディスク全体は暗号化しないでください。
# YES を確認してパーティションを暗号化します。
cryptsetup luksFormat --type luks1 "${DEV_RAIZ}"

# パスフレーズを使用してパーティションを開きます。
cryptsetup オープン "${DEV_RAIZ}" cryptroot

# これ以降、実際のルートはマップされたデバイスになります
エクスポート DISK="${DEV_LUKS}"
```
- LUKS はディスク全体ではなく /dev/sda3 の上にあります
- システムは /dev/mapper/cryptroot にインストールされます

👉 ここからは、すべてに $DISK. が使用されます

---

# ▶️ 7. ファイルシステム (FS) を作成し、root をマウントします
⚠️ **重要:**
> 以下の 2 つのブロックのうち 1 つだけを選択してください。

1. **EXT4**
```
mkfs.ext4 -F "${DISK}" -L ROOT
mount -v "${DISK}" /mnt
```
2. **XFS**
```
mkfs.xfs -f "${DISK}"
mount -v "${DISK}" /mnt
```
3. **JFS**
```
mkfs.jfs -f "${DISK}"
mount -v "${DISK}" /mnt
```
4. **単純な BTRFS**
```
mkfs.btrfs -f "${DISK}" -L ROOT
mount -v "${DISK}" /mnt
```
5. **サブボリュームのある BTRFS**
```
mkfs.btrfs -f "${DISK}" -L ROOT

マウント ${DISK} /mnt
btrfs サブボリューム作成 /mnt/@
btrfs サブボリューム作成 /mnt/@home
btrfs サブボリューム作成 /mnt/@log
btrfs サブボリューム作成 /mnt/@cache
btrfs サブボリューム作成 /mnt/@snapshots
アンマウント /mnt

mount -o デフォルト、noatime、ssd、compress=zstd:3、discard=async、space_cache=v2、commit=300、subvol=/@ ${DISK} /mnt
mkdir -p /mnt/{boot/efi,home,var/log,var/cache,.snapshots,swap}

mount -o デフォルト、noatime、ssd、compress=zstd:3、discard=async、space_cache=v2、commit=300、subvol=/@home ${DISK} /mnt/home
mount -o デフォルト、noatime、ssd、compress=zstd:3、discard=async、space_cache=v2、commit=300、subvol=/@cache ${DISK} /mnt/var/cache
mount -o デフォルト、noatime、ssd、compress=zstd:3、discard=async、space_cache=v2、commit=300、subvol=/@log ${DISK} /mnt/var/log
mount -o デフォルト、noatime、ssd、compress=zstd:3、discard=async、space_cache=v2、commit=300、subvol=/@snapshots ${DISK} /mnt/.snapshots
```
---

# ▶️ 8. ESP (EFI) の準備と組み立て
```
mkfs.fat -F32 -I "チリ_REF_0_チリ"
mkdir -p /mnt/boot/efi
mount -v "${DEV_EFI}" /mnt/boot/efi
```
>💡 BIOS パーティション (${DEV_BIOS}) にはファイル システムがなく、フォーマットもマウントも行われません。
---

# ▶️ 9. インストール o Void Linux なし chroot

1. chroot (/mnt) で使用するリポジトリ キー (XBPS キー) をコピーします。
```
mkdir -p /A{tc, 蒸気/xbps/xbps)
cp -rpaf /var/db/xbps/keys/*.plist /mnt/var/db/xbps/keys/
cp -fpa /etc/resolv.conf /mnt/etc/resolv.conf
```

2. 新しくマウントされたディスクに基本システムをインストールします。
```
xbps-install -Sy -R https://repo-default.voidlinux.org/current \
-r /mnt \
ベースシステム btrfs-progs cryptsetup grub grub-x86_64-efi dracut linux \
linux-headers linux-firmware linux-firmware-network glibc-locales \
xtools dhcpcd openssh vim nano grc zstd xz bash-completion vpm vsv \
socklog-void wget net-tools tmate ncurses jfsutils xfsprogs duf Tree eza chrony
```
---

# ▶️ 10. Gerar fstab no /mnt (chroot)
```bash
# /mnt/etc/fstab に fstab を生成
xgenfstab -U /mnt > /mnt/etc/fstab
```

```bash
# 正しく生成されたか確認する
猫/mnt/etc/fstab
```

# ▶️ 11. chroot を使用して、インストールされたシステムにアクセスします

1. 単なる雇用ではなく、
```
xchroot /mnt /bin/bash
```
---

# ▶️ 12. 初期設定(chroot内)
```
#configure hostname - ホスト名を定義します
echo void > /etc/ホスト名

#configure timezone - タイムゾーンを定義します
ln -sfv /usr/share/zoneinfo/"${TIMEZONE}" /etc/localtime

# ロケールを設定する
-i -e 's/^#\(en_.Utf-8 UTF-8\)/)/' \
-E 's/^#\pt_br.br.utf-8 UTF-8\)/' \' \
/etc/default/libc-locales

# ロケールを生成する
xbps-reconfigure -f glibc-locales

# /var/service シンボリックリンクで考えられるエラーを修正します (重要):
rm -f /var/サービス
ln -sf /etc/runit/runsvdir/default /var/service

# いくつかのサービスを有効にする
ln -sf /etc/sv/dbus /var/service/
ln -sf /etc/sv/dhcpcd /var/service/
ln -sf /etc/sv/sshd /var/service/
ln -sf /etc/sv/nanoklogd /var/service/
ln -sf /etc/sv/socklog-unix /var/service/
ln -sf /etc/sv/chronyd /var/service/

# sudo - Wheel グループを設定します (オプションですが推奨)
cat << 'EOF' > /etc/sudoers.d/g_wheel
%wヒール ALL=(ALL:ALL) NOPASSWD: ALL
終了後
#必要な権限
chmod 440 /etc/sudoers.d/g_wheel
```

## ユーザーの作成と構成

⚠️ **重要:** 以下に実際のユーザー名を定義します。
```bash
NEWUSER=your_user_here をエクスポート
```

ホーム ディレクトリ、基本グループ、および Bash シェルを使用してユーザーを作成します。
```bash
useradd -m -G オーディオ、ビデオ、ホイール、tty -s /bin/bash ${NEWUSER}
```

ユーザーパスワードを設定します (***重要***)
```bash
パスワード ${NEWUSER}
```

root ユーザーのパスワードを設定します (***重要***)
```bash
パスワードルート
```

root ユーザーのデフォルトのシェルを Bash に変更する
```bash
chsh -s /bin/bash root
```
---

# ▶️ 13. UUID の構成
⚠️ **重要:**
- パーティションの UUID を取得します。
```
UUID_LUKS=$(blkid -s UUID -o 値 "${DEV_RAIZ}")
UUID_ROOT=$(blkid -s UUID -o 値 "${DISK}")
UUID_EFI=$(blkid -s UUID -o 値 "${DEV_EFI}")
```
---

# ▶️ 14. 休止状態をサポートするスワップファイルを作成する (オプション)

### 重要な注意事項
```
- Btrfs のスワップファイルは常に **prealloc** として表示されますが、これは正常です。
- RAM のフルサイズである必要はありません。
- ほとんどの場合、休止状態には 60% で十分です。
・重荷重の場合→70％または80％でご使用ください。
```

1. 最適なスワップファイルのサイズを自動的に計算します
```
# 休止状態に関する最新の推奨事項: 総 RAM の 60%
SWAP_GB=$(LC_ALL=C awk '/MemTotal/ {print int($2 * 0.60 / 1024 / 1024)}' /proc/meminfo)
echo "推奨スワップファイル: ${SWAP_GB}G"
```
- または、希望のサイズを手動で設定します。
```
SWAP_GB=4
echo "ユーザー定義のスワップファイル: ${SWAP_GB}G"
```
2. スワップファイル用のディレクトリを作成します。
```
mkdir -p /swap
スワップオフ -a 2>/dev/null
rm -f /swap/swapfile
```
3. COW を無効にする (Btrfs で必要)
```
chattr +C /スワップ
```
4. 事前に定義したサイズでスワップファイルを作成します
```
fallocate -l ${SWAP_GB}G /swap/swapfile
chmod 600 /swap/swapfile
```
5. スワップファイルをフォーマットし、スワップをアクティブ化します。
```
mkswap /swap/swapfile
スワポン /swap/swapfile
```
6. オフセットを取得します。
```
# filefrag のパッケージをインストールする
xbps-install -Sy e2fsprogs

# オフセットを取得する
offset=$(filefrag -v /swap/swapfile | awk '/^ *0:/{print $4}')
```
---

# ▶️ 15. GRUBの設定
⚠️ **重要:**
> このブロックは賢いです:
- LUKS を使用しているかどうかを自動的に検出します
- 休止状態でスワップファイルを作成したかどうかを検出します
- 何も複製せずに /etc/default/grub を調整します
- 必要な行が不足している場合にのみ作成します。
- 必要がない場合は何も変更しないでください

以下のブロックを正確に使用してください。
```
HAS_RESUME=false
HAS_LUKS=false

[[ -n "${offset}" ]] && HAS_RESUME=true
[[ "${DISK}" = "${DEV_LUKS}" ]] && HAS_LUKS=true

# 安全のため古い行を削除します
sed -i '/^[[:space:]]*GRUB_CMDLINE_LINUX_DEFAULT=/d' /etc/default/grub

# GRUB_CMDLINE_LINUX

# 基本値
BASE="ログレベル=4"

# 概要を追加
$HAS_RESUMEの場合;それから
BASE="$BASE 再開=UUID=${UUID_ROOT} 再開_オフセット=${offset}"
なれ

#LUKSを追加
$HAS_LUKSの場合;それから
grep -q '^GRUB_ENABLE_CRYPTODISK=y' /etc/default/grub || echo 'GRUB_ENABLE_CRYPTODISK=y' >> /etc/default/grub
grep -q '^GRUB_PRELOAD_MODULES=' /etc/default/grub || echo 'GRUB_PRELOAD_MODULES="luks cryptodisk gcry_rijndael"' >> /etc/default/grub
BASE="$BASE rd.luks.uuid=${UUID_LUKS} rd.luks.name=${UUID_LUKS}=cryptroot root=/dev/mapper/cryptroot"
なれ

# 最終行を正しく再作成します
echo "GRUB_CMDLINE_LINUX_DEFAULT=\"${BASE}\"" >> /etc/default/grub
```
---

# ▶️ 16. initrd を再作成する
⚠️ **重要:**
```
mods=(/usr/lib/modules/*)
KVER=$(ベース名 "${mods[0]}")
エコー ${KVER}
dracut --force --kver ${KVER}
```
---

# ▶️ 17. 起動時にパスワードを 2 回要求されることを避けるためにキーファイルを作成します (LUKS のみ)
> システムが LUKS を使用しない場合は、この手順をスキップしてください。
```
if [ "${DISK}" = "${DEV_LUKS}" ];それから
echo "LUKS が検出されました: 自動ロック解除用のキーファイルを作成しています..."

# 安全なキーファイルを作成する
dd if=/dev/urandom of=/boot/volume.key bs=64 count=1
chmod 000 /boot/volume.key

# キーファイルを LUKS に追加します (現在のパスワードを要求されます)
cryptsetup luksAddKey "${DEV_RAIZ}" /boot/volume.key

# /etc/crypttab を設定する
cat << EOF >> /etc/crypttab
cryptroot ${DEV_RAIZ} /boot/volume.key luks
終了後

# keyfile と crypttab を initramfs に含める
mkdir -p /etc/dracut.conf.d
cat << EOF >> /etc/dracut.conf.d/10-crypt.conf
install_items+=" /boot/volume.key /etc/crypttab "
終了後

# キーファイルサポートを使用して initramfs を再生成する
xbps-reconfigure -fa
それ以外
echo "LUKS のないシステム: キーファイルの作成をスキップします。"
なれ
```

# ▶️ 18. **BIOS** および **UEFI** に GRUB をインストールします (実際のハイブリッド)
1. BIOS 用の GRUB をインストールします (レガシー)
```
grub-install --target=i386-pc ${DEVICE}
```
2. UEFI 用の GRUB をインストールする
```
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=void
```
3. UEFI フォールバック (ユニバーサル ブート) を作成します。このファイルにより、NVRAM が消去された場合でも起動が保証されます。
```
mkdir -p /boot/efi/EFI/BOOT
cp -f /boot/efi/EFI/void/grubx64.efi /boot/efi/EFI/BOOT/BOOTX64.EFI
```
4. 最終的な GRUB ファイルを生成する
```
grub-mkconfig -o /boot/grub/grub.cfg
```

---

# ▶️ 19. カスタマイズされたユーザー設定:

1. 環境設定:

```
# /etc/xbps.d/00-repository-main.conf をカスタマイズする
mkdir -pv /etc/xbps.d
cat << 'EOF' >> /etc/xbps.d/00-repository-main.conf
リポジトリ=https://repo-fastly.voidlinux.org/current
#repository=https://repo-fastly.voidlinux.org/current/nonfree
#repository=https://repo-fastly.voidlinux.org/current/multilib
#repository=https://repo-fastly.voidlinux.org/current/multilib/nonfree

リポジトリ=https://void.chililinux.com/voidlinux/current
#repository=https://void.chililinux.com/voidlinux/current/extras
#repository=https://void.chililinux.com/voidlinux/current/nonfree
#repository=https://void.chililinux.com/voidlinux/current/multilib
#repository=https://void.chililinux.com/voidlinux/current/multilib/nonfree
終了後

# /etc/rc.conf をカスタマイズします。本体のデフォルトのタイムゾーン、キーボードレイアウト、フォントを設定します。必要に応じて変更します。
cat << EOF >> /etc/rc.conf
TIMEZONE="チリ_REF_0_チリ"
KEYMAP="チリ_REF_0_チリ"
FONT=Lat2-Terminus16
終了後

# root の .bashrc をカスタマイズする
wget --quiet --no-check-certificate \
-O /etc//skel/.bashrc \
「https://raw.githubusercontent.com/voidlinux-br/void-install/refs/heads/main/.bashrc"」
chown root:root /etc/skel/.bashrc
chmod 644 /etc/skel/.bashrc

cat << 'EOF' > /etc/skel/.bash_profile
# ~/.bash_profile — .bashrc を Void にロードします

# .bashrcが存在する場合はロード
if [ -f ~/.bashrc ];それから
ソース ~/.bashrc
なれ
終了後

# root とユーザーにコピーします
/root "/home/${NEWUSER}" の d の場合;する
cp -f /etc/skel/.bash_profile "$d/"
cp -f /etc/skel/.bashrc "$d/"
終わり

chown "${NEWUSER}:${NEWUSER}" "/home/${NEWUSER}/.bash_profile" "/home/${NEWUSER}/.bashrc"
chmod 644 "/home/${NEWUSER}/.bash_profile" "/home/${NEWUSER}/.bashrc"

# カスタム svlogtail をダウンロードする
wget --quiet --no-check-certificate \
-O /usr/bin/svlogtail\
「https://raw.githubusercontent.com/voidlinux-br/void-install/refs/heads/main/svlogtail"」
chmod +x /usr/bin/svlogtail
```

2. SSH を構成します (オプションですが推奨)。
```
mkdir -pv /etc/ssh/sshd_config.d/
cat << 'EOF' > /etc/ssh/sshd_config.d/10-custom.conf
TTY を許可します はい
PrintMotd はい
最終ログの印刷はい
バナー /etc/issue.net

PermitRootLogin はい
KbdInteractiveAuthentication はい
X11転送はい
公開鍵認証はい
PubkeyAcceptedKeyTypes=+ssh-rsa
AuthorizedKeysファイル .ssh/authorized_keys
パスワード認証はい
チャレンジレスポンス認証はい
PAM を使用する はい

サブシステム sftp 内部 sftp
終了後
```
---

# ▶️ 20. ZRAM を有効にする (オプション)
Void Linux は、zramen サービスを使用して ZRAM を有効にし、SSD スワップの使用量を減らし、負荷時のパフォーマンスを向上させる圧縮メモリのブロックを作成します。
1.zramenをインストールする
```
xbps-install -Sy zramen
```
2. ZRAM を構成します (推奨構成)。
```
cat << 'EOF' > /etc/zramen.conf
zram_fraction=0.5
zram_devices=1
zram_algorithm=zstd
終了後
```
3. runit でサービスをアクティブ化します。
```
ln -s
```
> ZRAM は起動するたびに自動的にアクティブになります

---

# ▶️ 21. インストールを完了する
1.Sair do chroot:
```
出口
```
2. /mnt にマウントされているすべてのパーティション (サブボリュームおよび /boot/efi) をアンマウントします。
```
umount -R /mnt
```
3. chroot 内でアクティブ化されているスワップファイルまたはスワップ パーティションを無効にします。
```
スワップオフ -a
```
4. 物理マシンまたは VM を再起動して、実際のブートをテストします。
```
リブート
```
> 忘れずにインストール メディアを取り出し、新しくインストールしたディスクから起動してください。
楽しむ！

---

# 🎉 完全、ハイブリッド、将来性のあるシステム
- BIOS + UEFI のブート
- フォールバックUEFI
- スナップショット付き Btrfs (スナッパー/タイムシフト対応)
- スワップファイルを使用した実際の休止状態
- パフォーマンス用のZram

この SSD は **地球上のあらゆるマシン**を起動します。

# 免責事項

```
このチュートリアルは無料です。自由に使用、コピー、変更、再配布できます。
コンテンツは **MIT ライセンス** に基づいて利用可能であり、独自のライセンスの対象となるオープン ソース ソフトウェアから派生したスニペットやコマンドが含まれる場合があります。

保証は提供されません。ここにあるものはすべて「現状のまま」提供されます。
Use por sua conta e risco. Nem o autor, nem colaboradores, nem o Void Linux são responsáveis por perdas, danos, falhas de sistema ou qualquer consequência do uso deste material.

必要に応じて、ソース コードを入手し、このチュートリアルの独自のバージョンを確認、調整、生成することができます。
```

