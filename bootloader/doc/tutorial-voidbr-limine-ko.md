# Void 없음BR — BIOS + UEFI

BIOS/Legacy 및 UEFI 지원을 통해 VoidBR에 Limine을 설치하고 GRUB 설치를 유지하는 방법에 대한 튜토리얼입니다.

## 1. 디스코 레이아웃

사용된 예:

```text
/dev/vda
├── /dev/vda1   2 MiB     BIOS Boot
├── /dev/vda2   512 MiB   FAT32   /boot
└── /dev/vda3   restante  ext4    /
```

`vda1`은 BIOS 부팅 파티션입니다.

`vda2`는 FAT32이며 `/boot`로 사용됩니다.

`vda3`에는 루트 시스템이 포함되어 있습니다.

FAT32 `/boot`에는 원래 `/boot`에 존재하는 Limine, GRUB, 커널, initramfs 및 기타 파일이 포함됩니다.

---

## 2. 리민 설치

```bash
sudo vinstall -S
sudo vinstall -y limine
```

확인해 보세요:

```bash
limine --version
```

중요한 파일:

```text
/usr/bin/limine
/usr/share/limine/BOOTX64.EFI
/usr/share/limine/limine-bios.sys
```

---

## 3. efibootmgr 설치

```bash
sudo vinstall -y efibootmgr
```

---

## 4. 이전 /boot에 액세스하기 위해 루트를 마운트합니다.

현재 `/boot`는 루트 파티션(`vda3`) 내에 있습니다.

마운트 지점을 생성합니다:

```bash
sudo mkdir -p /mnt/rootfs
```

루트를 마운트합니다:

```bash
sudo mount /dev/vda3 /mnt/rootfs
```

확인해 보세요:

```bash
ls -lh /mnt/rootfs/boot
```

---

## 5. 새 /boot 파티션을 임시로 마운트합니다.

마운트 지점을 생성합니다:

```bash
sudo mkdir -p /mnt/newboot
```

FAT32 몬테:

```bash
sudo mount /dev/vda2 /mnt/newboot
```

확인해 보세요:

```bash
findmnt /mnt/newboot
```

다음과 같이 나타나야 합니다.

```text
/mnt/newboot  /dev/vda2  vfat
```

---

## 6. 이전 /boot를 모두 복사합니다.

GRUB도 유지 관리되므로 이전 `/boot`의 모든 내용을 복사하세요.

```bash
sudo cp -a /mnt/rootfs/boot/. /mnt/newboot/
```

예를 들어 이는 다음을 유지합니다.

```text
grub/
memtest86+/
vmlinuz-*
initramfs-*
config-*
memtest.bin
```

이전 `/boot`에 이미 존재하는 Limine 파일도 복사됩니다.

확인해 보세요:

```bash
ls -lah /mnt/newboot
```

---

## 7. 새 /boot 파티션을 마운트 해제합니다.

```bash
sudo umount /mnt/newboot
```

---

## 8. FAT32를 /boot로 영구적으로 마운트합니다.

```bash
sudo mount /dev/vda2 /boot
```

확인해 보세요:

```bash
findmnt /boot
```

다음이 표시되어야 합니다.

```text
/boot  /dev/vda2  vfat
```

확인해 보세요:

```bash
ls -lah /boot
```

다음 디렉터리를 포함하여 `/boot`의 이전 내용이 있어야 합니다.

```text
/boot/grub/
```

---

## 9. 루트 UUID 얻기

`root=UUID=`에 사용된 UUID는 루트 파티션(`vda3`)의 UUID여야 합니다.

```bash
blkid /dev/vda3
```

예:

```text
/dev/vda3: UUID="a128f5c1-eb0d-4dc3-a42d-131bde041284" TYPE="ext4"
```

---

## 10. limine.conf 생성

새로운 `/boot` FAT32에서 직접 파일을 생성합니다:

```bash
sudo nano /boot/limine.conf
```

예:

```text
TIMEOUT: 3
VERBOSE: no

/VoidBR (Kernel 6.18.44_1)
    PROTOCOL: linux
    PATH: boot():/vmlinuz-6.18.44_1
    MODULE_PATH: boot():/initramfs-6.18.44_1.img
    CMDLINE: root=UUID=a128f5c1-eb0d-4dc3-a42d-131bde041284 rw loglevel=4
```

바꾸다:

```text
a128f5c1-eb0d-4dc3-a42d-131bde041284
```

`vda3`의 실제 UUID로.

확인해 보세요:

```bash
cat /boot/limine.conf
```

---

## 11. UEFI 펌웨어 설치

디렉터리를 만듭니다.

```bash
sudo mkdir -p /boot/EFI/limine
```

EFI 실행 파일을 복사합니다.

```bash
sudo cp /usr/share/limine/BOOTX64.EFI /boot/EFI/limine/
```

확인해 보세요:

```bash
ls -lh /boot/EFI/limine/BOOTX64.EFI
```

---

## 12. BIOS용 Limine 파일 설치

복사:

```bash
sudo cp /usr/share/limine/limine-bios.sys /boot/
```

확인해 보세요:

```bash
ls -lh /boot/limine-bios.sys
```

---

## 13. BIOS용 Limine 설치

`vda1`은 BIOS 부팅 파티션입니다.

설치하다:

```bash
sudo limine bios-install /dev/vda 1
```

설치는 다음으로 끝나야 합니다.

```text
Limine BIOS stages installed successfully.
```

---

## 14. UEFI 항목 생성

`vda2`는 EFI 파티션이며 `/boot`로 마운트됩니다.

항목을 만듭니다.

```bash
sudo efibootmgr -c \
    -d /dev/vda \
    -p 2 \
    -L "Limine" \
    -l '\EFI\limine\BOOTX64.EFI'
```

확인해 보세요:

```bash
sudo efibootmgr -v
```

다음과 비슷한 것:

```text
Boot0004* Limine HD(2,GPT,...)/\EFI\limine\BOOTX64.EFI
```

---

## 15. fstab 구성

FAT32 UUID를 얻으십시오:

```bash
blkid /dev/vda2
```

예:

```text
/dev/vda2: UUID="1234-ABCD" TYPE="vfat"
```

편집하다:

```bash
sudo nano /etc/fstab
```

`vda2` 항목을 다음으로 변경합니다.

```text
UUID=1234-ABCD  /boot  vfat  defaults  0  2
```

`/boot/efi`에 `vda2`를 마운트한 이전 항목을 제거하거나 변경해야 합니다.

---

## 16. fstab의 테스트

분해:

```bash
sudo umount /boot
```

`fstab`을 사용하여 마운트:

```bash
sudo mount /boot
```

확인해 보세요:

```bash
findmnt /boot
```

다음이 표시되어야 합니다.

```text
/boot  /dev/vda2  vfat
```

---

## 17. 리미네를 확인하세요

```bash
ls -lh /boot/limine.conf
```

```bash
ls -lh /boot/limine-bios.sys
```

```bash
ls -lh /boot/EFI/limine/BOOTX64.EFI
```

커널을 확인하세요:

```bash
ls -lh /boot/vmlinuz-*
```

initramfs를 확인하세요.

```bash
ls -lh /boot/initramfs-*
```

---

## 18. GRUB를 확인해보세요

GRUB 디렉터리는 계속 존재해야 합니다.

```bash
ls -lah /boot/grub
```

모든 내용이 FAT32로 복사되었기 때문에 원본 `/boot`에 있던 GRUB 파일은 보존되었습니다.

---

## 19. 최종 컨퍼런스

```bash
findmnt /boot
```

다음이 표시되어야 합니다.

```text
/boot  /dev/vda2  vfat
```

Limine UEFI를 확인하세요:

```bash
sudo efibootmgr -v | grep -i limine
```

구성을 확인하세요.

```bash
cat /boot/limine.conf
```

커널은 다음을 통해 찾아야 합니다.

```text
PATH: boot():/vmlinuz-6.18.44_1
```

E 또는 initramfs:

```text
MODULE_PATH: boot():/initramfs-6.18.44_1.img
```

---

## 20. 최종 구조

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

## 21. UEFI 테스트

Limine 항목 번호를 확인하세요.

```bash
sudo efibootmgr -v
```

'0004'라고 가정하면 다음과 같습니다.

```bash
sudo efibootmgr -n 0004
```

확인해 보세요:

```bash
sudo efibootmgr | head -3
```

다음과 같이 나타나야 합니다.

```text
BootNext: 0004
```

다시 시작:

```bash
sudo reboot
```

---

## 22. 테스트 BIOS

기기의 펌웨어로 들어가서 Legacy/CSM/BIOS 모드를 선택하십시오.

`vda1`에 설치된 Limine은 동일한 `/boot` FAT32를 로드해야 합니다:

```text
/dev/vda2
└── /boot
    ├── limine.conf
    ├── vmlinuz-6.18.44_1
    ├── initramfs-6.18.44_1.img
    └── grub/
```

루트 시스템은 계속해서 다음과 같습니다.

```text
/dev/vda3
```

을 통해:

```text
root=UUID=<UUID-da-vda3>
```

---

## 결과

Limine과 GRUB에서는 동일한 FAT32 `/boot`를 사용합니다.

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
