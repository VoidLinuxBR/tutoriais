# VoidBR を制限しない — BIOS + UEFI

BIOS/レガシーおよび UEFI サポートを使用して VoidBR に Limine をインストールし、GRUB もインストールしたままにするチュートリアル。

## 1. レイアウト・ド・ディスコ

使用例:

```text
/dev/vda
├── /dev/vda1   2 MiB     BIOS Boot
├── /dev/vda2   512 MiB   FAT32   /boot
└── /dev/vda3   restante  ext4    /
```

「vda1」は BIOS ブート パーティションです。

`vda2` は FAT32 であり、`/boot` として使用されます。

`vda3` にはルート システムが含まれます。

FAT32 `/boot` には、元の `/boot` に存在する Limine、GRUB、カーネル、initramfs およびその他のファイルが含まれます。

---

## 2. Limineをインストールする

```bash
sudo vinstall -S
sudo vinstall -y limine
```

それをチェックしてください：

```bash
limine --version
```

重要なファイル:

```text
/usr/bin/limine
/usr/share/limine/BOOTX64.EFI
/usr/share/limine/limine-bios.sys
```

---

## 3.efibootmgrをインストールする

```bash
sudo vinstall -y efibootmgr
```

---

## 4. root をマウントして古い /boot にアクセスします。

現在の `/boot` はルート パーティション (`vda3`) 内にあります。

マウント ポイントを作成します。

```bash
sudo mkdir -p /mnt/rootfs
```

ルートをマウントします。

```bash
sudo mount /dev/vda3 /mnt/rootfs
```

それをチェックしてください：

```bash
ls -lh /mnt/rootfs/boot
```

---

## 5. 新しい /boot パーティションを一時的にマウントします。

マウント ポイントを作成します。

```bash
sudo mkdir -p /mnt/newboot
```

FAT32 をモンテ:

```bash
sudo mount /dev/vda2 /mnt/newboot
```

それをチェックしてください：

```bash
findmnt /mnt/newboot
```

次のように表示されるはずです。

```text
/mnt/newboot  /dev/vda2  vfat
```

---

## 6. 古い /boot をすべてコピーします。

GRUB もメンテナンスされるため、古い `/boot` の内容をすべてコピーします。

```bash
sudo cp -a /mnt/rootfs/boot/. /mnt/newboot/
```

これにより、たとえば次のように保存されます。

```text
grub/
memtest86+/
vmlinuz-*
initramfs-*
config-*
memtest.bin
```

古い `/boot` にすでに存在する Limine ファイルもコピーされます。

それをチェックしてください：

```bash
ls -lah /mnt/newboot
```

---

## 7. 新しい /boot パーティションをアンマウントします。

```bash
sudo umount /mnt/newboot
```

---

## 8. FAT32 を /boot として永続的にマウントします。

```bash
sudo mount /dev/vda2 /boot
```

それをチェックしてください：

```bash
findmnt /boot
```

以下が表示されるはずです:

```text
/boot  /dev/vda2  vfat
```

それをチェックしてください：

```bash
ls -lah /boot
```

`/boot` の古い内容 (ディレクトリを含む) が存在する必要があります。

```text
/boot/grub/
```

---

## 9. ルート UUID を取得する

`root=UUID=` で使用される UUID は、ルート パーティション (`vda3`) の UUID である必要があります。

```bash
blkid /dev/vda3
```

例：

```text
/dev/vda3: UUID="a128f5c1-eb0d-4dc3-a42d-131bde041284" TYPE="ext4"
```

---

## 10. limine.confを作成する

新しい `/boot` FAT32 にファイルを直接作成します。

```bash
sudo nano /boot/limine.conf
```

例：

```text
TIMEOUT: 3
VERBOSE: no

/VoidBR (Kernel 6.18.44_1)
    PROTOCOL: linux
    PATH: boot():/vmlinuz-6.18.44_1
    MODULE_PATH: boot():/initramfs-6.18.44_1.img
    CMDLINE: root=UUID=a128f5c1-eb0d-4dc3-a42d-131bde041284 rw loglevel=4
```

交換する：

```text
a128f5c1-eb0d-4dc3-a42d-131bde041284
```

`vda3` の実際の UUID によって決まります。

それをチェックしてください：

```bash
cat /boot/limine.conf
```

---

## 11.UEFIファームウェアをインストールする

ディレクトリを作成します。

```bash
sudo mkdir -p /boot/EFI/limine
```

EFI 実行可能ファイルをコピーします。

```bash
sudo cp /usr/share/limine/BOOTX64.EFI /boot/EFI/limine/
```

それをチェックしてください：

```bash
ls -lh /boot/EFI/limine/BOOTX64.EFI
```

---

## 12. BIOS 用の Limine ファイルをインストールします

コピー：

```bash
sudo cp /usr/share/limine/limine-bios.sys /boot/
```

それをチェックしてください：

```bash
ls -lh /boot/limine-bios.sys
```

---

## 13. BIOS 用に Limine をインストールする

「vda1」は BIOS ブート パーティションです。

インストール：

```bash
sudo limine bios-install /dev/vda 1
```

インストールは次のように終了するはずです。

```text
Limine BIOS stages installed successfully.
```

---

## 14. UEFI エントリの作成

`vda2` は EFI パーティションであり、`/boot` としてマウントされます。

エントリを作成します。

```bash
sudo efibootmgr -c \
    -d /dev/vda \
    -p 2 \
    -L "Limine" \
    -l '\EFI\limine\BOOTX64.EFI'
```

それをチェックしてください：

```bash
sudo efibootmgr -v
```

次のようなもの:

```text
Boot0004* Limine HD(2,GPT,...)/\EFI\limine\BOOTX64.EFI
```

---

## 15. fstabの設定

FAT32 UUID を取得します。

```bash
blkid /dev/vda2
```

例：

```text
/dev/vda2: UUID="1234-ABCD" TYPE="vfat"
```

編集：

```bash
sudo nano /etc/fstab
```

「vda2」エントリを次のように変更します。

```text
UUID=1234-ABCD  /boot  vfat  defaults  0  2
```

`/boot/efi` に `vda2` をマウントした古いエントリを削除または変更する必要があります。

---

## 16. fstabのテスター

分解:

```bash
sudo umount /boot
```

「fstab」を使用してマウントします。

```bash
sudo mount /boot
```

それをチェックしてください：

```bash
findmnt /boot
```

以下が表示されるはずです:

```text
/boot  /dev/vda2  vfat
```

---

## 17. チェックリミネ

```bash
ls -lh /boot/limine.conf
```

```bash
ls -lh /boot/limine-bios.sys
```

```bash
ls -lh /boot/EFI/limine/BOOTX64.EFI
```

カーネルをチェックアウトします。

```bash
ls -lh /boot/vmlinuz-*
```

initramfs を確認します。

```bash
ls -lh /boot/initramfs-*
```

---

## 18. GRUB をチェックする

GRUB ディレクトリがまだ存在している必要があります。

```bash
ls -lah /boot/grub
```

すべてのコンテンツが FAT32 にコピーされたため、元の `/boot` にあった GRUB ファイルは保存されました。

---

## 19. 最終会議

```bash
findmnt /boot
```

以下が表示されるはずです:

```text
/boot  /dev/vda2  vfat
```

Limine UEFI をチェックしてください:

```bash
sudo efibootmgr -v | grep -i limine
```

構成を確認します。

```bash
cat /boot/limine.conf
```

カーネルは次の場所にある必要があります。

```text
PATH: boot():/vmlinuz-6.18.44_1
```

E または initramfs:

```text
MODULE_PATH: boot():/initramfs-6.18.44_1.img
```

---

## 20. 最終構造

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

## 21. UEFIのテスト

Limine エントリー番号を確認します。

```bash
sudo efibootmgr -v
```

それが「0004」であると仮定すると、次のようになります。

```bash
sudo efibootmgr -n 0004
```

それをチェックしてください：

```bash
sudo efibootmgr | head -3
```

次のように表示されるはずです。

```text
BootNext: 0004
```

再起動：

```bash
sudo reboot
```

---

## 22. BIOS のテスト

マシンのファームウェアを入力し、レガシー/CSM/BIOS モードを選択します。

`vda1` にインストールされた Limine は、同じ `/boot` FAT32 をロードする必要があります。

```text
/dev/vda2
└── /boot
    ├── limine.conf
    ├── vmlinuz-6.18.44_1
    ├── initramfs-6.18.44_1.img
    └── grub/
```

ルート システムは引き続き次のようになります。

```text
/dev/vda3
```

を通して：

```text
root=UUID=<UUID-da-vda3>
```

---

## 結果

同じ FAT32 `/boot` が Limine と GRUB で使用されます。

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
