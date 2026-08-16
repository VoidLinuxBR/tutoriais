# Limine no VoidBR — BIOS + UEFI

Tutorial to install Limine on VoidBR with BIOS/Legacy and UEFI support, also keeping GRUB installed.

## 1. Layout do disco

Example used:

```text
/dev/vda
├── /dev/vda1   2 MiB     BIOS Boot
├── /dev/vda2   512 MiB   FAT32   /boot
└── /dev/vda3   restante  ext4    /
```

`vda1` is the BIOS Boot partition.

`vda2` is FAT32 and will be used as `/boot`.

`vda3` contains the root system.

The FAT32 `/boot` will contain the Limine, GRUB, kernels, initramfs and other files existing in the original `/boot`.

---

## 2. Install Limine

```bash
sudo vinstall -S
sudo vinstall -y limine
```

Check it out:

```bash
limine --version
```

Important files:

```text
/usr/bin/limine
/usr/share/limine/BOOTX64.EFI
/usr/share/limine/limine-bios.sys
```

---

## 3. Install efibootmgr

```bash
sudo vinstall -y efibootmgr
```

---

## 4. Mount root to access the old /boot

The current `/boot` is inside the root partition (`vda3`).

Create a mount point:

```bash
sudo mkdir -p /mnt/rootfs
```

Mount the root:

```bash
sudo mount /dev/vda3 /mnt/rootfs
```

Check it out:

```bash
ls -lh /mnt/rootfs/boot
```

---

## 5. Mount the new /boot partition temporarily

Create the mount point:

```bash
sudo mkdir -p /mnt/newboot
```

Monte a FAT32:

```bash
sudo mount /dev/vda2 /mnt/newboot
```

Check it out:

```bash
findmnt /mnt/newboot
```

It should appear:

```text
/mnt/newboot  /dev/vda2  vfat
```

---

## 6. Copy all old /boot

Since GRUB will also be maintained, copy all the contents of the old `/boot`:

```bash
sudo cp -a /mnt/rootfs/boot/. /mnt/newboot/
```

This preserves, for example:

```text
grub/
memtest86+/
vmlinuz-*
initramfs-*
config-*
memtest.bin
```

Limine files that already exist in the old `/boot` will also be copied.

Check it out:

```bash
ls -lah /mnt/newboot
```

---

## 7. Unmount the new /boot partition

```bash
sudo umount /mnt/newboot
```

---

## 8. Mount FAT32 permanently as /boot

```bash
sudo mount /dev/vda2 /boot
```

Check it out:

```bash
findmnt /boot
```

It should show:

```text
/boot  /dev/vda2  vfat
```

Check it out:

```bash
ls -lah /boot
```

The old contents of `/boot` must be present, including the directory:

```text
/boot/grub/
```

---

## 9. Get root UUID

The UUID used in `root=UUID=` must be the UUID of the root partition (`vda3`).

```bash
blkid /dev/vda3
```

Example:

```text
/dev/vda3: UUID="a128f5c1-eb0d-4dc3-a42d-131bde041284" TYPE="ext4"
```

---

## 10. Create limine.conf

Create the file directly in the new `/boot` FAT32:

```bash
sudo nano /boot/limine.conf
```

Example:

```text
TIMEOUT: 3
VERBOSE: no

/VoidBR (Kernel 6.18.44_1)
    PROTOCOL: linux
    PATH: boot():/vmlinuz-6.18.44_1
    MODULE_PATH: boot():/initramfs-6.18.44_1.img
    CMDLINE: root=UUID=a128f5c1-eb0d-4dc3-a42d-131bde041284 rw loglevel=4
```

Replace:

```text
a128f5c1-eb0d-4dc3-a42d-131bde041284
```

by the real UUID of `vda3`.

Check it out:

```bash
cat /boot/limine.conf
```

---

## 11. Install UEFI Firmware

Create the directory:

```bash
sudo mkdir -p /boot/EFI/limine
```

Copy the EFI executable:

```bash
sudo cp /usr/share/limine/BOOTX64.EFI /boot/EFI/limine/
```

Check it out:

```bash
ls -lh /boot/EFI/limine/BOOTX64.EFI
```

---

## 12. Install Limine file for BIOS

Copy:

```bash
sudo cp /usr/share/limine/limine-bios.sys /boot/
```

Check it out:

```bash
ls -lh /boot/limine-bios.sys
```

---

## 13. Install Limine for BIOS

`vda1` is the BIOS Boot Partition.

Install:

```bash
sudo limine bios-install /dev/vda 1
```

The installation should end with:

```text
Limine BIOS stages installed successfully.
```

---

## 14. Create the UEFI entry

`vda2` is the EFI partition and is mounted as `/boot`.

Create the entry:

```bash
sudo efibootmgr -c \
    -d /dev/vda \
    -p 2 \
    -L "Limine" \
    -l '\EFI\limine\BOOTX64.EFI'
```

Check it out:

```bash
sudo efibootmgr -v
```

Something similar to:

```text
Boot0004* Limine HD(2,GPT,...)/\EFI\limine\BOOTX64.EFI
```

---

## 15. Configure fstab

Get the FAT32 UUID:

```bash
blkid /dev/vda2
```

Example:

```text
/dev/vda2: UUID="1234-ABCD" TYPE="vfat"
```

Edit:

```bash
sudo nano /etc/fstab
```

Change the `vda2` entry to:

```text
UUID=1234-ABCD  /boot  vfat  defaults  0  2
```

The old entry that mounted `vda2` in `/boot/efi` must be removed or changed.

---

## 16. Testar o fstab

Disassemble:

```bash
sudo umount /boot
```

Mount using `fstab`:

```bash
sudo mount /boot
```

Check it out:

```bash
findmnt /boot
```

It should show:

```text
/boot  /dev/vda2  vfat
```

---

## 17. Check Limine

```bash
ls -lh /boot/limine.conf
```

```bash
ls -lh /boot/limine-bios.sys
```

```bash
ls -lh /boot/EFI/limine/BOOTX64.EFI
```

Check out the kernels:

```bash
ls -lh /boot/vmlinuz-*
```

Check the initramfs:

```bash
ls -lh /boot/initramfs-*
```

---

## 18. Check out GRUB

The GRUB directory must still be present:

```bash
ls -lah /boot/grub
```

The GRUB files that were in the original `/boot` were preserved because all content was copied to FAT32.

---

## 19. Final conference

```bash
findmnt /boot
```

It should show:

```text
/boot  /dev/vda2  vfat
```

Check out Limine UEFI:

```bash
sudo efibootmgr -v | grep -i limine
```

Check the configuration:

```bash
cat /boot/limine.conf
```

The kernel must be located through:

```text
PATH: boot():/vmlinuz-6.18.44_1
```

E o initramfs:

```text
MODULE_PATH: boot():/initramfs-6.18.44_1.img
```

---

## 20. Final structure

```text
/dev/vda
├── vda1
│   └── BIOS Boot
│
├── vda2
│   └── FAT32
│       └── /boot
│           ├── limine.conf
│           ├── limine-bios.sys
│           ├── vmlinuz-6.18.44_1
│           ├── initramfs-6.18.44_1.img
│           ├── config-6.18.44_1
│           ├── memtest.bin
│           ├── memtest86+/
│           ├── grub/
│           └── EFI/
│               └── limine/
│                   └── BOOTX64.EFI
│
└── vda3
    └── ext4
        └── /
```

---

## 21. Testing UEFI

Find out the Limine entry number:

```bash
sudo efibootmgr -v
```

Assuming it is `0004`:

```bash
sudo efibootmgr -n 0004
```

Check it out:

```bash
sudo efibootmgr | head -3
```

It should appear:

```text
BootNext: 0004
```

Restart:

```bash
sudo reboot
```

---

## 22. Test BIOS

Enter the machine's firmware and select Legacy/CSM/BIOS mode.

Limine installed on `vda1` must load the same `/boot` FAT32:

```text
/dev/vda2
└── /boot
    ├── limine.conf
    ├── vmlinuz-6.18.44_1
    ├── initramfs-6.18.44_1.img
    └── grub/
```

The root system will continue to be:

```text
/dev/vda3
```

through:

```text
root=UUID=<UUID-da-vda3>
```

---

## Result

The same FAT32 `/boot` is used by Limine and GRUB:

```text
                    /dev/vda2
                     FAT32
                      /boot
                        │
              ┌─────────┴─────────┐
              │                   │
            Limine               GRUB
              │                   │
        ┌─────┴─────┐             │
        │           │             │
       UEFI        BIOS           │
        │           │             │
  BOOTX64.EFI   BIOS stages       │
        │           │             │
        └─────┬─────┘             │
              │                   │
              └─────────┬─────────┘
                        │
                     Kernel
                        │
                    initramfs
                        │
                    /dev/vda3
                       ext4
                        /
```
