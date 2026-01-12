# 🧩 VOID LINUX 教程 — 使用 EXT4、XFS、JFS 或 BTRFS（子卷）、LUKS、休眠和 ZRAM 的混合安裝（UEFI + BIOS）
### 修訂和驗證版本 — 正確分區 + 通用啟動

本指南安裝一個完全**混合** Void Linux，能夠啟動任何類型的機器 - 舊的、新的或有問題的：

- 💾 **現代 UEFI** （具有正常輸入和後備）
- 🧮 **BIOS/Legacy**（完全兼容）
- 🧰 **帶 BIOS 啟動的 GPT (EF02)** — 對舊硬件的最大支持
- 🚀 **帶有子卷的 Btrfs**（可選），現成的快照
- 🔐 **LUKS1 與 GRUB 完全兼容**
- 🌙 **通過交換文件真正休眠**
- 🧊 **針對性能配置的 ZRAM**
- 🧱 **完全支持 EXT4、XFS、JFS 和 BTRFS**
- 💡 **Initramfs/GRUB 自動配置（LUKS + 恢復）**

📌 **不妥協，不重新安裝 GRUB，不浪費時間。 **
📌 **即使在已擦除 NVRAM 的計算機上也能保證啟動（BOOTX64.EFI 後備）。 **

---

# ▶️ 1. Bootar o Live ISO

建議：使用 glibc 版本以獲得更好的兼容性：
- 從以下位置下載 iso：
```
https://repo-default.voidlinux.org/live/current/void-live-x86_64-20250202-base.iso
```
- 或者在以下位置查找最新版本：
```
辣椒_REF_0_辣椒
```

1. 以 root 身份登錄。
```bash
登錄：根
密碼：voidlinux
```

2. 將 shell 從 *sh* 切換到 *bash*。
*dash/sh* **未實現**許多腳本使用的多項功能。
```bash
巴什
```

3. 將鍵盤佈局更改為 **ABNT2**，確保重音符號和符號的正確映射：
```bash
加載鍵 br-abnt2
```

4. 粘貼到終端（可選）— 用顏色、user@host:path 和最後一個命令的狀態進行提示 (✔/✘)。有用又美觀。
```bash
導出 PS1='\[\e[1;32m\]\u\[\e[1;33m\]@\[\e[1;36m\]\h\[\e[1;31m\]:\w \
$([[ $? -eq 0 ]] && echo -e "\e[1;32m✔" || echo -e "\e[1;31m✘$?") \
\[\e[0m\]\$ '
```

# ▶️ 2. 連接互聯網
- 對於 **Wi-Fi** *（如果使用電纜，請跳過此步驟）*：
```bash
wpa_passphrase "SSID" "密碼" > wifi.conf
wpa_supplicant -B -i wlan0 -c wifi.conf
dhcpcd 無線局域網0
```

1. 測試連接：
```bash
平-c3 8.8.8.8
ping -c3 repo-default.voidlinux.org
```

2.安裝所需的包：
⚠️ **重要：**
```bash
xbps-install -Sy xbps parted jfsutils xfsprogs nano zstd xz bash-completion
```
---

# ▶️ 3.識別磁盤
1. 列出可用磁盤並記下設備名稱（例如：`/dev/sda`、`/dev/vda`、`/dev/nvme0n1`）：
```bash
fdisk -l | fdisk -l | grep -E '^(磁盤|磁盤)'
```

# ▶️ 4. 定義教程中使用的變量：
⚠️ **重要：**

1. **使用前**定義設備：
> 1. **我們假設**教程為 `/dev/sda` (正常) 或 `/dev/nvme0n1` (nvme)
> 2. **根據您的光盤調整**（僅選擇**一個**或**另一個**型號）

對於 **普通** 磁盤（例如 /dev/sda）
```bash
導出設備=/dev/sda
導出 DEV_BIOS=${DEVICE}1
導出 DEV_EFI=${DEVICE}2
導出 DEV_ROOT=${DEVICE}3
導出 DEV_LUKS=/dev/mapper/cryptroot
```
對於 **NVMe** 磁盤（例如 /dev/nvme0n1），分區後綴更改 (`p`)
```bash
導出設備=/dev/nvme0n1
導出 DEV_BIOS=${DEVICE}p1
導出 DEV_EFI=${DEVICE}p2
導出 DEV_RAIZ=${DEVICE}p3
導出 DEV_LUKS=/dev/mapper/cryptroot
```

> 📌 **注意：**
> 設備 → 整個磁盤
DEV_BIOS → BIOS 啟動分區（1–2 MiB，無 FS，不掛載）
DEV_EFI → EFI 分區（FAT32）
DEV_ROOT → 根分區（普通或 LUKS）
DEV_LUKS → LUKS 映射 (/dev/mapper/cryptroot)

- 👉 在這裡您可以定義椎間盤的解剖結構。指南中的其他內容都遵循這些變量。
- 🔎 為什麼這是必要的？
因為在開始時聲明所有內容可以使下一個過程防錯。

2. 定義 **KEYMAP** 和 **TIMEZONE**（根據需要更改）：
```bash
導出 KEYMAP=br-abnt2
```
```bash
導出 TIMEZONE=美國/聖保羅
```

---

# ▶️ 5.分區磁盤
> - BIOS 分區**必須**是第一個。這提高了與舊主板、有問題的引導加載程序以及期望引導代碼位於磁盤第一個區域的 BIOS 的兼容性。
> - ESP 可以稍後出現，沒有任何問題 - UEFI 不關心位置。

### 理想且正確的順序：

- 1️⃣ BIOS 啟動 (EF02)
- 2️⃣ ESP（EFI系統，FAT32）
- 3️⃣ Btrfs/Ext4/Xfs/Jfs（根）

### 使用parted進行分區（自動）
> 這裡 **DEVICE** 已經在那裡定義了，所以不存在“神奇”變量。
```
wipefs-a“${DEVICE}”
分開--腳本“${DEVICE}”--\
mklabel gpt \
mkpart 主要 1MiB 2MiB 名稱 1 BIOS 設置 1 BIOS_grub 位於 \
mkpart 主要 fat32 2MiB 514MiB 名稱 2 EFI 設置 2 esp 位於 \
mkpart 主要 514MiB 100% 名稱 3 ROOT \
對齊檢查最優 1 \
對齊檢查最優 2 \
對齊檢查最優 1
分開--腳本“${DEVICE}”--打印
```
> - 分區1 → BIOS引導（bios_grub，無FS，不掛載）
> - 分區 2 → EFI (FAT32)
> - 分區 3 → ROOT（稍後我們將使用 EXT4/XFS/JFS/BTRFS 對其進行格式化，無論有或沒有 LUKS）
> - 我使用了 mkpart Primary 514MiB 100%，但沒有精確指定 FS，以避免佔用 FS。稍後你選擇FS。
---

# ▶️ 6.選擇安裝模式（NORMAL或LUKS）
⚠️ **重要：**
> 僅選擇下面兩個塊之一。
**不**運行這兩個步驟。

1. 正常安裝**（無LUKS）**
```bash
導出磁盤=“${DEV_RAIZ}”
```
- 將 DISK 設置為實際設備 /dev/sda3

2. **使用 LUKS** 安裝（加密根）
```
# 僅加密 LUKS1 上的根分區（兼容 GRUB）- 絕不加密整個磁盤
# 通過“是”確認來加密分區：
cryptsetup luksFormat --type luks1“${DEV_RAIZ}”

# 使用您的密碼打開分區。
cryptsetup 打開“${DEV_RAIZ}”密碼根

# 從現在開始，真正的根是映射設備
導出磁盤=“${DEV_LUKS}”
```
- LUKS 位於 /dev/sda3 之上，而不是整個磁盤
- 系統將安裝在/dev/mapper/cryptroot

👉 從這裡開始，一切都使用$DISK.

---

# ▶️ 7. 創建文件系統（FS）並掛載root
⚠️ **重要：**
> 僅選擇下面兩個塊之一。

1. **外部4**
```
mkfs.ext4 -F "${DISK}" -L ROOT
掛載-v“${DISK}”/mnt
```
2.**XFS**
```
mkfs.xfs -f“${DISK}”
掛載-v“${DISK}”/mnt
```
3. **JFS**
```
mkfs.jfs -f“${DISK}”
掛載-v“${DISK}”/mnt
```
4. **簡單的BTRFS**
```
mkfs.btrfs -f "${DISK}" -L ROOT
掛載-v“${DISK}”/mnt
```
5. **帶有子卷的 BTRFS**
```
mkfs.btrfs -f "${DISK}" -L ROOT

掛載${DISK} /mnt
btrfs 子卷創建 /mnt/@
btrfs 子卷創建 /mnt/@home
btrfs 子卷創建 /mnt/@log
btrfs 子卷創建 /mnt/@cache
btrfs 子卷創建 /mnt/@snapshots
卸載/mnt

mount -o 默認,noatime,ssd,compress=zstd:3,discard=async,space_cache=v2,commit=300,subvol=/@ ${DISK} /mnt
mkdir -p /mnt/{boot/efi,home,var/log,var/cache,.snapshots,swap}

mount -o 默認,noatime,ssd,compress=zstd:3,discard=async,space_cache=v2,commit=300,subvol=/@home ${DISK} /mnt/home
mount -o 默認,noatime,ssd,compress=zstd:3,discard=async,space_cache=v2,commit=300,subvol=/@cache ${DISK} /mnt/var/cache
mount -o 默認,noatime,ssd,compress=zstd:3,discard=async,space_cache=v2,commit=300,subvol=/@log ${DISK} /mnt/var/log
mount -o 默認,noatime,ssd,compress=zstd:3,discard=async,space_cache=v2,commit=300,subvol=/@snapshots ${DISK} /mnt/.snapshots
```
---

# ▶️ 8.準備並組裝ESP（EFI）
```
mkfs.fat -F32 -I“${DEV_EFI}”
mkdir -p /mnt/boot/efi
掛載-v“${DEV_EFI}”/mnt/boot/efi
```
>💡 BIOS分區（${DEV_BIOS}）沒有文件系統，不格式化，不掛載。
---

# ▶️ 9. Instalar o Void Linux 無需 chroot

1. 複製要在 chroot (/mnt) 中使用的存儲庫密鑰（XBPS 密鑰）
```
mkdir -p /A{tc, 蒸汽/xbps/xbps)
cp -rpaf /var/db/xbps/keys/*.plist /mnt/var/db/xbps/keys/
cp -fpa /etc/resolv.conf /mnt/etc/resolv.conf
```

2. 將基本系統安裝到新安裝的磁盤上：
```
xbps-install -Sy -R https://repo-default.voidlinux.org/current \
-r /mnt \
基本系統 btrfs-progs cryptsetup grub grub-x86_64-efi dracut linux \
linux-headers linux-firmware linux-firmware-network glibc-locales \
xtools dhcpcd openssh vim nano grc zstd xz bash-completion vpm vsv \
socklog-void wget 網絡工具 tmate ncurses jfsutils xfsprogs duf 樹 eza chrony
```
---

# ▶️ 10. Gerar fstab no /mnt (chroot)
```bash
# 在/mnt/etc/fstab中生成fstab
xgenfstab -U /mnt > /mnt/etc/fstab
```

```bash
# 檢查是否正確生成
貓 /mnt/etc/fstab
```

# ▶️ 11.使用chroot訪問已安裝的系統

1. 僱傭而不是克羅特：
```
xchroot /mnt /bin/bash
```
---

# ▶️ 12. 初始設置（在 chroot 中）
```
# 配置主機名 - 定義主機名
echo void > /etc/主機名

# 配置時區 - 定義時區
ln -sfv /usr/share/zoneinfo/"${TIMEZONE}" /etc/localtime

# 配置語言環境
-i -e 's/^#\(en_.Utf-8 UTF-8\)/)/' \
-E 's/^#\pt_br.br.utf-8 UTF-8\)/'\'\
/etc/default/libc-區域設置

# 生成語言環境
xbps-重新配置-f glibc-區域設置

# 修復 /var/service 符號鏈接中可能存在的錯誤（重要）：
rm -f /var/服務
ln -sf /etc/runit/runsvdir/default /var/service

# 激活一些服務
ln -sf /etc/sv/dbus /var/service/
ln -sf /etc/sv/dhcpcd /var/service/
ln -sf /etc/sv/sshd /var/service/
ln -sf /etc/sv/nanoklogd /var/service/
ln -sf /etc/sv/socklog-unix /var/service/
ln -sf /etc/sv/chronyd /var/service/

# 配置 sudo -wheel 組（可選，但推薦）
cat << 'EOF' > /etc/sudoers.d/g_wheel
%wheel ALL=(全部:全部) NOPASSWD: 全部
EOF
#所需權限
chmod 440 /etc/sudoers.d/g_wheel
```

## 創建和配置用戶

⚠️ **重要：** 在下面定義真實的用戶名。
```bash
導出 NEWUSER=your_user_here
```

創建具有主目錄、基本組和 Bash shell 的用戶：
```bash
useradd -m -G 音頻、視頻、wheel、tty -s /bin/bash ${NEWUSER}
```

設置您的用戶密碼（***重要***）
```bash
密碼${NEWUSER}
```

設置 root 用戶密碼（***重要***）
```bash
密碼根
```

將 root 用戶的默認 shell 更改為 Bash
```bash
chsh -s /bin/bash 根目錄
```
---

# ▶️ 13.配置UUID
⚠️ **重要：**
- 獲取分區的UUID：
```
UUID_LUKS=$(blkid -s UUID -o 值“${DEV_RAIZ}”)
UUID_ROOT=$(blkid -s UUID -o 值“${DISK}”)
UUID_EFI=$(blkid -s UUID -o 值“${DEV_EFI}”)
```
---

# ▶️ 14. 創建支持休眠的交換文件（可選）

### 重要提示
```
- Btrfs 中的交換文件總是顯示為 **prealloc**，這是正常的。
- 它不需要是 RAM 的完整大小。
- 大多數情況下，60% 足以用於休眠。
- 對於重負載 → 使用 70% 或 80%。
```

1.自動計算最佳交換文件大小
```
# 現代建議的休眠：總 RAM 的 60%
SWAP_GB=$(LC_ALL=C awk '/MemTotal/ {print int($2 * 0.60 / 1024 / 1024)}' /proc/meminfo)
echo "推薦的交換文件：${SWAP_GB}G"
```

```
SWAP_GB=4
echo "用戶定義的交換文件：${SWAP_GB}G"
```
2. 為交換文件創建目錄
```
mkdir -p /交換
swapoff -a 2>/dev/null
rm -f /交換/交換文件
```
3.禁用COW（Btrfs中需要）
```
chattr +C /交換
```
4. 創建具有先前定義大小的交換文件
```

chmod 600 /交換/交換文件
```
5.格式化swap文件並激活swap
```
mkswap /交換/交換文件
交換/交換/交換文件
```
6. 獲取偏移量：
```
# 安裝filefrag包
xbps-安裝-Sy e2fsprogs

# 獲取偏移量
偏移量=$(filefrag -v /swap/swapfile | awk '/^ *0:/{print $4}')
```
---

# ▶️    15. Configurar o GRUB
⚠️ **重要：**
> 這個塊很聰明：

- 檢測您是否使用休眠創建了交換文件
- 調整 /etc/default/grub 而不重複任何內容
- 僅當缺少必要的行時才創建它們
- 如果沒有必要，不要改變任何東西

完全使用下面的塊：
```
HAS_RESUME=假
HAS_LUKS=假

[[ -n "${offset}" ]] && HAS_RESUME=true
[[“${DISK}”=“${DEV_LUKS}”]] && HAS_LUKS=true

# 為了安全起見，刪除舊線
sed -i '/^[[:空格:]]*GRUB_CMDLINE_LINUX_DEFAULT=/d' /etc/default/grub

# GRUB_CMDLINE_LINUX

# 基值
BASE=“日誌級別=4”

# 添加摘要
如果$HAS_RESUME；然後
BASE =“$BASE恢復= UUID = ${UUID_ROOT}恢復_偏移= ${offset}”
是

# 添加幸運
如果$HAS_LUKS；然後
grep -q '^GRUB_ENABLE_CRYPTODISK=y' /etc/default/grub || echo 'GRUB_ENABLE_CRYPTODISK=y' >> /etc/default/grub
grep -q '^GRUB_PRELOAD_MODULES=' /etc/default/grub || echo 'GRUB_PRELOAD_MODULES="luks cryptodisk gcry_rijndael"' >> /etc/default/grub
BASE="$BASE rd.luks.uuid=${UUID_LUKS} rd.luks.name=${UUID_LUKS}=cryptroot root=/dev/mapper/cryptroot"
是

# 正確重新創建最後一行
echo "GRUB_CMDLINE_LINUX_DEFAULT=\"${BASE}\"" >> /etc/default/grub
```
---

# ▶️ 16. 重新創建 initrd
⚠️ **重要：**
```
mods=(/usr/lib/modules/*)
KVER=$(基本名稱“${mods[0]}”)
回顯${KVER}
dracut --force --kver ${KVER}
```
---

# ▶️ 17. 創建密鑰文件以避免在啟動時兩次詢問密碼（僅限 LUKS）
> 如果系統不使用 LUKS，請跳過此步驟。
```
如果[“${DISK}”=“${DEV_LUKS}”];然後
echo“檢測到LUKS：正在創建用於自動解鎖的密鑰文件...”

# 創建安全密鑰文件
dd if=/dev/urandom of=/boot/volume.key bs=64 count=1
chmod 000 /boot/volume.key

# 將密鑰文件添加到 LUKS（會詢問您當前的密碼）
cryptsetup luksAddKey "${DEV_RAIZ}" /boot/volume.key

# 配置/etc/crypttab
貓 << EOF >> /etc/crypttab
cryptroot ${DEV_RAIZ} /boot/volume.key luks
EOF

# 在 initramfs 中包含 keyfile 和 crypttab
mkdir -p /etc/dracut.conf.d
cat << EOF >> /etc/dracut.conf.d/10-crypt.conf
install_items+=" /boot/volume.key /etc/crypttab "
EOF

# 重新生成具有密鑰文件支持的 initramfs
xbps-重新配置-fa
別的
echo“沒有 LUKS 的系統：跳過密鑰文件創建。”
是
```

# ▶️ 18.在**BIOS**和**UEFI**中安裝GRUB（真正的混合）
1. 安裝適用於 BIOS 的 GRUB（舊版）
```
grub-install --target=i386-pc ${DEVICE}
```
2.為UEFI安裝GRUB
```
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=void
```
3. 創建 UEFI 回退（通用啟動）。即使 NVRAM 被擦除，該文件也能保證啟動。
```
mkdir -p /boot/efi/EFI/BOOT
cp -f /boot/efi/EFI/void/grubx64.efi /boot/efi/EFI/BOOT/BOOTX64.EFI
```
4.生成最終的GRUB文件
```
grub-mkconfig -o /boot/grub/grub.cfg
```

---

# ▶️ 19. 自定義用戶設置：

1、環境設置：

```
# 自定義/etc/xbps.d/00-repository-main.conf
mkdir -pv /etc/xbps.d
cat << 'EOF' >> /etc/xbps.d/00-repository-main.conf
存儲庫=https://repo-fastly.voidlinux.org/current
#repository=https://repo-fastly.voidlinux.org/current/nonfree
#repository=https://repo-fastly.voidlinux.org/current/multilib
#repository=https://repo-fastly.voidlinux.org/current/multilib/nonfree

存儲庫=https://void.chililinux.com/voidlinux/current
#repository=https://void.chililinux.com/voidlinux/current/extras
#repository=https://void.chililinux.com/voidlinux/current/nonfree
#repository=https://void.chililinux.com/voidlinux/current/multilib
#repository=https://void.chililinux.com/voidlinux/current/multilib/nonfree
EOF

# 自定義/etc/rc.conf。設置控制台的默認時區、鍵盤佈局和字體。根據需要進行更改。
貓 << EOF >> /etc/rc.conf
時區=“${TIMEZONE}”
KEYMAP="${KEYMAP}"
FONT=Lat2-Terminus16
EOF

# 自定義 root 的 .bashrc
wget --quiet --no-check-certificate \
-O /etc//skel/.bashrc \
“辣椒_REF_0_辣椒
chown root:root /etc/skel/.bashrc
chmod 644 /etc/skel/.bashrc

貓 << 'EOF' > /etc/skel/.bash_profile
# ~/.bash_profile — 將 .bashrc 加載到 Void 中

# 如果.bashrc存在，則加載
如果 [ -f ~/.bashrc ];然後
源~/.bashrc
是
EOF

# 複製到root和用戶
for d in /root "/home/${NEWUSER}";做
cp -f /etc/skel/.bash_profile“$d/”
cp -f /etc/skel/.bashrc "$d/"
完畢

chown "${NEWUSER}:${NEWUSER}" "/home/${NEWUSER}/.bash_profile" "/home/${NEWUSER}/.bashrc"
chmod 644“/home/${NEWUSER}/.bash_profile”“/home/${NEWUSER}/.bashrc”

# 下載自定義 svlogtail
wget --quiet --no-check-certificate \
-O /usr/bin/svlogtail\
“辣椒_REF_0_辣椒
chmod +x /usr/bin/svlogtail
```

2.配置ssh（可選，但推薦）：
```
mkdir -pv /etc/ssh/sshd_config.d/
cat << 'EOF' > /etc/ssh/sshd_config.d/10-custom.conf
允許 TTY 是
PrintMotd 是
打印最後日誌 是
橫幅 /etc/issue.net

允許根登錄 是
KbdInteractiveAuthentication 是
X11轉發 是
公鑰驗證 是
PubkeyAcceptedKeyTypes=+ssh-rsa
AuthorizedKeysFile .ssh/authorized_keys
密碼驗證 是
挑戰響應身份驗證 是
使用PAM 是

子系統 sftp 內部 sftp
EOF
```
---

# ▶️ 20.啟用ZRAM（可選）
Void Linux 使用 zramen 服務來啟用 ZRAM，創建一個壓縮內存塊，以減少 SSD 交換使用並提高負載下的性能。
1.安裝zramen
```
xbps-install -Sy zramen
```
2.配置ZRAM（推薦配置）：
```
貓 << 'EOF' > /etc/zramen.conf
zram_fraction=0.5
zram_設備=1
zram_算法=zstd
EOF
```
3.激活runit中的服務
```
ln-s
```
> ZRAM 將在每次啟動時自動激活

---

# ▶️ 21.完成安裝
1. 執行chroot：
```
出口
```
2. 卸載掛載在 /mnt 上的所有分區（子捲和 /boot/efi）：
```
卸載-R /mnt
```
3. 禁用已在 chroot 中激活的任何交換文件或交換分區：
```
交換-a
```
4、重啟物理機或VM來測試實際啟動情況：
```
重新啟動
```
> 不要忘記取出安裝介質並從新安裝的光盤啟動。
享受！

---

# 🎉 完整、混合、面向未來的系統
- 啟動 BIOS + UEFI
- 後備 UEFI
- 帶快照的 Btrfs（Snapper/Timeshift 就緒）
- 真正的休眠與交換文件
- 茲拉姆的表現

此 SSD 可啟動**地球上的任何機器**。

# 免責聲明

```
本教程是免費的：您可以根據需要使用、複製、修改和重新分發。
內容根據 **MIT 許可證**提供，並且可能包括源自受其自身許可證約束的開源軟件的片段或命令。

不提供任何保證——此處的所有內容均“按原樣”交付。
使用風險自負。作者、貢獻者和 Void Linux 均不對使用本材料的損失、損壞、系統故障或任何後果負責。

如果您願意，您可以獲取源代碼、查看、改編並生成您自己的本教程版本。
```

