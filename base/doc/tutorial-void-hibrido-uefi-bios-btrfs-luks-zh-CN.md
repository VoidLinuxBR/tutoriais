# 🧩 VOID LINUX 教程 — 使用 EXT4、XFS、JFS 或 BTRFS（子卷）、LUKS、休眠和 ZRAM 的混合安装（UEFI + BIOS）
### 修订和验证版本 — 正确分区 + 通用启动

本指南安装一个完全**混合** Void Linux，能够启动任何类型的机器 - 旧的、新的或有问题的：

- 💾 **现代 UEFI** （具有正常输入和后备）
- 🧮 **BIOS/Legacy**（完全兼容）
- 🧰 **带 BIOS 启动的 GPT (EF02)** — 对旧硬件的最大支持
- 🚀 **带有子卷的 Btrfs**（可选），现成的快照
- 🔐 **LUKS1 与 GRUB 完全兼容**
- 🌙 **通过交换文件真正休眠**
- 🧊 **针对性能配置的 ZRAM**
- 🧱 **完全支持 EXT4、XFS、JFS 和 BTRFS**
- 💡 **Initramfs/GRUB 自动配置（LUKS + 恢复）**

📌 **不妥协，不重新安装 GRUB，不浪费时间。**
📌 **即使在已擦除 NVRAM 的计算机上也能保证启动（BOOTX64.EFI 后备）。**

---

# ▶️ 1. Bootar o Live ISO

建议：使用 glibc 版本以获得更好的兼容性：
- 从以下位置下载 iso：
```
辣椒_REF_0_辣椒
```
- 或者在以下位置查找最新版本：
```
辣椒_REF_0_辣椒
```

1. 以 root 身份登录。
```bash
登录：根
密码：voidlinux
```

2. 将 shell 从 *sh* 切换到 *bash*。
*dash/sh* **未实现**许多脚本使用的多项功能。
```bash
巴什
```

3. 将键盘布局更改为 **ABNT2**，确保重音符号和符号的正确映射：
```bash
加载键 br-abnt2
```

4. 粘贴到终端（可选）— 用颜色、user@host:path 和最后一个命令的状态进行提示 (✔/✘)。有用又美观。
```bash
导出 PS1='\[\e[1;32m\]\u\[\e[1;33m\]@\[\e[1;36m\]\h\[\e[1;31m\]:\w \
$([[ $? -eq 0 ]] && echo -e "\e[1;32m✔" || echo -e "\e[1;31m✘$?") \
\[\e[0m\]\$ '
```

# ▶️ 2. 连接互联网
- 对于 **Wi-Fi** *（如果使用电缆，请跳过此步骤）*：
```bash
wpa_passphrase "SSID" "密码" > wifi.conf
wpa_supplicant -B -i wlan0 -c wifi.conf
dhcpcd 无线局域网0
```

1. 测试连接：
```bash
平-c3 8.8.8.8
ping -c3 repo-default.voidlinux.org
```

2.安装所需的包：
⚠️ **重要：**
```bash
xbps-install -Sy xbps parted jfsutils xfsprogs nano zstd xz bash-completion
```
---

# ▶️ 3.识别磁盘
1. 列出可用磁盘并记下设备名称（例如：`/dev/sda`、`/dev/vda`、`/dev/nvme0n1`）：
```bash
fdisk -l | fdisk -l | grep -E '^(磁盘|磁盘)'
```

# ▶️ 4. 定义教程中使用的变量：
⚠️ **重要：**

1. **使用前**定义设备：
> 1. **我们假设**教程为 `/dev/sda` (正常) 或 `/dev/nvme0n1` (nvme)
> 2. **根据您的光盘调整**（仅选择**一个**或**另一个**型号）

对于 **普通** 磁盘（例如 /dev/sda）
```bash
导出设备=/dev/sda
导出 DEV_BIOS=${DEVICE}1
导出 DEV_EFI=${DEVICE}2
导出 DEV_ROOT=${DEVICE}3
导出 DEV_LUKS=/dev/mapper/cryptroot
```
对于 **NVMe** 磁盘（例如 /dev/nvme0n1），分区后缀更改 (`p`)
```bash
导出设备=/dev/nvme0n1
导出 DEV_BIOS=${DEVICE}p1
导出 DEV_EFI=${DEVICE}p2
导出 DEV_RAIZ=${DEVICE}p3
导出 DEV_LUKS=/dev/mapper/cryptroot
```

> 📌 **注意：**
> 设备 → 整个磁盘
DEV_BIOS → BIOS 启动分区（1–2 MiB，无 FS，不挂载）
DEV_EFI → EFI 分区（FAT32）
DEV_ROOT → 根分区（普通或 LUKS）
DEV_LUKS → LUKS 映射 (/dev/mapper/cryptroot)

- 👉 在这里您可以定义椎间盘的解剖结构。指南中的其他内容都遵循这些变量。
- 🔎 为什么这是必要的？
因为在开始时声明所有内容可以使下一个过程防错。

2. 定义 **KEYMAP** 和 **TIMEZONE**（根据需要更改）：
```bash
导出 KEYMAP=br-abnt2
```
```bash
导出 TIMEZONE=美国/圣保罗
```

---

# ▶️ 5.分区磁盘
> - BIOS 分区**必须**是第一个。这提高了与旧主板、有问题的引导加载程序以及期望引导代码位于磁盘第一个区域的 BIOS 的兼容性。
> - ESP 可以稍后出现，没有任何问题 - UEFI 不关心位置。

### 理想且正确的顺序：

- 1️⃣ BIOS 启动 (EF02)
- 2️⃣ ESP（EFI系统，FAT32）
- 3️⃣ Btrfs/Ext4/Xfs/Jfs（根）

### 使用parted进行分区（自动）
> 这里 **DEVICE** 已经在那里定义了，所以不存在“神奇”变量。
```
wipefs-a“${DEVICE}”
分开--脚本“${DEVICE}”--\
mklabel gpt \
mkpart 主要 1MiB 2MiB 名称 1 BIOS 设置 1 BIOS_grub 位于 \
mkpart 主要 fat32 2MiB 514MiB 名称 2 EFI 设置 2 esp 位于 \
mkpart 主要 514MiB 100% 名称 3 ROOT \
对齐检查最优 1 \
对齐检查最优 2 \
对齐检查最优 1
分开--脚本“${DEVICE}”--打印
```
> - 分区1 → BIOS引导（bios_grub，无FS，不挂载）
> - 分区 2 → EFI (FAT32)
> - 分区 3 → ROOT（稍后我们将使用 EXT4/XFS/JFS/BTRFS 对其进行格式化，无论有或没有 LUKS）
> - 我使用了 mkpart Primary 514MiB 100%，但没有精确指定 FS，以避免占用 FS。稍后你选择FS。
---

# ▶️ 6.选择安装模式（NORMAL或LUKS）
⚠️ **重要：**
> 仅选择下面两个块之一。
**不**运行这两个步骤。

1. 正常安装**（无LUKS）**
```bash
导出磁盘=“${DEV_RAIZ}”
```
- 将 DISK 设置为实际设备 /dev/sda3

2. **使用 LUKS** 安装（加密根）
```
# 仅加密 LUKS1 上的根分区（兼容 GRUB）- 绝不加密整个磁盘
# 通过“是”确认来加密分区：
cryptsetup luksFormat --type luks1“${DEV_RAIZ}”

# 使用您的密码打开分区。
cryptsetup 打开“${DEV_RAIZ}”密码根

# 从现在开始，真正的根是映射设备
导出磁盘=“${DEV_LUKS}”
```
- LUKS 位于 /dev/sda3 之上，而不是整个磁盘
- 系统将安装在/dev/mapper/cryptroot

👉 从这里开始，一切都使用$DISK.

---

# ▶️ 7. 创建文件系统（FS）并挂载root
⚠️ **重要：**
> 仅选择下面两个块之一。

1. **外部4**
```
mkfs.ext4 -F "${DISK}" -L ROOT
挂载-v“${DISK}”/mnt
```
2.**XFS**
```
mkfs.xfs -f“${DISK}”
挂载-v“${DISK}”/mnt
```
3. **JFS**
```
mkfs.jfs -f“${DISK}”
挂载-v“${DISK}”/mnt
```
4. **简单的BTRFS**
```
mkfs.btrfs -f "${DISK}" -L ROOT
挂载-v“${DISK}”/mnt
```
5. **带有子卷的 BTRFS**
```
mkfs.btrfs -f "${DISK}" -L ROOT

挂载${DISK} /mnt
btrfs 子卷创建 /mnt/@
btrfs 子卷创建 /mnt/@home
btrfs 子卷创建 /mnt/@log
btrfs 子卷创建 /mnt/@cache
btrfs 子卷创建 /mnt/@snapshots
卸载/mnt

mount -o 默认,noatime,ssd,compress=zstd:3,discard=async,space_cache=v2,commit=300,subvol=/@ ${DISK} /mnt
mkdir -p /mnt/{boot/efi,home,var/log,var/cache,.snapshots,swap}

mount -o 默认,noatime,ssd,compress=zstd:3,discard=async,space_cache=v2,commit=300,subvol=/@home ${DISK} /mnt/home
mount -o 默认,noatime,ssd,compress=zstd:3,discard=async,space_cache=v2,commit=300,subvol=/@cache ${DISK} /mnt/var/cache
mount -o 默认,noatime,ssd,compress=zstd:3,discard=async,space_cache=v2,commit=300,subvol=/@log ${DISK} /mnt/var/log
mount -o 默认,noatime,ssd,compress=zstd:3,discard=async,space_cache=v2,commit=300,subvol=/@snapshots ${DISK} /mnt/.snapshots
```
---

# ▶️ 8.准备并组装ESP（EFI）
```
mkfs.fat -F32 -I“${DEV_EFI}”
mkdir -p /mnt/boot/efi
挂载-v“${DEV_EFI}”/mnt/boot/efi
```
>💡 BIOS分区（${DEV_BIOS}）没有文件系统，不格式化，不挂载。
---

# ▶️ 9. Instalar o Void Linux 无需 chroot

1. 复制要在 chroot (/mnt) 中使用的存储库密钥（XBPS 密钥）
```
mkdir -p /A{tc, 蒸汽/xbps/xbps)
cp -rpaf /var/db/xbps/keys/*.plist /mnt/var/db/xbps/keys/
cp -fpa /etc/resolv.conf /mnt/etc/resolv.conf
```

2. 将基本系统安装到新安装的磁盘上：
```
xbps-install -Sy -R https://repo-default.voidlinux.org/current \
-r /mnt \
基本系统 btrfs-progs cryptsetup grub grub-x86_64-efi dracut linux \
linux-headers linux-firmware linux-firmware-network glibc-locales \
xtools dhcpcd openssh vim nano grc zstd xz bash-completion vpm vsv \
socklog-void wget 网络工具 tmate ncurses jfsutils xfsprogs duf 树 eza chrony
```
---

# ▶️ 10. Gerar fstab no /mnt (chroot)
```bash
# 在/mnt/etc/fstab中生成fstab
xgenfstab -U /mnt > /mnt/etc/fstab
```

```bash
# 检查是否正确生成
猫 /mnt/etc/fstab
```

# ▶️ 11.使用chroot访问已安装的系统

1. 雇佣而不是克罗特：
```
xchroot /mnt /bin/bash
```
---

# ▶️ 12. 初始设置（在 chroot 中）
```
# 配置主机名 - 定义主机名
echo void > /etc/主机名

# 配置时区 - 定义时区
ln -sfv /usr/share/zoneinfo/"${TIMEZONE}" /etc/localtime

# 配置语言环境
-i -e 's/^#\(en_.Utf-8 UTF-8\)/)/' \
-E 's/^#\pt_br.br.utf-8 UTF-8\)/'\'\
/etc/default/libc-区域设置

# 生成语言环境
xbps-重新配置-f glibc-区域设置

# 修复 /var/service 符号链接中可能存在的错误（重要）：
rm -f /var/服务
ln -sf /etc/runit/runsvdir/default /var/service

# 激活一些服务
ln -sf /etc/sv/dbus /var/service/
ln -sf /etc/sv/dhcpcd /var/service/
ln -sf /etc/sv/sshd /var/service/
ln -sf /etc/sv/nanoklogd /var/service/
ln -sf /etc/sv/socklog-unix /var/service/
ln -sf /etc/sv/chronyd /var/service/

# 配置 sudo -wheel 组（可选，但推荐）
cat << 'EOF' > /etc/sudoers.d/g_wheel
%wheel ALL=(全部:全部) NOPASSWD: 全部
EOF
#所需权限
chmod 440 /etc/sudoers.d/g_wheel
```

## 创建和配置用户

⚠️ **重要：** 在下面定义真实的用户名。
```bash
导出 NEWUSER=your_user_here
```

创建具有主目录、基本组和 Bash shell 的用户：
```bash
useradd -m -G 音频、视频、wheel、tty -s /bin/bash ${NEWUSER}
```

设置您的用户密码（***重要***）
```bash
密码${NEWUSER}
```

设置 root 用户密码（***重要***）
```bash
密码根
```

将 root 用户的默认 shell 更改为 Bash
```bash
chsh -s /bin/bash 根目录
```
---

# ▶️ 13.配置UUID
⚠️ **重要：**
- 获取分区的UUID：
```
UUID_LUKS=$(blkid -s UUID -o 值“${DEV_RAIZ}”)
UUID_ROOT=$(blkid -s UUID -o 值“${DISK}”)
UUID_EFI=$(blkid -s UUID -o 值“${DEV_EFI}”)
```
---

# ▶️ 14. 创建支持休眠的交换文件（可选）

### 重要提示
```
- Btrfs 中的交换文件总是显示为 **prealloc**，这是正常的。
- 它不需要是 RAM 的完整大小。
- 大多数情况下，60% 足以用于休眠。
- 对于重负载 → 使用 70% 或 80%。
```

1.自动计算最佳交换文件大小
```
# 现代建议的休眠：总 RAM 的 60%
SWAP_GB=$(LC_ALL=C awk '/MemTotal/ {print int($2 * 0.60 / 1024 / 1024)}' /proc/meminfo)
echo "推荐的交换文件：${SWAP_GB}G"
```

```
SWAP_GB=4
echo "用户定义的交换文件：${SWAP_GB}G"
```
2. 为交换文件创建目录
```
mkdir -p /交换
swapoff -a 2>/dev/null
rm -f /交换/交换文件
```
3.禁用COW（Btrfs中需要）
```
chattr +C /交换
```
4. 创建具有先前定义大小的交换文件
```

chmod 600 /交换/交换文件
```
5.格式化swap文件并激活swap
```
mkswap /交换/交换文件
交换/交换/交换文件
```
6. 获取偏移量：
```
# 安装filefrag包
xbps-安装-Sy e2fsprogs

# 获取偏移量
偏移量=$(filefrag -v /swap/swapfile | awk '/^ *0:/{print $4}')
```
---

# ▶️    15. Configurar o GRUB
⚠️ **重要：**
> 这个块很聪明：

- 检测您是否使用休眠创建了交换文件
- 调整 /etc/default/grub 而不重复任何内容
- 仅当缺少必要的行时才创建它们
- 如果没有必要，不要改变任何东西

完全使用下面的块：
```
HAS_RESUME=假
HAS_LUKS=假

[[ -n "${offset}" ]] && HAS_RESUME=true
[[“${DISK}”=“${DEV_LUKS}”]] && HAS_LUKS=true

# 为了安全起见，删除旧线
sed -i '/^[[:空格:]]*GRUB_CMDLINE_LINUX_DEFAULT=/d' /etc/default/grub

# GRUB_CMDLINE_LINUX

# 基值
BASE=“日志级别=4”

# 添加摘要
如果$HAS_RESUME；然后
BASE =“$BASE恢复= UUID = ${UUID_ROOT}恢复_偏移= ${offset}”
是

# 添加幸运
如果$HAS_LUKS；然后
grep -q '^GRUB_ENABLE_CRYPTODISK=y' /etc/default/grub || echo 'GRUB_ENABLE_CRYPTODISK=y' >> /etc/default/grub
grep -q '^GRUB_PRELOAD_MODULES=' /etc/default/grub || echo 'GRUB_PRELOAD_MODULES="luks cryptodisk gcry_rijndael"' >> /etc/default/grub
BASE="$BASE rd.luks.uuid=${UUID_LUKS} rd.luks.name=${UUID_LUKS}=cryptroot root=/dev/mapper/cryptroot"
是

# 正确重新创建最后一行
echo "GRUB_CMDLINE_LINUX_DEFAULT=\"${BASE}\"" >> /etc/default/grub
```
---

# ▶️ 16. 重新创建 initrd
⚠️ **重要：**
```
mods=(/usr/lib/modules/*)
KVER=$(基本名称“${mods[0]}”)
回显${KVER}
dracut --force --kver ${KVER}
```
---

# ▶️ 17. 创建密钥文件以避免在启动时两次询问密码（仅限 LUKS）
> 如果系统不使用 LUKS，请跳过此步骤。
```
如果[“${DISK}”=“${DEV_LUKS}”];然后
echo“检测到LUKS：正在创建用于自动解锁的密钥文件...”

# 创建安全密钥文件
dd if=/dev/urandom of=/boot/volume.key bs=64 count=1
chmod 000 /boot/volume.key

# 将密钥文件添加到 LUKS（会询问您当前的密码）
cryptsetup luksAddKey "${DEV_RAIZ}" /boot/volume.key

# 配置/etc/crypttab
猫 << EOF >> /etc/crypttab
cryptroot ${DEV_RAIZ} /boot/volume.key luks
EOF

# 在 initramfs 中包含 keyfile 和 crypttab
mkdir -p /etc/dracut.conf.d
cat << EOF >> /etc/dracut.conf.d/10-crypt.conf
install_items+=" /boot/volume.key /etc/crypttab "
EOF

# 重新生成具有密钥文件支持的 initramfs
xbps-重新配置-fa
别的
echo“没有 LUKS 的系统：跳过密钥文件创建。”
是
```

# ▶️ 18.在**BIOS**和**UEFI**中安装GRUB（真正的混合）
1. 安装适用于 BIOS 的 GRUB（旧版）
```
grub-install --target=i386-pc ${DEVICE}
```
2.为UEFI安装GRUB
```
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=void
```
3. 创建 UEFI 回退（通用启动）。即使 NVRAM 被擦除，该文件也能保证启动。
```
mkdir -p /boot/efi/EFI/BOOT
cp -f /boot/efi/EFI/void/grubx64.efi /boot/efi/EFI/BOOT/BOOTX64.EFI
```
4.生成最终的GRUB文件
```
grub-mkconfig -o /boot/grub/grub.cfg
```

---

# ▶️ 19. 自定义用户设置：

1、环境设置：

```
# 自定义/etc/xbps.d/00-repository-main.conf
mkdir -pv /etc/xbps.d
cat << 'EOF' >> /etc/xbps.d/00-repository-main.conf
存储库=https://repo-fastly.voidlinux.org/current
#repository=https://repo-fastly.voidlinux.org/current/nonfree
#repository=https://repo-fastly.voidlinux.org/current/multilib
#repository=https://repo-fastly.voidlinux.org/current/multilib/nonfree

存储库=https://void.chililinux.com/voidlinux/current
#repository=https://void.chililinux.com/voidlinux/current/extras
#repository=https://void.chililinux.com/voidlinux/current/nonfree
#repository=https://void.chililinux.com/voidlinux/current/multilib
#repository=https://void.chililinux.com/voidlinux/current/multilib/nonfree
EOF

# 自定义/etc/rc.conf。设置控制台的默认时区、键盘布局和字体。根据需要进行更改。
猫 << EOF >> /etc/rc.conf
时区=“${TIMEZONE}”
KEYMAP="${KEYMAP}"
FONT=Lat2-Terminus16
EOF

# 自定义 root 的 .bashrc
wget --quiet --no-check-certificate \
-O /etc//skel/.bashrc \
“辣椒_REF_0_辣椒
chown root:root /etc/skel/.bashrc
chmod 644 /etc/skel/.bashrc

猫 << 'EOF' > /etc/skel/.bash_profile
# ~/.bash_profile — 将 .bashrc 加载到 Void 中

# 如果.bashrc存在，则加载
如果 [ -f ~/.bashrc ];然后
源~/.bashrc
是
EOF

# 复制到root和用户
for d in /root "/home/${NEWUSER}";做
cp -f /etc/skel/.bash_profile“$d/”
cp -f /etc/skel/.bashrc "$d/"
完毕

chown "${NEWUSER}:${NEWUSER}" "/home/${NEWUSER}/.bash_profile" "/home/${NEWUSER}/.bashrc"
chmod 644“/home/${NEWUSER}/.bash_profile”“/home/${NEWUSER}/.bashrc”

# 下载自定义 svlogtail
wget --quiet --no-check-certificate \
-O /usr/bin/svlogtail\
“辣椒_REF_0_辣椒
chmod +x /usr/bin/svlogtail
```

2.配置ssh（可选，但推荐）：
```
mkdir -pv /etc/ssh/sshd_config.d/
cat << 'EOF' > /etc/ssh/sshd_config.d/10-custom.conf
允许 TTY 是
PrintMotd 是
打印最后日志 是
横幅 /etc/issue.net

允许根登录 是
KbdInteractiveAuthentication 是
X11转发 是
公钥验证 是
PubkeyAcceptedKeyTypes=+ssh-rsa
AuthorizedKeysFile .ssh/authorized_keys
密码验证 是
挑战响应身份验证 是
使用PAM 是

子系统 sftp 内部 sftp
EOF
```
---

# ▶️ 20.启用ZRAM（可选）
Void Linux 使用 zramen 服务来启用 ZRAM，创建一个压缩内存块，以减少 SSD 交换使用并提高负载下的性能。
1.安装zramen
```
xbps-install -Sy zramen
```
2.配置ZRAM（推荐配置）：
```
猫 << 'EOF' > /etc/zramen.conf
zram_fraction=0.5
zram_设备=1
zram_算法=zstd
EOF
```
3.激活runit中的服务
```
ln-s
```
> ZRAM 将在每次启动时自动激活

---

# ▶️ 21.完成安装
1. 执行chroot：
```
出口
```
2. 卸载挂载在 /mnt 上的所有分区（子卷和 /boot/efi）：
```
卸载-R /mnt
```
3. 禁用已在 chroot 中激活的任何交换文件或交换分区：
```
交换-a
```
4、重启物理机或VM来测试实际启动情况：
```
重新启动
```
> 不要忘记取出安装介质并从新安装的光盘启动。
享受！

---

# 🎉 完整、混合、面向未来的系统
- 启动 BIOS + UEFI
- 后备 UEFI
- 带快照的 Btrfs（Snapper/Timeshift 就绪）
- 真正的休眠与交换文件
- 兹拉姆的表现

此 SSD 可启动**地球上的任何机器**。

# 免责声明

```
本教程是免费的：您可以根据需要使用、复制、修改和重新分发。
内容根据 **MIT 许可证**提供，并且可能包括源自受其自身许可证约束的开源软件的片段或命令。

不提供任何保证——此处的所有内容均“按原样”交付。
使用风险自负。作者、贡献者和 Void Linux 均不对使用本材料的损失、损坏、系统故障或任何后果负责。

如果您愿意，您可以获取源代码、查看、改编并生成您自己的本教程版本。
```

