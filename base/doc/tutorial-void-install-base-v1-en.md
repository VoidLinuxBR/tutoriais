# 🔥 Base Void Linux Installation Tutorial

# Before you start

This tutorial describes a **manual installation of Void Linux**, using direct disk partitioning, `chroot` and explicit system configuration.
It **is not an automatic installer**.

## ⚠️ Read carefully

- This guide **assumes familiarity with Linux**, terminal and basic system concepts (disks, partitions, boot, services).
- Several commands **permanently delete data** (`parted`, `mkfs`, `umount -R`).
- An error when defining the disk (`/dev/sdX`, `/dev/nvmeX`) can result in **total data loss**.
- Read **the entire tutorial before executing any command**.

## 🖥️ Recommended environment

- **VM (VirtualBox, QEMU, KVM, etc.)** for testing and learning.
- Dedicated hardware **no important data**.
- Laboratory environment or conscious facility.

❌ **Not recommended** for direct use in production without adaptations.

## 🔐 About security

During the installation process, some settings **prioritize practicality**, not security:
- `root` user login via SSH can be temporarily enabled.
- Password authentication may be active.
- Legacy compatibility (e.g. `ssh-rsa`) may be allowed.

👉 **These settings must be reviewed after installation**, especially on systems exposed to the network.

## 🧠 Important to know

- Execute the commands **one by one**, checking the output.
- Adjust disk names, network interfaces and users according to your system.
- **Do not blindly copy and paste**.
- If in doubt, **stop** and review the current step.

## 🛠️ In case of error

If something goes wrong:
- Do not restart blindly.
- Reassemble the partitions.
- Log back into the system with `chroot`.
- Check GRUB, EFI and `initramfs`.

Making mistakes is part of it. Understanding the error is what separates user from operator.

---

> This guide is aimed at users who prefer **full control** over the installation, following the classic Unix approach:
> **understand → configure → validate → continue**.

## Start Installation
Start with the Void Linux ISO (x86_64 glibc or musl).

1. Sign in as root
```bash
login    : root
password : voidlinux
```
2. Switch shell from *sh* to *bash*.
*dash/sh* **DOES NOT implement** several features that many scripts use.
```bash
bash
```
3. Change the keyboard layout to **ABNT2**, ensuring correct mapping of accents and symbols:
```bash
loadkeys br-abnt2
```

4. Connect to the Internet
- For **Wi-Fi** *(if on cable, skip this step)*:
```bash
wpa_passphrase "WIFI_NETWORK_NAME" "NETWORK_PASSWORD" > wifi.conf
wpa_supplicant -B -i wlan0 -c wifi.conf
dhcpcd wlan0
```
> 📌 **Note:** `wlan0` may vary (`wlp2s0`, `wlp0s3`, etc.).
> Use the command below to identify the correct interface:
>
> ```bash
> ip -br a
> ```

5. Test the connection:
```bash
ping -c3 8.8.8.8
ping -c3 repo-default.voidlinux.org
```

## Enable **root** user login via SSH (optional).
This step only applies when the system is running in a VM; in case of local boot (without VM), the installation can proceed normally through the local terminal.
- This is necessary to access the **VM from the host** and continue the installation remotely; after that, commands can be pasted/executed directly in the terminal via SSH.

1. Configure ssh
```bash
echo 'PermitRootLogin yes' >> /etc/ssh/sshd_config
```
2. Restart the ssh service
```bash
sv restart sshd
```

3. View network interface IP
```bash
ip -4 route get 1.1.1.1 | awk '{print $7}'
```
>Note down the IP of the network interface and use it to connect to the VM via SSH.

4. Access the VM via SSH from the host.
```bash
sudo ssh <ip-da-vm>
```
> Default password: `voidlinux`

## Configure a colored prompt in the terminal (optional)
It will display user, host, current path and the status of the last command:
```bash
export PROMPT_COMMAND='RET=$?'
export PS1='\[\e[1;33m\]\u\[\e[0m\]@\[\e[1;35m\]\h\[\e[0m\]:\[\e[0;37m\]\w\[\e[0m\] \[\e[1;32m\]$( [ $RET -eq 0 ] && printf ✔ || printf "\e[1;31m✘$RET" )\[\e[0m\] \$ '
```
> 📌 This prompt is only valid for the current session; to make it permanent add it to `.bashrc`.

## Install required packages
⚠️ **IMPORTANT:**
```bash
xbps-install -Sy xbps parted nano vim zstd xz bash-completion
```

## Partition the disk
1. Identify the disk
```bash
fdisk -l | grep -E '^(Disk|Disk) '
```
> We will assume for the tutorial `/dev/sda`

2. Adjust the variables below according to the disk that will be used (**IMPORTANT**):
```bash
# SATA/SCSI disks (sdX)
export DEVICE=/dev/sda
export DEV_EFI=${DEVICE}2
export DEV_ROOT=${DEVICE}3
```

> 📌 **Note:**
> For **NVMe** disks, the partition suffix changes (`p`):
> ```bash
> export DEVICE=/dev/nvme0n1
> export DEV_EFI=${DEVICE}p2
> export DEV_RAIZ=${DEVICE}p3
> ```

3. Partition the disk using **parted** (automatic mode).
This scheme creates:
- BIOS partition (bios_grub)
- EFI Partition (ESP)
- Root partition (ROOT)
```bash
wipefs -a "${DEVICE}"
parted --script "${DEVICE}" -- \
mklabel gpt \
mkpart primary 1MiB 2MiB name 1 BIOS set 1 bios_grub on \
mkpart primary fat32 2MiB 514MiB name 2 EFI set 2 esp on \
mkpart primary 514MiB 100% name 3 ROOT \
align-check optimal 1 \
align-check optimal 2 \
align-check optimal 3
parted --script "${DEVICE}" -- print
```

## Format partitions
```bash
# Format the root partition (ext4)
mkfs.ext4 -F ${DEV_RAIZ}

# Format the EFI partition (FAT32)
mkfs.fat -F32 -I ${DEV_EFI}
```

## Mount the volumes in `/mnt`
```bash
# Mount the root partition
mount ${DEV_RAIZ} /mnt

# Create the necessary mount points
/t /mnt/{hame,boot/efi,var/log,var/cache, proced, proc, Proc,)

# Mount the EFI partition
mount ${DEV_EFI} /mnt/boot/efi
```

## Install the base system
Installs the Void Linux base system into the `/mnt` mounted environment, including kernel, firmware, bootloader, networking and essential tools.
```bash
xbps-install -Sy -R https://repo-default.voidlinux.org/current \
-r /mnt \
base-system e2fsprogs grub-x86_64-efi dracut linux \
linux-headers linux-firmware linux-firmware-network glibc-locales \
xtools dhcpcd openssh vim nano grc zstd xz bash-completion vpm vsv \
socklog-void wget net-tools tmate ncurses chrony
```

> 📌 **Note:**
> - `grub-x86_64-efi` → bootloader UEFI
> - `linux` → kernel
> - `linux-firmware-network` → network drivers
> - `xtools` → required to use `xgenfstab` without fail

## Create `fstab`
Automatically generates the system's permanent mount file.
```bash
xgenfstab -U /mnt > /mnt/etc/fstab
```

## Entrar and system (chroot)
Access the system installed at `/mnt` to continue configuration.
```bash
xchroot /mnt /bin/bash
```

## Generate INITRAMFS
Dracut configuration for virtualization environments (VM-safe)
```bash
cat > /etc/dracut.conf.d/99-vm-safe.conf << 'EOF'
# /etc/dracut.conf.d/99-vm-safe.conf
hostonly=no
compress="gzip"
add_drivers+=" virtio virtio_pci virtio_blk virtio_net virtio_scsi "
EOF
```

Automatically detects the installed kernel version and generates the corresponding `initramfs` using **dracut**.
```bash
mods=(/usr/lib/modules/*)
KVER=$(basename "${mods[0]}")
echo ${KVER}
dracut --force --kver ${KVER}
```

## Configure GRUB

> 📌 Both methods (BIOS and UEFI) are installed on purpose.
> This allows the same disk to boot on both **Legacy BIOS** and **UEFI** systems, increasing portability between machines.

1. Create the GRUB support directory:
```bash
mkdir -p /boot/grub
```

2. Install GRUB for **BIOS (Legacy)**:
```bash
grub-install --target=i386-pc ${DEVICE}
```

3. Install GRUB for **UEFI**:
```bash
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=void
```

4. Create UEFI fallback (universal boot).
This file guarantees booting even if NVRAM is erased:
```bash
mkdir -p /boot/efi/EFI/BOOT
cp -f /boot/efi/EFI/void/grubx64.efi /boot/efi/EFI/BOOT/BOOTX64.EFI
```

5. Generate the final GRUB configuration file:
```bash
grub-mkconfig -o /boot/grub/grub.cfg
```

## Creating and configuring users

⚠️ **IMPORTANT:** define the real user name below.
```bash
export NEWUSER=your_user_here
```

Create the user with home directory, basic groups and Bash shell:
```bash
useradd -m -G audio,video,wheel,tty -s /bin/bash ${NEWUSER}
```

Set your user password (***IMPORTANT***)
```bash
passwd ${NEWUSER}
```

Set root user password (***IMPORTANT***)
```bash
passwd root
```

Change root user's default shell to Bash
```bash
chsh -s /bin/bash root
```

## Basic settings
```bash
# Setar Hostname
echo void > /etc/hostname

# Set Localtime
ln -sf /usr/share/zoneinfo/America/Sao_Paulo /etc/localtime

# Setar Local
sed -i 's/#en_US.UTF-8/en_US.UTF-8/' /etc/default/libc-locales
sed -i 's/#pt_BR.UTF-8/pt_BR.UTF-8/' /etc/default/libc-locales

# Generate locales:
xbps-reconfigure -f glibc-locales

# Fix possible error in /var/service symlink (important):
rm -f /var/service
ln -sf /etc/runit/runsvdir/default /var/service

# Activate some services:
ln -sf /etc/sv/dbus /var/service/
ln -sf /etc/sv/dhcpcd /var/service/
ln -sf /etc/sv/sshd /var/service/
ln -sf /etc/sv/nanoklogd /var/service/
ln -sf /etc/sv/socklog-unix /var/service/
ln -sf /etc/sv/chronyd /var/service/

# download custom svlogtail (optional, but recommended):
wget --quiet --no-check-certificate -O /usr/bin/svlogtail \
"https://raw.githubusercontent.com/voidlinux-br/void-install/refs/heads/main/svlogtail" && \
chmod +x /usr/bin/svlogtail

# Create a resolv.conf
printf 'nameserver 1.1.1.1\nnameserver 8.8.8.8\n' > /etc/resolv.conf

#Configure sudo - wheel group (optional, but recommended)
cat << 'EOF' > /etc/sudoers.d/g_wheel
%wheel ALL=(ALL:ALL) NOPASSWD: ALL
EOF

#Required permissions
chmod 440 /etc/sudoers.d/g_wheel
```

## Customize `/etc/xbps.d/00-repository-main.conf`
*(Optional, but recommended)*

Creates the **XBPS** configuration directory (if it does not already exist) and defines a list of official and alternative repositories.
**repo-fastly** repositories tend to have better latency.

```bash
mkdir -pv /etc/xbps.d

cat << 'EOF' > /etc/xbps.d/00-repository-main.conf
# Official repository (Fastly – best latency)
repository=https://repo-fastly.voidlinux.org/current
#repository=https://repo-fastly.voidlinux.org/current/nonfree
#repository=https://repo-fastly.voidlinux.org/current/multilib
#repository=https://repo-fastly.voidlinux.org/current/multilib/nonfree

# Alternative repository (Chili Linux)
repository=https://void.chililinux.com/voidlinux/current
#repository=https://void.chililinux.com/voidlinux/current/extras
#repository=https://void.chililinux.com/voidlinux/current/nonfree
#repository=https://void.chililinux.com/voidlinux/current/multilib
#repository=https://void.chililinux.com/voidlinux/current/multilib/nonfree
EOF
```

## Customize `/etc/rc.conf`
Sets the console's default time zone, keyboard layout, and font.
Change as needed.
```bash
cat << 'EOF' > /etc/rc.conf
TIMEZONE=America/Sao_Paulo
KEYMAP=br-abnt2
FONT=Lat2-Terminus16
EOF
```

Virtio modules (virtual machine).
```bash
cat > /etc/modules-load.d/virtio.conf << 'EOF'
virtio
virtio_pci
virtio_net
virtio_blk
virtio_scsi
EOF
```

## Customize user `.bashrc`
Creates a default `.bash_profile` and ensures that `.bashrc` is automatically loaded at login.
> ⚠️ Make sure the user was created in the previous step.

```bash
# Download default .bashrc to /etc/skel
wget --quiet --no-check-certificate \
-O /etc/skel/.bashrc \
"https://raw.githubusercontent.com/voidlinux-br/void-install/refs/heads/main/.bashrc"

chown root:root /etc/skel/.bashrc
chmod 644 /etc/skel/.bashrc

# Create default .bash_profile
cat << 'EOF' > /etc/skel/.bash_profile
# ~/.bash_profile — loads .bashrc into Void

# If .bashrc exists, load
if [ -f ~/.bashrc ]; then
source ~/.bashrc
be
EOF

# Copy to root and user
for d in /root "/home/${NEWUSER}"; do
cp -f /etc/skel/.bash_profile "$d/"
cp -f /etc/skel/.bashrc "$d/"
done

# Adjust user permissions
chown "${NEWUSER}:${NEWUSER}" \
"/home/${NEWUSER}/.bash_profile" \
"/home/${NEWUSER}/.bashrc"

chmod 644 \
"/home/${NEWUSER}/.bash_profile" \
"/home/${NEWUSER}/.bashrc"
```

## Configure SSH
*(Optional, but recommended)*

Creates a supplementary configuration file for **sshd**, leaving the main file intact.
```bash
mkdir -pv /etc/ssh/sshd_config.d

cat << 'EOF' > /etc/ssh/sshd_config.d/10-custom.conf
# General settings
PermitTTY yes
PrintMotd yes
PrintLastLog yes
Banner /etc/issue.net

# Authentication
PermitRootLogin yes
PasswordAuthentication yes
KbdInteractiveAuthentication yes
ChallengeResponseAuthentication yes
PubkeyAuthentication yes
PubkeyAcceptedKeyTypes=+ssh-rsa
AuthorizedKeysFile .ssh/authorized_keys
UsePAM yes

# Features
X11Forwarding yes
Subsystem sftp internal-sftp
EOF
```
> ⚠️ It is recommended to review and harden these SSH settings after the first boot, especially on systems exposed to the Internet.

## Finalize installation
Sair do chroot
```bash
exit
```
Unmount all partitions mounted on `/mnt` (including subvolumes and `/boot/efi`):
```bash
umount -R /mnt
```
Restart the physical machine or VM to test the actual boot:
```bash
reboot
```
> 📌 **Note: Don't forget to remove the installation media.

# 🎉 Enjoy!
**Void Linux** is now installed and ready to use.

# DISCLAIMER

> This tutorial is free: you can use, copy, modify and redistribute as you wish.
> Content is made available under the **MIT License** and may include excerpts or commands derived from open source software, subject to their own licenses.
>
> No guarantees are provided — everything here is delivered **“as is”**.
> Use at your own risk. Neither the author, nor contributors, nor Void Linux are responsible for loss, damage, system failures or any consequences of the use of this material.
>
> You are free to review, adapt and generate your own version of this tutorial.

