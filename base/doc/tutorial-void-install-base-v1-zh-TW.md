# 🔥 Base Void Linux 安裝教程

# 開始之前

本教程描述了使用直接磁盤分區、“chroot”和顯式系統配置的 **手動安裝 Void Linux**。
它**不是自動安裝程序**。

## ⚠️仔細閱讀

- 本指南**假設熟悉 Linux**、終端和基本系統概念（磁盤、分區、引導、服務）。
- 多個命令**永久刪除數據**（`parted`、`mkfs`、`umount -R`）。
- 定義磁盤（`/dev/sdX`、`/dev/nvmeX`）時出現錯誤可能會導致**全部數據丟失**。
- 在執行任何命令之前閱讀**整個教程**。

## 🖥️推薦環境

- **VM（VirtualBox、QEMU、KVM 等）** 用於測試和學習。
- 專用硬件**無重要數據**。
- 實驗室環境或有意識的設施。

❌ **不建議** 不經修改直接用於生產。

## 🔐關於安全

安裝過程中，有些設置**優先考慮實用性**，而不是安全性：
- 可以暫時啟用通過 SSH 的“root”用戶登錄。
- 密碼驗證可能處於活動狀態。
- 可能允許舊版兼容性（例如 `ssh-rsa`）。

👉 **安裝後必須檢查這些設置**，尤其是在暴露於網絡的系統上。

## 🧠 重要了解

- 執行命令**一一**，檢查輸出。
- 根據您的系統調整磁盤名稱、網絡接口和用戶。
- **不要盲目複製粘貼**。
- 如有疑問，**停止**並檢查當前步驟。

## 🛠️ 如果出現錯誤

如果出現問題：
- 不要盲目重啟。
- 重新組裝分區。
- 使用“chroot”重新登錄系統。
- 檢查 GRUB、EFI 和 `initramfs`。

犯錯誤是其中的一部分。理解錯誤是用戶與操作員的區別。

---

> 本指南針對喜歡**完全控制**安裝的用戶，遵循經典的 Unix 方法：
> **理解→配置→驗證→繼續**。

## 開始安裝
從 Void Linux ISO（x86_64 glibc 或 musl）開始。

1. 以 root 身份登錄
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

4. 連接互聯網
- 對於 **Wi-Fi** *（如果使用電纜，請跳過此步驟）*：
```bash
wpa_passphrase“WIFI_NETWORK_NAME”“NETWORK_PASSWORD”> wifi.conf
wpa_supplicant -B -i wlan0 -c wifi.conf
dhcpcd 無線局域網0
```
> 📌 **注意：** `wlan0` 可能會有所不同（`wlp2s0`、`wlp0s3` 等）。
> 使用以下命令來識別正確的接口：
>
> ```bash
> ip -br a
> ```

5. 測試連接：
```bash
平-c3 8.8.8.8
ping -c3 repo-default.voidlinux.org
```

## 通過 SSH 啟用 **root** 用戶登錄（可選）。
此步驟僅適用於系統運行在虛擬機中的情況；如果是本地啟動（沒有VM），則可以通過本地終端正常安裝。
- 這是從主機訪問 **VM 並繼續遠程安裝所必需的；之後，可以通過 SSH 直接在終端中粘貼/執行命令。

1.配置ssh
```bash
echo 'PermitRootLogin 是' >> /etc/ssh/sshd_config
```
2.重啟ssh服務
```bash
sv 重新啟動 sshd
```

3.查看網絡接口IP
```bash
ip -4 路由獲取 1.1.1.1 | awk '{print $7}'
```
>記下網絡接口的IP並使用它通過SSH連接到VM。

4. 從主機通過 SSH 訪問虛擬機。
```bash
sudo ssh <ip-da-vm>
```
> 默認密碼：`voidlinux`

## 在終端中配置彩色提示（可選）
它將顯示用戶、主機、當前路徑和最後一個命令的狀態：
```bash
導出 PROMPT_COMMAND='RET=$?'
導出 PS1='\[\e[1;33m\]\u\[\e[0m\]@\[\e[1;35m\]\h\[\e[0m\]:\[\e[0;37m\]\w\[\e[0m\] \[\e[1;32m\]$( [ $RET -eq 0 ] && printf ✔ || printf "\e[1;31m✘$RET" )\[\e[0m\] \$ '
```
> 📌此提示僅對當前會話有效；要使其永久，請將其添加到“.bashrc”。

## 安裝需要的包
⚠️ **重要：**
```bash
xbps-install -Sy xbps parted nano vim zstd xz bash-completion
```

## 對磁盤進行分區
1.識別磁盤
```bash
fdisk -l | fdisk -l | grep -E '^(磁盤|磁盤)'
```
> 我們將假設教程為 `/dev/sda`

2. 根據將使用的磁盤調整以下變量（**重要**）：
```bash
# SATA/SCSI 磁盤 (sdX)
導出設備=/dev/sda
導出 DEV_EFI=${DEVICE}2
導出 DEV_ROOT=${DEVICE}3
```

> 📌 **注意：**
> 對於 **NVMe** 磁盤，分區後綴更改 (`p`)：
> ```bash
> 導出設備=/dev/nvme0n1
> 導出 DEV_EFI=${DEVICE}p2
> 導出 DEV_RAIZ=${DEVICE}p3
> ```

3. 使用**parted**（自動模式）對磁盤進行分區。
該方案創建：
- BIOS 分區 (bios_grub)
- EFI 分區（ESP）
- 根分區（ROOT）
```bash
wipefs-a“${DEVICE}”
分開--腳本“${DEVICE}”--\
mklabel gpt \
mkpart 主要 1MiB 2MiB 名稱 1 BIOS 設置 1 BIOS_grub 位於 \
mkpart 主要 fat32 2MiB 514MiB 名稱 2 EFI 設置 2 esp 位於 \
mkpart 主要 514MiB 100% 名稱 3 ROOT \
對齊檢查最優 1 \
對齊檢查最優 2 \
對齊檢查最優 3
分開--腳本“${DEVICE}”--打印
```

## 格式化分區
```bash
# 格式化根分區（ext4）
mkfs.ext4 -F ${DEV_RAIZ}

# 格式化EFI分區（FAT32）
mkfs.fat -F32 -I ${DEV_EFI}
```

## 將捲掛載到 `/mnt` 中
```bash
# 掛載根分區
掛載${DEV_RAIZ} /mnt

# 創建必要的掛載點
/t /mnt/{hame,boot/efi,var/log,var/cache, proced, proc, Proc,)

#掛載EFI分區
掛載 ${DEV_EFI} /mnt/boot/efi
```

## 安裝基礎系統
將Void Linux基礎系統安裝到`/mnt`安裝環境中，包括內核、固件、引導加載程序、網絡和基本工具。
```bash
xbps-install -Sy -R https://repo-default.voidlinux.org/current \
-r /mnt \
基本系統 e2fsprogs grub-x86_64-efi dracut linux \
linux-headers linux-firmware linux-firmware-network glibc-locales \
xtools dhcpcd openssh vim nano grc zstd xz bash-completion vpm vsv \
socklog-void wget 網絡工具 tmate ncurses chrony
```

> 📌 **注意：**
> - `grub-x86_64-efi` → 引導加載程序 UEFI
> - `linux` → 內核
> - `linux-firmware-network` → 網絡驅動程序
> - `xtools` → 必須使用 `xgenfstab` 才能成功

## 創建`fstab`
自動生成系統的永久掛載文件。
```bash
xgenfstab -U /mnt > /mnt/etc/fstab
```

## 入口和系統（chroot）
訪問安裝在`/mnt`的系統以繼續配置。
```bash
xchroot /mnt /bin/bash
```

## 生成 INITRAMFS
虛擬化環境的 Dracut 配置（VM 安全）
```bash
貓 > /etc/dracut.conf.d/99-vm-safe.conf << 'EOF'
# /etc/dracut.conf.d/99-vm-safe.conf
僅主機=否
壓縮=“gzip”
add_drivers+=“virtio virtio_pci virtio_blk virtio_net virtio_scsi”
EOF
```

自動檢測安裝的內核版本並使用 **dracut** 生成相應的 `initramfs`。
```bash
mods=(/usr/lib/modules/*)
KVER=$(基本名稱“${mods[0]}”)
回顯${KVER}
dracut --force --kver ${KVER}
```

## 配置GRUB

> 📌 兩種方法（BIOS 和 UEFI）都是故意安裝的。
> 這允許同一個磁盤在 **Legacy BIOS** 和 **UEFI** 系統上啟動，從而提高機器之間的可移植性。

1.創建GRUB支持目錄：
```bash
mkdir -p /啟動/grub
```

2. 安裝 **BIOS（舊版）** 的 GRUB：
```bash
grub-install --target=i386-pc ${DEVICE}
```

3. 為 **UEFI** 安裝 GRUB：
```bash
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=void
```

4. 創建 UEFI 後備（通用啟動）。
即使 NVRAM 被擦除，該文件也能保證啟動：
```bash
mkdir -p /boot/efi/EFI/BOOT
cp -f /boot/efi/EFI/void/grubx64.efi /boot/efi/EFI/BOOT/BOOTX64.EFI
```

5. 生成最終的GRUB配置文件：
```bash
grub-mkconfig -o /boot/grub/grub.cfg
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

## 基本設置
```bash
# Setar 主機名
echo void > /etc/主機名

# 設置當地時間
ln -sf /usr/share/zoneinfo/America/Sao_Paulo /etc/localtime

# 士達本地
sed -i 's/#en_US.UTF-8/en_US.UTF-8/' /etc/default/libc-locales
sed -i 's/#pt_BR.UTF-8/pt_BR.UTF-8/' /etc/default/libc-locales

# 生成語言環境：
xbps-重新配置-f glibc-區域設置

# 修復 /var/service 符號鏈接中可能存在的錯誤（重要）：
rm -f /var/服務
ln -sf /etc/runit/runsvdir/default /var/service

# 激活一些服務：
ln -sf /etc/sv/dbus /var/service/
ln -sf /etc/sv/dhcpcd /var/service/
ln -sf /etc/sv/sshd /var/service/
ln -sf /etc/sv/nanoklogd /var/service/
ln -sf /etc/sv/socklog-unix /var/service/
ln -sf /etc/sv/chronyd /var/service/

# 下載自定義 svlogtail （可選，但推薦）：
wget --quiet --no-check-certificate -O /usr/bin/svlogtail \
“https://raw.githubusercontent.com/voidlinux-br/void-install/refs/heads/main/svlogtail" && \
chmod +x /usr/bin/svlogtail

# 創建一個resolv.conf
printf '名稱服務器 1.1.1.1\n名稱服務器 8.8.8.8\n' > /etc/resolv.conf

#配置sudo-wheel組（可選，但推薦）
cat << 'EOF' > /etc/sudoers.d/g_wheel
%wheel ALL=(全部:全部) NOPASSWD: 全部
EOF

#所需權限
chmod 440 /etc/sudoers.d/g_wheel
```

## 自定義 `/etc/xbps.d/00-repository-main.conf`
*（可選，但推薦）*

創建 **XBPS** 配置目錄（如果尚不存在）並定義官方和替代存儲庫的列表。
**repo-fastly** 存儲庫往往具有更好的延遲。

```bash
mkdir -pv /etc/xbps.d

cat << 'EOF' > /etc/xbps.d/00-repository-main.conf
# 官方存儲庫（快速 - 最佳延遲）
存儲庫=https://repo-fastly.voidlinux.org/current
#repository=https://repo-fastly.voidlinux.org/current/nonfree
#repository=https://repo-fastly.voidlinux.org/current/multilib
#repository=https://repo-fastly.voidlinux.org/current/multilib/nonfree

# 替代存儲庫 (Chili Linux)
存儲庫=https://void.chililinux.com/voidlinux/current
#repository=https://void.chililinux.com/voidlinux/current/extras
#repository=https://void.chililinux.com/voidlinux/current/nonfree
#repository=https://void.chililinux.com/voidlinux/current/multilib
#repository=https://void.chililinux.com/voidlinux/current/multilib/nonfree
EOF
```

## 自定義 `/etc/rc.conf`
設置控制台的默認時區、鍵盤佈局和字體。
根據需要進行更改。
```bash
貓 << 'EOF' > /etc/rc.conf
TIMEZONE=美國/聖保羅
鍵盤映射=br-abnt2
FONT=Lat2-Terminus16
EOF
```

Virtio 模塊（虛擬機）。
```bash
cat > /etc/modules-load.d/virtio.conf << 'EOF'
虛擬機
virtio_pci
virtio_net
virtio_blk
virtio_scsi
EOF
```

## 自定義用戶`.bashrc`
創建默認的“.bash_profile”並確保在登錄時自動加載“.bashrc”。
> ⚠️ 確保用戶是在上一步中創建的。

```bash
# 下載默認的.bashrc到/etc/skel
wget --quiet --no-check-certificate \
-O /etc/skel/.bashrc \
“辣椒_REF_0_辣椒

chown root:root /etc/skel/.bashrc
chmod 644 /etc/skel/.bashrc

# 創建默認的.bash_profile
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

# 調整用戶權限
chown "${NEWUSER}:${NEWUSER}" \
“/home/${NEWUSER}/.bash_profile”\
“/home/${NEWUSER}/.bashrc”

chmod 644 \
“/home/${NEWUSER}/.bash_profile”\
“/home/${NEWUSER}/.bashrc”
```

## 配置 SSH
*（可選，但推薦）*

為 **sshd** 創建補充配置文件，保持主文件不變。
```bash
mkdir -pv /etc/ssh/sshd_config.d

cat << 'EOF' > /etc/ssh/sshd_config.d/10-custom.conf
# 常規設置
允許 TTY 是
PrintMotd 是
打印最後日誌 是
橫幅 /etc/issue.net

＃ 驗證
允許根登錄 是
密碼驗證 是
KbdInteractiveAuthentication 是
挑戰響應身份驗證 是
公鑰驗證 是
PubkeyAcceptedKeyTypes=+ssh-rsa
AuthorizedKeysFile .ssh/authorized_keys
使用PAM 是

＃ 特徵
X11轉發 是
子系統 sftp 內部 sftp
EOF
```
> ⚠️ 建議在首次啟動後檢查並強化這些 SSH 設置，尤其是在暴露於 Internet 的系統上。

## 完成安裝
Sair do chroot
```bash
出口
```
卸載掛載在 `/mnt` 上的所有分區（包括子捲和 `/boot/efi`）：
```bash
卸載-R /mnt
```
重啟物理機或虛擬機測試實際啟動情況：
```bash
重新啟動
```
> 📌 **注意：不要忘記刪除安裝介質。

# 🎉 享受吧！
**Void Linux** 現已安裝並可以使用。

# 免責聲明

> 本教程是免費的：您可以根據需要使用、複製、修改和重新分發。
> 內容根據 **MIT 許可證** 提供，並且可能包括源自開源軟件的摘錄或命令，但須遵守其自己的許可證。
>
> 不提供任何保證 — 這裡的所有內容均按 **“按原樣”** 交付。
> 使用時需自行承擔風險。作者、貢獻者和 Void Linux 均不對使用本材料的損失、損壞、系統故障或任何後果負責。
>
> 您可以自由查看、改編和生成本教程的您自己的版本。

