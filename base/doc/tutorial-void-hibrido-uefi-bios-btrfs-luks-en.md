# 🧩 VOID LINUX TUTORIAL — HYBRID INSTALLATION (UEFI + BIOS) WITH EXT4, XFS, JFS OR BTRFS (SUBVOLUMES), LUKS, HIBERNATION AND ZRAM
### REVISED AND VALIDATED VERSION — CORRECT PARTITIONING + UNIVERSAL BOOT

This guide installs a fully **hybrid** Void Linux, capable of booting any type of machine — old, new or problematic:

- 💾 **Modern UEFI** (with normal input and fallback)
- 🧮 **BIOS/Legacy** (full compatibility)
- 🧰 **GPT with BIOS Boot (EF02)** — maximum support for old hardware
- 🚀 **Btrfs with subvolumes** (optional), ready-made snapshots
- 🔐 **LUKS1 fully compatible with GRUB**
- 🌙 **Real hibernation via swapfile**
- 🧊 **ZRAM configured for performance**
- 🧱 **Full support for EXT4, XFS, JFS and BTRFS**
- 💡 **Initramfs/GRUB automatically configured (LUKS + resume)**

📌 **No compromise, no reinstalling GRUB, no wasting time.**
📌 **Guaranteed boot even on a machine with erased NVRAM (BOOTX64.EFI fallback).**

---

# ▶️    1. Bootar o Live ISO

Suggestion: Use the glibc version for superior compatibility:
- download the iso from:
```
https://translate.google.com/translate?hl=pt-BR&sl=auto&tl=en&u=https://repo-default.voidlinux.org/live/current/void-live-x86_64-20250202-base.iso
```
- or look for the latest version at:
```
https://translate.google.com/translate?hl=pt-BR&sl=auto&tl=en&u=https://voidlinux.org/download/
```

1. Log in as root.
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

4. Paste into the terminal (optional) — Prompt with colors, user@host:path and status of the last command (✔/✘). Useful and beautiful.
```bash
export PS1='\[\e[1;32m\]\u\[\e[1;33m\]@\[\e[1;36m\]\h\[\e[1;31m\]:\w \
$([[ $? -eq 0 ]] && echo -e "\e[1;32m✔" || echo -e "\e[1;31m✘$?") \
\[\e[0m\]\$ '
```

# ▶️ 2. Connect to the Internet
- For **Wi-Fi** *(if on cable, skip this step)*:
```bash
wpa_passphrase "SSID" "PASSWORD" > wifi.conf
wpa_supplicant -B -i wlan0 -c wifi.conf
dhcpcd wlan0
```

1. Test the connection:
```bash
ping -c3 8.8.8.8
ping -c3 repo-default.voidlinux.org
```

2. Install required packages:
⚠️ **IMPORTANT:**
```bash
xbps-install -Sy xbps parted jfsutils xfsprogs nano zstd xz bash-completion
```
---

# ▶️ 3. Identify the disk
1. List the available disks and note the device name (ex: `/dev/sda`, `/dev/vda`, `/dev/nvme0n1`):
```bash
fdisk -l | grep -E '^(Disk|Disk) '
```

# ▶️ 4. Define variables used in the tutorial:
⚠️ **IMPORTANT:**

1. Define devices **BEFORE** use:
> 1. **We will assume** for the tutorial `/dev/sda` (normal) or `/dev/nvme0n1` (nvme)
> 2. **Adjust** according to your disc (choose just **one** or **another** model)

For **normal** disks (e.g. /dev/sda)
```bash
export DEVICE=/dev/sda
export DEV_BIOS=${DEVICE}1
export DEV_EFI=${DEVICE}2
export DEV_ROOT=${DEVICE}3
export DEV_LUKS=/dev/mapper/cryptroot
```
For **NVMe** disks (e.g. /dev/nvme0n1), the partition suffix changes (`p`)
```bash
export DEVICE=/dev/nvme0n1
export DEV_BIOS=${DEVICE}p1
export DEV_EFI=${DEVICE}p2
export DEV_RAIZ=${DEVICE}p3
export DEV_LUKS=/dev/mapper/cryptroot
```

> 📌 **Note:**
> DEVICE → entire disk
DEV_BIOS → BIOS boot partition (1–2 MiB, no FS, does not mount)
DEV_EFI → EFI partition (FAT32)
DEV_ROOT → root partition (normal or LUKS)
DEV_LUKS → LUKS mapping (/dev/mapper/cryptroot)

- 👉 Here you define the anatomy of the disc. Everything else in the guide just follows these variables.
- 🔎 Why is this necessary?
Because declaring everything at the beginning makes the next process typo-proof.

2. Define **KEYMAP** and **TIMEZONE** (change as needed):
```bash
export KEYMAP=br-abnt2
```
```bash
export TIMEZONE=America/Sao_Paulo
```

---

# ▶️ 5. Partition disk
> - The BIOS partition **MUST** be first. This increases compatibility with older motherboards, problematic bootloaders, and BIOSes that expect boot code in the first areas of the disk.
> - ESP can come later without any problem — UEFI doesn't care about the position.

### Ideal and correct order:

- 1️⃣ BIOS Boot (EF02)
- 2️⃣ ESP (EFI System, FAT32)
- 3️⃣ Btrfs/Ext4/Xfs/Jfs (root)

### Partition using parted (automatic)
> Here the **DEVICE** is already defined up there, so there is no “magic” variable.
```
wipefs -a "${DEVICE}"
parted --script "${DEVICE}" -- \
mklabel gpt \
mkpart primary 1MiB 2MiB name 1 BIOS set 1 bios_grub on \
mkpart primary fat32 2MiB 514MiB name 2 EFI set 2 esp on \
mkpart primary 514MiB 100% name 3 ROOT \
align-check optimal 1 \
align-check optimal 2 \
align-check optimal 1
parted --script "${DEVICE}" -- print
```
> - Partition 1 → BIOS boot (bios_grub, no FS, does not mount)
> - Partition 2 → EFI (FAT32)
> - Partition 3 → ROOT (we will format it later with EXT4/XFS/JFS/BTRFS, with or without LUKS)
> - I used mkpart primary 514MiB 100% without specifying FS precisely to avoid tying up the FS. You choose FS later.
---

# ▶️ 6. Choose the installation mode (NORMAL or LUKS)
⚠️ **IMPORTANT:**
> Choose ONLY ONE of the two blocks below.
**NOT** to run both steps.

1. NORMAL INSTALLATION **(without LUKS)**
```bash
export DISK="${DEV_RAIZ}"
```
- Sets DISK to the actual device /dev/sda3

2. INSTALLATION **WITH LUKS** (encrypted root)
```
# Encrypt ONLY the root partition on LUKS1 (GRUB compatible) - never the entire disk
# Encrypt the partition by confirming with YES:
cryptsetup luksFormat --type luks1 "${DEV_RAIZ}"

# Open the partition with your passphrase.
cryptsetup open "${DEV_RAIZ}" cryptroot

# From now on, the real root is the mapped device
export DISK="${DEV_LUKS}"
```
- LUKS sits on top of /dev/sda3, not the entire disk
- The system will be installed in /dev/mapper/cryptroot

👉 From here on, EVERYTHING uses $DISK.

---

# ▶️ 7. Create the file system (FS) and mount root
⚠️ **IMPORTANT:**
> Choose ONLY ONE of the two blocks below.

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
4. **Simple BTRFS**
```
mkfs.btrfs -f "${DISK}" -L ROOT
mount -v "${DISK}" /mnt
```
5. **BTRFS with subvolumes**
```
mkfs.btrfs -f "${DISK}" -L ROOT

mount ${DISK} /mnt
btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@log
btrfs subvolume create /mnt/@cache
btrfs subvolume create /mnt/@snapshots
umount /mnt

mount -o defaults,noatime,ssd,compress=zstd:3,discard=async,space_cache=v2,commit=300,subvol=/@ ${DISK} /mnt
mkdir -p /mnt/{boot/efi,home,var/log,var/cache,.snapshots,swap}

mount -o defaults,noatime,ssd,compress=zstd:3,discard=async,space_cache=v2,commit=300,subvol=/@home      ${DISK} /mnt/home
mount -o defaults,noatime,ssd,compress=zstd:3,discard=async,space_cache=v2,commit=300,subvol=/@cache     ${DISK} /mnt/var/cache
mount -o defaults,noatime,ssd,compress=zstd:3,discard=async,space_cache=v2,commit=300,subvol=/@log       ${DISK} /mnt/var/log
mount -o defaults,noatime,ssd,compress=zstd:3,discard=async,space_cache=v2,commit=300,subvol=/@snapshots ${DISK} /mnt/.snapshots
```
---

# ▶️ 8. Prepare and assemble the ESP (EFI)
```
mkfs.fat -F32 -I "${DEV_EFI}"
mkdir -p /mnt/boot/efi
mount -v "${DEV_EFI}" /mnt/boot/efi
```
>💡 The BIOS partition (${DEV_BIOS}) has no file system, does not format, does not mount.
---

# ▶️    9. Instalar o Void Linux no chroot

1. Copy the repository keys (XBPS keys) to be used in the chroot (/mnt)
```
mkdir -p /A{tc, vapor/xbps/xbps)
cp -rpaf /var/db/xbps/keys/*.plist /mnt/var/db/xbps/keys/
cp -fpa /etc/resolv.conf /mnt/etc/resolv.conf
```

2. Install the base system onto the newly mounted disk:
```
xbps-install -Sy -R https://repo-default.voidlinux.org/current \
-r /mnt \
base-system btrfs-progs cryptsetup grub grub-x86_64-efi dracut linux \
linux-headers linux-firmware linux-firmware-network glibc-locales \
xtools dhcpcd openssh vim nano grc zstd xz bash-completion vpm vsv \
socklog-void wget net-tools tmate ncurses jfsutils xfsprogs duf tree eza chrony
```
---

# ▶️ 10. Gerar fstab no /mnt (chroot)
```bash
# Generate fstab in /mnt/etc/fstab
xgenfstab -U /mnt > /mnt/etc/fstab
```

```bash
# check if it was generated correctly
cat /mnt/etc/fstab
```

# ▶️ 11. Access the installed system using chroot

1. Employ not croit:
```
xchroot /mnt /bin/bash
```
---

# ▶️ 12. Initial Settings (in chroot)
```
# configure hostname - defines the hostname
echo void > /etc/hostname

# configure timezone - defines the time zone
ln -sfv /usr/share/zoneinfo/"${TIMEZONE}" /etc/localtime

# configure locales
-i -e 's/^#\(en_.Utf-8 UTF-8\)/)/' \
-E 's/^#\pt_br.br.utf-8 UTF-8\)/' \' \
/etc/default/libc-locales

# generate locales
xbps-reconfigure -f glibc-locales

# Fix possible error in /var/service symlink (important):
rm -f /var/service
ln -sf /etc/runit/runsvdir/default /var/service

# Activate some services
ln -sf /etc/sv/dbus /var/service/
ln -sf /etc/sv/dhcpcd /var/service/
ln -sf /etc/sv/sshd /var/service/
ln -sf /etc/sv/nanoklogd /var/service/
ln -sf /etc/sv/socklog-unix /var/service/
ln -sf /etc/sv/chronyd /var/service/

# Configure sudo - wheel group (optional, but recommended)
cat << 'EOF' > /etc/sudoers.d/g_wheel
%wheel ALL=(ALL:ALL) NOPASSWD: ALL
EOF
#Required permissions
chmod 440 /etc/sudoers.d/g_wheel
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
---

# ▶️ 13. Configure UUIDs
⚠️ **IMPORTANT:**
- Get the UUIDs of the partitions:
```
UUID_LUKS=$(blkid -s UUID -o value "${DEV_RAIZ}")
UUID_ROOT=$(blkid -s UUID -o value "${DISK}")
UUID_EFI=$(blkid -s UUID -o value "${DEV_EFI}")
```
---

# ▶️ 14. Create swapfile with hibernation support (optional)

### Important notes
```
- Swapfile in Btrfs always appears as **prealloc**, it's normal.
- It does not need to be the full size of RAM.
- 60% is enough for hibernation in most cases.
- For heavy loads → use 70% or 80%.
```

1. Automatically calculate optimal swapfile size
```
# Modern recommendation for hibernation: 60% of total RAM
SWAP_GB=$(LC_ALL=C awk '/MemTotal/ {print int($2 * 0.60 / 1024 / 1024)}' /proc/meminfo)
echo "Recommended swapfile: ${SWAP_GB}G"
```
- or, manually set the desired size:
```
SWAP_GB=4
echo "User-defined swapfile: ${SWAP_GB}G"
```
2. Create directory for the swapfile
```
mkdir -p /swap
swapoff -a 2>/dev/null
rm -f /swap/swapfile
```
3. Disable COW (required in Btrfs)
```
chattr +C /swap
```
4. Create the swapfile with the previously defined size
```
fallocate -l ${SWAP_GB}G /swap/swapfile
chmod 600 /swap/swapfile
```
5. Format the swapfile and activate swap
```
mkswap /swap/swapfile
swapon /swap/swapfile
```
6. Get offset:
```
# Install the package for filefrag
xbps-install -Sy e2fsprogs

# Get the offset
offset=$(filefrag -v /swap/swapfile | awk '/^ *0:/{print $4}')
```
---

# ▶️ 15. Configure GRUB
⚠️ **IMPORTANT:**
> This block is smart:
- Automatically detects if you are using LUKS
- Detects if you created swapfile with hibernate
- Adjusts /etc/default/grub without duplicating anything
- Creates the necessary lines only if they are missing
- Don't change anything if it's not necessary

Use exactly the block below:
```
HAS_RESUME=false
HAS_LUKS=false

[[ -n "${offset}" ]] && HAS_RESUME=true
[[ "${DISK}" = "${DEV_LUKS}" ]] && HAS_LUKS=true

# Remove old line for safety
sed -i '/^[[:space:]]*GRUB_CMDLINE_LINUX_DEFAULT=/d' /etc/default/grub

# GRUB_CMDLINE_LINUX

# Base value
BASE="loglevel=4"

# Add summary
if $HAS_RESUME; then
BASE="$BASE resume=UUID=${UUID_ROOT} resume_offset=${offset}"
be

# Add LUKS
if $HAS_LUKS; then
grep -q '^GRUB_ENABLE_CRYPTODISK=y' /etc/default/grub || echo 'GRUB_ENABLE_CRYPTODISK=y' >> /etc/default/grub
grep -q '^GRUB_PRELOAD_MODULES=' /etc/default/grub    || echo 'GRUB_PRELOAD_MODULES="luks cryptodisk gcry_rijndael"' >> /etc/default/grub
BASE="$BASE rd.luks.uuid=${UUID_LUKS} rd.luks.name=${UUID_LUKS}=cryptroot root=/dev/mapper/cryptroot"
be

# Recreate the final line correctly
echo "GRUB_CMDLINE_LINUX_DEFAULT=\"${BASE}\"" >> /etc/default/grub
```
---

# ▶️ 16. Recreate the initrd
⚠️ **IMPORTANT:**
```
mods=(/usr/lib/modules/*)
KVER=$(basename "${mods[0]}")
echo ${KVER}
dracut --force --kver ${KVER}
```
---

# ▶️ 17. Create Keyfile to avoid asking for password twice at boot (LUKS only)
> If the system DOES NOT use LUKS, skip this step.
```
if [ "${DISK}" = "${DEV_LUKS}" ]; then
echo "LUKS detected: creating keyfile for automatic unlocking..."

# Create secure keyfile
dd if=/dev/urandom of=/boot/volume.key bs=64 count=1
chmod 000 /boot/volume.key

# Add keyfile to LUKS (will ask for your current password)
cryptsetup luksAddKey "${DEV_RAIZ}" /boot/volume.key

# Configure /etc/crypttab
cat << EOF >> /etc/crypttab
cryptroot ${DEV_RAIZ} /boot/volume.key luks
EOF

# Include keyfile and crypttab in initramfs
mkdir -p /etc/dracut.conf.d
cat << EOF >> /etc/dracut.conf.d/10-crypt.conf
install_items+=" /boot/volume.key /etc/crypttab "
EOF

# Regenerate initramfs with keyfile support
xbps-reconfigure -fa
else
echo "System without LUKS: skipping keyfile creation."
be
```

# ▶️ 18. Install GRUB in **BIOS** and **UEFI** (real hybrid)
1. Install GRUB for BIOS (Legacy)
```
grub-install --target=i386-pc ${DEVICE}
```
2. Install GRUB for UEFI
```
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=void
```
3. Create UEFI fallback (universal boot). This file guarantees booting even when NVRAM is erased.
```
mkdir -p /boot/efi/EFI/BOOT
cp -f /boot/efi/EFI/void/grubx64.efi /boot/efi/EFI/BOOT/BOOTX64.EFI
```
4. Generate final GRUB file
```
grub-mkconfig -o /boot/grub/grub.cfg
```

---

# ▶️ 19. Customized user settings:

1. Environment Settings:

```
# Customize /etc/xbps.d/00-repository-main.conf
mkdir -pv /etc/xbps.d
cat << 'EOF' >> /etc/xbps.d/00-repository-main.conf
repository=https://repo-fastly.voidlinux.org/current
#repository=https://repo-fastly.voidlinux.org/current/nonfree
#repository=https://repo-fastly.voidlinux.org/current/multilib
#repository=https://repo-fastly.voidlinux.org/current/multilib/nonfree

repository=https://void.chililinux.com/voidlinux/current
#repository=https://void.chililinux.com/voidlinux/current/extras
#repository=https://void.chililinux.com/voidlinux/current/nonfree
#repository=https://void.chililinux.com/voidlinux/current/multilib
#repository=https://void.chililinux.com/voidlinux/current/multilib/nonfree
EOF

# Customize /etc/rc.conf. Sets the console's default time zone, keyboard layout, and font. Change as needed.
cat << EOF >> /etc/rc.conf
TIMEZONE="${TIMEZONE}"
KEYMAP="${KEYMAP}"
FONT=Lat2-Terminus16
EOF

# Customize root's .bashrc
wget --quiet --no-check-certificate \
-O /etc//skel/.bashrc \
"https://raw.githubusercontent.com/voidlinux-br/void-install/refs/heads/main/.bashrc"
chown root:root /etc/skel/.bashrc
chmod 644 /etc/skel/.bashrc

cat << 'EOF' > /etc/skel/.bash_profile
# ~/.bash_profile — loads .bashrc into Void

# If .bashrc exists, load
if [ -f ~/.bashrc ]; then
source ~/.bashrc
be
EOF

# copy to root and user
for d in /root "/home/${NEWUSER}"; do
cp -f /etc/skel/.bash_profile "$d/"
cp -f /etc/skel/.bashrc "$d/"
done

chown "${NEWUSER}:${NEWUSER}" "/home/${NEWUSER}/.bash_profile" "/home/${NEWUSER}/.bashrc"
chmod 644 "/home/${NEWUSER}/.bash_profile" "/home/${NEWUSER}/.bashrc"

# download custom svlogtail
wget --quiet --no-check-certificate \
-O /usr/bin/svlogtail\
"https://raw.githubusercontent.com/voidlinux-br/void-install/refs/heads/main/svlogtail"
chmod +x /usr/bin/svlogtail
```

2. Configure ssh (optional, but recommended):
```
mkdir -pv /etc/ssh/sshd_config.d/
cat << 'EOF' > /etc/ssh/sshd_config.d/10-custom.conf
PermitTTY yes
PrintMotd yes
PrintLastLog yes
Banner /etc/issue.net

PermitRootLogin yes
KbdInteractiveAuthentication yes
X11Forwarding yes
PubkeyAuthentication yes
PubkeyAcceptedKeyTypes=+ssh-rsa
AuthorizedKeysFile .ssh/authorized_keys
PasswordAuthentication yes
ChallengeResponseAuthentication yes
UsePAM yes

Subsystem sftp internal-sftp
EOF
```
---

# ▶️ 20. Enable ZRAM (optional)
Void Linux uses the zramen service to enable ZRAM, creating a block of compressed memory that reduces SSD swap usage and improves performance under load.
1. Install zramen
```
xbps-install -Sy zramen
```
2. Configure ZRAM (recommended configuration):
```
cat << 'EOF' > /etc/zramen.conf
zram_fraction=0.5
zram_devices=1
zram_algorithm=zstd
EOF
```
3. Activate the service in runit
```
ln -s
```
> ZRAM will be automatically activated on every boot

---

# ▶️ 21. Finish installation
1. Sair do chroot:
```
exit
```
2. Unmount all partitions mounted on /mnt (subvolumes and /boot/efi):
```
umount -R /mnt
```
3. Disable any swapfile or swap partition that has been activated within the chroot:
```
swapoff -a
```
4. Restart the physical machine or VM to test the actual boot:
```
reboot
```
> Don't forget to remove the installation media and boot from the newly installed disc.
Enjoy!

---

# 🎉 COMPLETE, HYBRID, FUTURE-PROOF SYSTEM
- Boot BIOS + UEFI
- Fallback UEFI
- Btrfs with snapshots (Snapper/Timeshift ready)
- Real hibernation with swapfile
- Zram for performance

This SSD boots **any machine on the planet**.

# DISCLAIMER

```
This tutorial is free: you can use, copy, modify and redistribute as you wish.
Content is made available under the **MIT License**, and may include snippets or commands derived from open source software subject to its own licenses.

No guarantees are provided — everything here is delivered “as is”.
Use at your own risk. Neither the author, nor contributors, nor Void Linux are responsible for loss, damage, system failures or any consequences of the use of this material.

If you wish, you can obtain the source code, review, adapt, and generate your own version of this tutorial.
```

