# 限制 VoidBR — BIOS + UEFI

在具有 BIOS/Legacy 和 UEFI 支援的 VoidBR 上安裝 Limine 的教學課程，同時保持 GRUB 的安裝。

## 1. 迪斯科佈局

使用的範例：

```text
/dev/vda
├── /dev/vda1   2 MiB     BIOS Boot
├── /dev/vda2   512 MiB   FAT32   /boot
└── /dev/vda3   restante  ext4    /
```

`vda1` 是 BIOS 啟動分區。

`vda2` 是 FAT32，將用作 `/boot`。

`vda3` 包含根系統。

FAT32 `/boot` 將包含 Limine、GRUB、kernel、initramfs 以及原始 `/boot` 中存在的其他檔案。

---

## 2.安裝Limine

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

## 

```bash
sudo vinstall -y efibootmgr
```

---

## 

目前的“/boot”位於根分區（“vda3”）內。

建立掛載點：

```bash
sudo mkdir -p /mnt/rootfs
```

掛載根：

```bash
sudo mount /dev/vda3 /mnt/rootfs
```

一探究竟：

```bash
ls -lh /mnt/rootfs/boot
```

---

## 5.臨時掛載新的/boot分割區

建立掛載點：

```bash
sudo mkdir -p /mnt/newboot
```

蒙特一個FAT32：

```bash
sudo mount /dev/vda2 /mnt/newboot
```

一探究竟：

```bash
findmnt /mnt/newboot
```

它應該會出現：

```text
/mnt/newboot  /dev/vda2  vfat
```

---

## 6.複製所有舊的/boot

由於 GRUB 也將被維護，因此複製舊的 `/boot` 的所有內容：

```bash
sudo cp -a /mnt/rootfs/boot/. /mnt/newboot/
```

例如，這可以保留：

```text
grub/
memtest86+/
vmlinuz-*
initramfs-*
config-*
memtest.bin
```

舊的“/boot”中已存在的限製文件也將被複製。

一探究竟：

```bash
ls -lah /mnt/newboot
```

---

## 7.卸載新的/boot分割區

```bash
sudo umount /mnt/newboot
```

---

## 8.將FAT32永久掛載為/boot

```bash
sudo mount /dev/vda2 /boot
```

一探究竟：

```bash
findmnt /boot
```

它應該顯示：

```text
/boot  /dev/vda2  vfat
```

一探究竟：

```bash
ls -lah /boot
```

`/boot` 的舊內容必須存在，包括目錄：

```text
/boot/grub/
```

---

## 9. 取得根UUID

`root=UUID=` 中使用的 UUID 必須是根分割區 (`vda3`) 的 UUID。

```bash
blkid /dev/vda3
```

例子：

```text
/dev/vda3: UUID="a128f5c1-eb0d-4dc3-a42d-131bde041284" TYPE="ext4"
```

---

## 10.創建limine.conf

直接在新的`/boot` FAT32 中建立檔案：

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

透過`vda3`的真實UUID。

一探究竟：

```bash
cat /boot/limine.conf
```

---

## 11.安裝UEFI韌體



```bash
sudo mkdir -p /boot/EFI/limine
```

複製 EFI 執行檔：

```bash
sudo cp /usr/share/limine/BOOTX64.EFI /boot/EFI/limine/
```

一探究竟：

```bash
ls -lh /boot/EFI/limine/BOOTX64.EFI
```

---

## 12.安裝BIOS的Limine文件



```bash
sudo cp /usr/share/limine/limine-bios.sys /boot/
```

一探究竟：

```bash
ls -lh /boot/limine-bios.sys
```

---

## 13.為BIOS安裝Limine

`vda1` 是 BIOS 開機分割區。

安裝：

```bash
sudo limine bios-install /dev/vda 1
```

安裝應以以下內容結束：

```text
Limine BIOS stages installed successfully.
```

---

## 14.建立UEFI條目

`vda2` 是 EFI 分割區，掛載為 `/boot`。

建立條目：

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

類似：

```text
Boot0004* Limine HD(2,GPT,...)/\EFI\limine\BOOTX64.EFI
```

---

## 15.配置fstab

取得 FAT32 UUID：

```bash
blkid /dev/vda2
```

例子：

```text
/dev/vda2: UUID="1234-ABCD" TYPE="vfat"
```

編輯：

```bash
sudo nano /etc/fstab
```



```text
UUID=1234-ABCD  /boot  vfat  defaults  0  2
```

必須刪除或變更在“/boot/efi”中安裝“vda2”的舊條目。

---

## 16. fstab測試星

拆卸：

```bash
sudo umount /boot
```

使用“fstab”掛載：

```bash
sudo mount /boot
```

一探究竟：

```bash
findmnt /boot
```

它應該顯示：

```text
/boot  /dev/vda2  vfat
```

---

## 17.檢查界限

```bash
ls -lh /boot/limine.conf
```

```bash
ls -lh /boot/limine-bios.sys
```

```bash
ls -lh /boot/EFI/limine/BOOTX64.EFI
```

查看核心：

```bash
ls -lh /boot/vmlinuz-*
```

檢查 initramfs：

```bash
ls -lh /boot/initramfs-*
```

---

## 18. 查看 GRUB

GRUB 目錄必須仍然存在：

```bash
ls -lah /boot/grub
```

原始“/boot”中的 GRUB 檔案被保留，因為所有內容都被複製到 FAT32。

---

## 19. 決賽

```bash
findmnt /boot
```

它應該顯示：

```text
/boot  /dev/vda2  vfat
```

查看 Limine UEFI：

```bash
sudo efibootmgr -v | grep -i limine
```

檢查配置：

```bash
cat /boot/limine.conf
```

必須透過以下方式找到內核：

```text
PATH: boot():/vmlinuz-6.18.44_1
```

E o initramfs：

```text
MODULE_PATH: boot():/initramfs-6.18.44_1.img
```

---

## 20. 最終結構

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

## 21. 測試 UEFI

找出 Limine 條目編號：

```bash
sudo efibootmgr -v
```



```bash
sudo efibootmgr -n 0004
```

一探究竟：

```bash
sudo efibootmgr | head -3
```

它應該會出現：

```text
BootNext: 0004
```

重新啟動：

```bash
sudo reboot
```

---

## 22.測試BIOS



安裝在 `vda1` 上的 Limine 必須載入相同的 `/boot` FAT32：

```text
/dev/vda2
└── /boot
    ├── limine.conf
    ├── vmlinuz-6.18.44_1
    ├── initramfs-6.18.44_1.img
    └── grub/
```

根系統將繼續是：

```text
/dev/vda3
```

通過：

```text
root=UUID=<UUID-da-vda3>
```

---

## 結果

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
