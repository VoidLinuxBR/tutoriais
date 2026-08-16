# 限制 VoidBR — BIOS + UEFI

在具有 BIOS/Legacy 和 UEFI 支持的 VoidBR 上安装 Limine 的教程，同时保持 GRUB 的安装。

## 1. 迪斯科布局

使用的示例：

```text
/dev/vda
├── /dev/vda1   2 MiB     BIOS Boot
├── /dev/vda2   512 MiB   FAT32   /boot
└── /dev/vda3   restante  ext4    /
```

`vda1` 是 BIOS 启动分区。

`vda2` 是 FAT32，将用作 `/boot`。

`vda3` 包含根系统。

FAT32 `/boot` 将包含 Limine、GRUB、kernel、initramfs 以及原始 `/boot` 中存在的其他文件。

---

## 2.安装Limine

```bash
sudo vinstall -S
sudo vinstall -y limine
```

一探究竟：

```bash
limine --version
```

重要文件：

```text
/usr/bin/limine
/usr/share/limine/BOOTX64.EFI
/usr/share/limine/limine-bios.sys
```

---

## 3.安装efibootmgr

```bash
sudo vinstall -y efibootmgr
```

---

## 4.挂载root访问旧的/boot

当前的“/boot”位于根分区（“vda3”）内。

创建挂载点：

```bash
sudo mkdir -p /mnt/rootfs
```

挂载根：

```bash
sudo mount /dev/vda3 /mnt/rootfs
```

一探究竟：

```bash
ls -lh /mnt/rootfs/boot
```

---

## 5.临时挂载新的/boot分区

创建挂载点：

```bash
sudo mkdir -p /mnt/newboot
```

蒙特一个FAT32：

```bash
sudo mount /dev/vda2 /mnt/newboot
```

一探究竟：

```bash
findmnt /mnt/newboot
```

它应该出现：

```text
/mnt/newboot  /dev/vda2  vfat
```

---

## 6.复制所有旧的/boot

由于 GRUB 也将被维护，因此复制旧的 `/boot` 的所有内容：

```bash
sudo cp -a /mnt/rootfs/boot/. /mnt/newboot/
```

例如，这可以保留：

```text
grub/
memtest86+/
vmlinuz-*
initramfs-*
config-*
memtest.bin
```

旧的“/boot”中已存在的限制文件也将被复制。

一探究竟：

```bash
ls -lah /mnt/newboot
```

---

## 7.卸载新的/boot分区

```bash
sudo umount /mnt/newboot
```

---

## 8.将FAT32永久挂载为/boot

```bash
sudo mount /dev/vda2 /boot
```

一探究竟：

```bash
findmnt /boot
```

它应该显示：

```text
/boot  /dev/vda2  vfat
```

一探究竟：

```bash
ls -lah /boot
```

`/boot` 的旧内容必须存在，包括目录：

```text
/boot/grub/
```

---

## 9. 获取根UUID

`root=UUID=` 中使用的 UUID 必须是根分区 (`vda3`) 的 UUID。

```bash
blkid /dev/vda3
```

例子：

```text
/dev/vda3: UUID="a128f5c1-eb0d-4dc3-a42d-131bde041284" TYPE="ext4"
```

---

## 10.创建limine.conf

直接在新的`/boot` FAT32 中创建文件：

```bash
sudo nano /boot/limine.conf
```

例子：

```text
TIMEOUT: 3
VERBOSE: no

/VoidBR (Kernel 6.18.44_1)
    PROTOCOL: linux
    PATH: boot():/vmlinuz-6.18.44_1
    MODULE_PATH: boot():/initramfs-6.18.44_1.img
    CMDLINE: root=UUID=a128f5c1-eb0d-4dc3-a42d-131bde041284 rw loglevel=4
```

代替：

```text
a128f5c1-eb0d-4dc3-a42d-131bde041284
```

通过`vda3`的真实UUID。

一探究竟：

```bash
cat /boot/limine.conf
```

---

## 11.安装UEFI固件

创建目录：

```bash
sudo mkdir -p /boot/EFI/limine
```

复制 EFI 可执行文件：

```bash
sudo cp /usr/share/limine/BOOTX64.EFI /boot/EFI/limine/
```

一探究竟：

```bash
ls -lh /boot/EFI/limine/BOOTX64.EFI
```

---

## 12.安装BIOS的Limine文件

复制：

```bash
sudo cp /usr/share/limine/limine-bios.sys /boot/
```

一探究竟：

```bash
ls -lh /boot/limine-bios.sys
```

---

## 13.为BIOS安装Limine

`vda1` 是 BIOS 引导分区。

安装：

```bash
sudo limine bios-install /dev/vda 1
```

安装应以以下内容结束：

```text
Limine BIOS stages installed successfully.
```

---

## 14.创建UEFI条目

`vda2` 是 EFI 分区，挂载为 `/boot`。

创建条目：

```bash
sudo efibootmgr -c \
    -d /dev/vda \
    -p 2 \
    -L "Limine" \
    -l '\EFI\limine\BOOTX64.EFI'
```

一探究竟：

```bash
sudo efibootmgr -v
```

类似于：

```text
Boot0004* Limine HD(2,GPT,...)/\EFI\limine\BOOTX64.EFI
```

---

## 15.配置fstab

获取 FAT32 UUID：

```bash
blkid /dev/vda2
```

例子：

```text
/dev/vda2: UUID="1234-ABCD" TYPE="vfat"
```

编辑：

```bash
sudo nano /etc/fstab
```

将“vda2”条目更改为：

```text
UUID=1234-ABCD  /boot  vfat  defaults  0  2
```

必须删除或更改在“/boot/efi”中安装“vda2”的旧条目。

---

## 16. fstab测试星

拆卸：

```bash
sudo umount /boot
```

使用“fstab”挂载：

```bash
sudo mount /boot
```

一探究竟：

```bash
findmnt /boot
```

它应该显示：

```text
/boot  /dev/vda2  vfat
```

---

## 17.检查界限

```bash
ls -lh /boot/limine.conf
```

```bash
ls -lh /boot/limine-bios.sys
```

```bash
ls -lh /boot/EFI/limine/BOOTX64.EFI
```

查看内核：

```bash
ls -lh /boot/vmlinuz-*
```

检查 initramfs：

```bash
ls -lh /boot/initramfs-*
```

---

## 18. 查看 GRUB

GRUB 目录必须仍然存在：

```bash
ls -lah /boot/grub
```

原始“/boot”中的 GRUB 文件被保留，因为所有内容都被复制到 FAT32。

---

## 19. 决赛

```bash
findmnt /boot
```

它应该显示：

```text
/boot  /dev/vda2  vfat
```

查看 Limine UEFI：

```bash
sudo efibootmgr -v | grep -i limine
```

检查配置：

```bash
cat /boot/limine.conf
```

必须通过以下方式找到内核：

```text
PATH: boot():/vmlinuz-6.18.44_1
```

E o initramfs：

```text
MODULE_PATH: boot():/initramfs-6.18.44_1.img
```

---

## 20. 最终结构

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

## 21. 测试 UEFI

找出 Limine 条目号：

```bash
sudo efibootmgr -v
```

假设它是“0004”：

```bash
sudo efibootmgr -n 0004
```

一探究竟：

```bash
sudo efibootmgr | head -3
```

它应该出现：

```text
BootNext: 0004
```

重新启动：

```bash
sudo reboot
```

---

## 22.测试BIOS

输入机器的固件并选择 Legacy/CSM/BIOS 模式。

安装在 `vda1` 上的 Limine 必须加载相同的 `/boot` FAT32：

```text
/dev/vda2
└── /boot
    ├── limine.conf
    ├── vmlinuz-6.18.44_1
    ├── initramfs-6.18.44_1.img
    └── grub/
```

根系统将继续是：

```text
/dev/vda3
```

通过：

```text
root=UUID=<UUID-da-vda3>
```

---

## 结果

Limine 和 GRUB 使用相同的 FAT32 `/boot`：

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
