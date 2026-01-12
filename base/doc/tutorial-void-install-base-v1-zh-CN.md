# 🔥 Base Void Linux 安装教程

# 开始之前

本教程描述了使用直接磁盘分区、“chroot”和显式系统配置的 **手动安装 Void Linux**。
它**不是自动安装程序**。

## ⚠️仔细阅读

- 本指南**假设熟悉 Linux**、终端和基本系统概念（磁盘、分区、引导、服务）。
- 多个命令**永久删除数据**（`parted`、`mkfs`、`umount -R`）。
- 定义磁盘（`/dev/sdX`、`/dev/nvmeX`）时出现错误可能会导致**全部数据丢失**。
- 在执行任何命令之前阅读**整个教程**。

## 🖥️推荐环境

- **VM（VirtualBox、QEMU、KVM 等）** 用于测试和学习。
- 专用硬件**无重要数据**。
- 实验室环境或有意识的设施。

❌ **不建议** 不经修改直接用于生产。

## 🔐关于安全

安装过程中，有些设置**优先考虑实用性**，而不是安全性：
- 可以暂时启用通过 SSH 的“root”用户登录。
- 密码验证可能处于活动状态。
- 可能允许旧版兼容性（例如 `ssh-rsa`）。

👉 **安装后必须检查这些设置**，尤其是在暴露于网络的系统上。

## 🧠 重要了解

- 执行命令**一一**，检查输出。
- 根据您的系统调整磁盘名称、网络接口和用户。
- **不要盲目复制粘贴**。
- 如有疑问，**停止**并检查当前步骤。

## 🛠️ 如果出现错误

如果出现问题：
- 不要盲目重启。
- 重新组装分区。
- 使用“chroot”重新登录系统。
- 检查 GRUB、EFI 和 `initramfs`。

犯错误是其中的一部分。理解错误是用户与操作员的区别。

---

> 本指南针对喜欢**完全控制**安装的用户，遵循经典的 Unix 方法：
> **理解→配置→验证→继续**。

## 开始安装
从 Void Linux ISO（x86_64 glibc 或 musl）开始。

1. 以 root 身份登录
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

4. 连接互联网
- 对于 **Wi-Fi** *（如果使用电缆，请跳过此步骤）*：
```bash
wpa_passphrase“WIFI_NETWORK_NAME”“NETWORK_PASSWORD”> wifi.conf
wpa_supplicant -B -i wlan0 -c wifi.conf
dhcpcd 无线局域网0
```
> 📌 **注意：** `wlan0` 可能会有所不同（`wlp2s0`、`wlp0s3` 等）。
> 使用以下命令来识别正确的接口：
>
> ```bash
> ip -br a
> ```

5. 测试连接：
```bash
平-c3 8.8.8.8
ping -c3 repo-default.voidlinux.org
```

## 通过 SSH 启用 **root** 用户登录（可选）。
此步骤仅适用于系统运行在虚拟机中的情况；如果是本地启动（没有VM），则可以通过本地终端正常安装。
- 这是从主机访问 **VM 并继续远程安装所必需的；之后，可以通过 SSH 直接在终端中粘贴/执行命令。

1.配置ssh
```bash
echo 'PermitRootLogin 是' >> /etc/ssh/sshd_config
```
2.重启ssh服务
```bash
sv 重新启动 sshd
```

3.查看网络接口IP
```bash
ip -4 路由获取 1.1.1.1 | awk '{print $7}'
```
>记下网络接口的IP并使用它通过SSH连接到VM。

4. 从主机通过 SSH 访问虚拟机。
```bash
sudo ssh <ip-da-vm>
```
> 默认密码：`voidlinux`

## 在终端中配置彩色提示（可选）
它将显示用户、主机、当前路径和最后一个命令的状态：
```bash
导出 PROMPT_COMMAND='RET=$?'
导出 PS1='\[\e[1;33m\]\u\[\e[0m\]@\[\e[1;35m\]\h\[\e[0m\]:\[\e[0;37m\]\w\[\e[0m\] \[\e[1;32m\]$( [ $RET -eq 0 ] && printf ✔ || printf "\e[1;31m✘$RET" )\[\e[0m\] \$ '
```
> 📌此提示仅对当前会话有效；要使其永久，请将其添加到“.bashrc”。

## 安装需要的包
⚠️ **重要：**
```bash
xbps-install -Sy xbps parted nano vim zstd xz bash-completion
```

## 对磁盘进行分区
1.识别磁盘
```bash
fdisk -l | fdisk -l | grep -E '^(磁盘|磁盘)'
```
> 我们将假设教程为 `/dev/sda`

2. 根据将使用的磁盘调整以下变量（**重要**）：
```bash
# SATA/SCSI 磁盘 (sdX)
导出设备=/dev/sda
导出 DEV_EFI=${DEVICE}2
导出 DEV_ROOT=${DEVICE}3
```

> 📌 **注意：**
> 对于 **NVMe** 磁盘，分区后缀更改 (`p`)：
> ```bash
> 导出设备=/dev/nvme0n1
> 导出 DEV_EFI=${DEVICE}p2
> 导出 DEV_RAIZ=${DEVICE}p3
> ```

3. 使用**parted**（自动模式）对磁盘进行分区。
该方案创建：
- BIOS 分区 (bios_grub)
- EFI 分区（ESP）
- 根分区（ROOT）
```bash
wipefs-a“${DEVICE}”
分开--脚本“${DEVICE}”--\
mklabel gpt \
mkpart 主要 1MiB 2MiB 名称 1 BIOS 设置 1 BIOS_grub 位于 \
mkpart 主要 fat32 2MiB 514MiB 名称 2 EFI 设置 2 esp 位于 \
mkpart 主要 514MiB 100% 名称 3 ROOT \
对齐检查最优 1 \
对齐检查最优 2 \
对齐检查最优 3
分开--脚本“${DEVICE}”--打印
```

## 格式化分区
```bash
# 格式化根分区（ext4）
mkfs.ext4 -F ${DEV_RAIZ}

# 格式化EFI分区（FAT32）
mkfs.fat -F32 -I ${DEV_EFI}
```

## 将卷挂载到 `/mnt` 中
```bash
# 挂载根分区
挂载${DEV_RAIZ} /mnt

# 创建必要的挂载点
/t /mnt/{hame,boot/efi,var/log,var/cache, proced, proc, Proc,)

#挂载EFI分区
挂载 ${DEV_EFI} /mnt/boot/efi
```

## 安装基础系统
将Void Linux基础系统安装到`/mnt`安装环境中，包括内核、固件、引导加载程序、网络和基本工具。
```bash
xbps-install -Sy -R https://repo-default.voidlinux.org/current \
-r /mnt \
基本系统 e2fsprogs grub-x86_64-efi dracut linux \
linux-headers linux-firmware linux-firmware-network glibc-locales \
xtools dhcpcd openssh vim nano grc zstd xz bash-completion vpm vsv \
socklog-void wget 网络工具 tmate ncurses chrony
```

> 📌 **注意：**
> - `grub-x86_64-efi` → 引导加载程序 UEFI
> - `linux` → 内核
> - `linux-firmware-network` → 网络驱动程序
> - `xtools` → 必须使用 `xgenfstab` 才能成功

## 创建`fstab`
自动生成系统的永久挂载文件。
```bash
xgenfstab -U /mnt > /mnt/etc/fstab
```

## 入口和系统（chroot）
访问安装在`/mnt`的系统以继续配置。
```bash
xchroot /mnt /bin/bash
```

## 生成 INITRAMFS
虚拟化环境的 Dracut 配置（VM 安全）
```bash
猫 > /etc/dracut.conf.d/99-vm-safe.conf << 'EOF'
# /etc/dracut.conf.d/99-vm-safe.conf
仅主机=否
压缩=“gzip”
add_drivers+=“virtio virtio_pci virtio_blk virtio_net virtio_scsi”
EOF
```

自动检测安装的内核版本并使用 **dracut** 生成相应的 `initramfs`。
```bash
mods=(/usr/lib/modules/*)
KVER=$(基本名称“${mods[0]}”)
回显${KVER}
dracut --force --kver ${KVER}
```

## 配置GRUB

> 📌 两种方法（BIOS 和 UEFI）都是故意安装的。
> 这允许同一个磁盘在 **Legacy BIOS** 和 **UEFI** 系统上启动，从而提高机器之间的可移植性。

1.创建GRUB支持目录：
```bash
mkdir -p /启动/grub
```

2. 安装 **BIOS（旧版）** 的 GRUB：
```bash
grub-install --target=i386-pc ${DEVICE}
```

3. 为 **UEFI** 安装 GRUB：
```bash
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=void
```

4. 创建 UEFI 后备（通用启动）。
即使 NVRAM 被擦除，该文件也能保证启动：
```bash
mkdir -p /boot/efi/EFI/BOOT
cp -f /boot/efi/EFI/void/grubx64.efi /boot/efi/EFI/BOOT/BOOTX64.EFI
```

5. 生成最终的GRUB配置文件：
```bash
grub-mkconfig -o /boot/grub/grub.cfg
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

## 基本设置
```bash
# Setar 主机名
echo void > /etc/主机名

# 设置当地时间
ln -sf /usr/share/zoneinfo/America/Sao_Paulo /etc/localtime

# 士达本地
sed -i 's/#en_US.UTF-8/en_US.UTF-8/' /etc/default/libc-locales
sed -i 's/#pt_BR.UTF-8/pt_BR.UTF-8/' /etc/default/libc-locales

# 生成语言环境：
xbps-重新配置-f glibc-区域设置

# 修复 /var/service 符号链接中可能存在的错误（重要）：
rm -f /var/服务
ln -sf /etc/runit/runsvdir/default /var/service

# 激活一些服务：
ln -sf /etc/sv/dbus /var/service/
ln -sf /etc/sv/dhcpcd /var/service/
ln -sf /etc/sv/sshd /var/service/
ln -sf /etc/sv/nanoklogd /var/service/
ln -sf /etc/sv/socklog-unix /var/service/
ln -sf /etc/sv/chronyd /var/service/

# 下载自定义 svlogtail （可选，但推荐）：
wget --quiet --no-check-certificate -O /usr/bin/svlogtail \
“https://raw.githubusercontent.com/voidlinux-br/void-install/refs/heads/main/svlogtail" && \
chmod +x /usr/bin/svlogtail

# 创建一个resolv.conf
printf '名称服务器 1.1.1.1\n名称服务器 8.8.8.8\n' > /etc/resolv.conf

#配置sudo-wheel组（可选，但推荐）
cat << 'EOF' > /etc/sudoers.d/g_wheel
%wheel ALL=(全部:全部) NOPASSWD: 全部
EOF

#所需权限
chmod 440 /etc/sudoers.d/g_wheel
```

## 自定义 `/etc/xbps.d/00-repository-main.conf`
*（可选，但推荐）*

创建 **XBPS** 配置目录（如果尚不存在）并定义官方和替代存储库的列表。
**repo-fastly** 存储库往往具有更好的延迟。

```bash
mkdir -pv /etc/xbps.d

cat << 'EOF' > /etc/xbps.d/00-repository-main.conf
# 官方存储库（快速 - 最佳延迟）
存储库=https://repo-fastly.voidlinux.org/current
#repository=https://repo-fastly.voidlinux.org/current/nonfree
#repository=https://repo-fastly.voidlinux.org/current/multilib
#repository=https://repo-fastly.voidlinux.org/current/multilib/nonfree

# 替代存储库 (Chili Linux)
存储库=https://void.chililinux.com/voidlinux/current
#repository=https://void.chililinux.com/voidlinux/current/extras
#repository=https://void.chililinux.com/voidlinux/current/nonfree
#repository=https://void.chililinux.com/voidlinux/current/multilib
#repository=https://void.chililinux.com/voidlinux/current/multilib/nonfree
EOF
```

## 自定义 `/etc/rc.conf`
设置控制台的默认时区、键盘布局和字体。
根据需要进行更改。
```bash
猫 << 'EOF' > /etc/rc.conf
TIMEZONE=美国/圣保罗
键盘映射=br-abnt2
FONT=Lat2-Terminus16
EOF
```

Virtio 模块（虚拟机）。
```bash
cat > /etc/modules-load.d/virtio.conf << 'EOF'
虚拟机
virtio_pci
virtio_net
virtio_blk
virtio_scsi
EOF
```

## 自定义用户`.bashrc`
创建默认的“.bash_profile”并确保在登录时自动加载“.bashrc”。
> ⚠️ 确保用户是在上一步中创建的。

```bash
# 下载默认的.bashrc到/etc/skel
wget --quiet --no-check-certificate \
-O /etc/skel/.bashrc \
“辣椒_REF_0_辣椒

chown root:root /etc/skel/.bashrc
chmod 644 /etc/skel/.bashrc

# 创建默认的.bash_profile
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

# 调整用户权限
chown "${NEWUSER}:${NEWUSER}" \
“/home/${NEWUSER}/.bash_profile”\
“/home/${NEWUSER}/.bashrc”

chmod 644 \
“/home/${NEWUSER}/.bash_profile”\
“/home/${NEWUSER}/.bashrc”
```

## 配置 SSH
*（可选，但推荐）*

为 **sshd** 创建补充配置文件，保持主文件不变。
```bash
mkdir -pv /etc/ssh/sshd_config.d

cat << 'EOF' > /etc/ssh/sshd_config.d/10-custom.conf
# 常规设置
允许 TTY 是
PrintMotd 是
打印最后日志 是
横幅 /etc/issue.net

＃ 验证
允许根登录 是
密码验证 是
KbdInteractiveAuthentication 是
挑战响应身份验证 是
公钥验证 是
PubkeyAcceptedKeyTypes=+ssh-rsa
AuthorizedKeysFile .ssh/authorized_keys
使用PAM 是

＃ 特征
X11转发 是
子系统 sftp 内部 sftp
EOF
```
> ⚠️ 建议在首次启动后检查并强化这些 SSH 设置，尤其是在暴露于 Internet 的系统上。

## 完成安装
Sair do chroot
```bash
出口
```
卸载挂载在 `/mnt` 上的所有分区（包括子卷和 `/boot/efi`）：
```bash
卸载-R /mnt
```
重启物理机或虚拟机测试实际启动情况：
```bash
重新启动
```
> 📌 **注意：不要忘记删除安装介质。

# 🎉 享受吧！
**Void Linux** 现已安装并可以使用。

# 免责声明

> 本教程是免费的：您可以根据需要使用、复制、修改和重新分发。
> 内容根据 **MIT 许可证** 提供，并且可能包括源自开源软件的摘录或命令，但须遵守其自己的许可证。
>
> 不提供任何保证 — 这里的所有内容均按 **“按原样”** 交付。
> 使用时需自行承担风险。作者、贡献者和 Void Linux 均不对使用本材料的损失、损坏、系统故障或任何后果负责。
>
> 您可以自由查看、改编和生成本教程的您自己的版本。

