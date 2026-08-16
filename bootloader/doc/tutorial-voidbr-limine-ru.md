# Limine no VoidBR — BIOS + UEFI

Руководство по установке Limine на VoidBR с поддержкой BIOS/Legacy и UEFI, а также с сохранением установленного GRUB.

## 1. Расстановка дискотек

Используемый пример:

```text
/dev/vda
├── /dev/vda1   2 MiB     BIOS Boot
├── /dev/vda2   512 MiB   FAT32   /boot
└── /dev/vda3   restante  ext4    /
```

`vda1` — это загрузочный раздел BIOS.

`vda2` имеет файловую систему FAT32 и будет использоваться как `/boot`.

`vda3` содержит корневую систему.

`/boot` FAT32 будет содержать Limine, GRUB, ядра, initramfs и другие файлы, существующие в исходном `/boot`.

---

## 2. Установите Лимин

```bash
sudo vinstall -S
sudo vinstall -y limine
```

Проверьте это:

```bash
limine --version
```

Важные файлы:

```text
/usr/bin/limine
/usr/share/limine/BOOTX64.EFI
/usr/share/limine/limine-bios.sys
```

---

## 3. Установите efibootmgr.

```bash
sudo vinstall -y efibootmgr
```

---

## 4. Подключите root для доступа к старому каталогу /boot.

Текущий `/boot` находится внутри корневого раздела (`vda3`).

Создайте точку монтирования:

```bash
sudo mkdir -p /mnt/rootfs
```

Монтируем корень:

```bash
sudo mount /dev/vda3 /mnt/rootfs
```

Проверьте это:

```bash
ls -lh /mnt/rootfs/boot
```

---

## 5. Временно смонтируйте новый раздел /boot.

Создайте точку монтирования:

```bash
sudo mkdir -p /mnt/newboot
```

Монтируем FAT32:

```bash
sudo mount /dev/vda2 /mnt/newboot
```

Проверьте это:

```bash
findmnt /mnt/newboot
```

Должно появиться:

```text
/mnt/newboot  /dev/vda2  vfat
```

---

## 6. Копируем весь старый /boot

Поскольку GRUB также будет поддерживаться, скопируйте все содержимое старого `/boot`:

```bash
sudo cp -a /mnt/rootfs/boot/. /mnt/newboot/
```

Это сохраняет, например:

```text
grub/
memtest86+/
vmlinuz-*
initramfs-*
config-*
memtest.bin
```

Файлы Limine, которые уже существуют в старом каталоге `/boot`, также будут скопированы.

Проверьте это:

```bash
ls -lah /mnt/newboot
```

---

## 7. Отключите новый раздел /boot.

```bash
sudo umount /mnt/newboot
```

---

## 8. Установите FAT32 навсегда в /boot.

```bash
sudo mount /dev/vda2 /boot
```

Проверьте это:

```bash
findmnt /boot
```

Он должен показать:

```text
/boot  /dev/vda2  vfat
```

Проверьте это:

```bash
ls -lah /boot
```

Должно присутствовать старое содержимое `/boot`, включая каталог:

```text
/boot/grub/
```

---

## 9. Получите root UUID

UUID, используемый в `root=UUID=`, должен быть UUID корневого раздела (`vda3`).

```bash
blkid /dev/vda3
```

Пример:

```text
/dev/vda3: UUID="a128f5c1-eb0d-4dc3-a42d-131bde041284" TYPE="ext4"
```

---

## 10. Создайте файл libine.conf.

Создайте файл непосредственно в новом каталоге `/boot` FAT32:

```bash
sudo nano /boot/limine.conf
```

Пример:

```text
TIMEOUT: 3
VERBOSE: no

/VoidBR (Kernel 6.18.44_1)
    PROTOCOL: linux
    PATH: boot():/vmlinuz-6.18.44_1
    MODULE_PATH: boot():/initramfs-6.18.44_1.img
    CMDLINE: root=UUID=a128f5c1-eb0d-4dc3-a42d-131bde041284 rw loglevel=4
```

Заменять:

```text
a128f5c1-eb0d-4dc3-a42d-131bde041284
```

по реальному UUID `vda3`.

Проверьте это:

```bash
cat /boot/limine.conf
```

---

## 11. Установите прошивку UEFI.

Создайте каталог:

```bash
sudo mkdir -p /boot/EFI/limine
```

Скопируйте исполняемый файл EFI:

```bash
sudo cp /usr/share/limine/BOOTX64.EFI /boot/EFI/limine/
```

Проверьте это:

```bash
ls -lh /boot/EFI/limine/BOOTX64.EFI
```

---

## 12. Установите файл Limine для BIOS.

Копировать:

```bash
sudo cp /usr/share/limine/limine-bios.sys /boot/
```

Проверьте это:

```bash
ls -lh /boot/limine-bios.sys
```

---

## 13. Установите Limine для BIOS.

`vda1` — это загрузочный раздел BIOS.

Установить:

```bash
sudo limine bios-install /dev/vda 1
```

Установка должна завершиться следующим:

```text
Limine BIOS stages installed successfully.
```

---

## 14. Создайте запись UEFI.

`vda2` — это раздел EFI, который монтируется как `/boot`.

Создайте запись:

```bash
sudo efibootmgr -c \
    -d /dev/vda \
    -p 2 \
    -L "Limine" \
    -l '\EFI\limine\BOOTX64.EFI'
```

Проверьте это:

```bash
sudo efibootmgr -v
```

Что-то похожее на:

```text
Boot0004* Limine HD(2,GPT,...)/\EFI\limine\BOOTX64.EFI
```

---

## 15. Настройте fstab

Получите UUID FAT32:

```bash
blkid /dev/vda2
```

Пример:

```text
/dev/vda2: UUID="1234-ABCD" TYPE="vfat"
```

Редактировать:

```bash
sudo nano /etc/fstab
```

Измените запись `vda2` на:

```text
UUID=1234-ABCD  /boot  vfat  defaults  0  2
```

Старую запись, которая монтировала `vda2` в `/boot/efi`, необходимо удалить или изменить.

---

## 16. Тест о фстаб

Разобрать:

```bash
sudo umount /boot
```

Монтируем с помощью fstab:

```bash
sudo mount /boot
```

Проверьте это:

```bash
findmnt /boot
```

Он должен показать:

```text
/boot  /dev/vda2  vfat
```

---

## 17. Проверьте известняк

```bash
ls -lh /boot/limine.conf
```

```bash
ls -lh /boot/limine-bios.sys
```

```bash
ls -lh /boot/EFI/limine/BOOTX64.EFI
```

Проверьте ядра:

```bash
ls -lh /boot/vmlinuz-*
```

Проверьте initramfs:

```bash
ls -lh /boot/initramfs-*
```

---

## 18. Проверьте GRUB

Каталог GRUB все еще должен присутствовать:

```bash
ls -lah /boot/grub
```

Файлы GRUB, которые находились в исходном каталоге «/boot», были сохранены, поскольку все содержимое было скопировано в FAT32.

---

## 19. Итоговая конференция

```bash
findmnt /boot
```

Он должен показать:

```text
/boot  /dev/vda2  vfat
```

Проверьте Limine UEFI:

```bash
sudo efibootmgr -v | grep -i limine
```

Проверьте конфигурацию:

```bash
cat /boot/limine.conf
```

Ядро должно находиться через:

```text
PATH: boot():/vmlinuz-6.18.44_1
```

Э или initramfs:

```text
MODULE_PATH: boot():/initramfs-6.18.44_1.img
```

---

## 20. Окончательная структура

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

## 21. Тестирование UEFI

Узнайте номер входа Limine:

```bash
sudo efibootmgr -v
```

Предполагая, что это `0004`:

```bash
sudo efibootmgr -n 0004
```

Проверьте это:

```bash
sudo efibootmgr | head -3
```

Должно появиться:

```text
BootNext: 0004
```

Перезапуск:

```bash
sudo reboot
```

---

## 22. Тестирование BIOS

Введите прошивку устройства и выберите режим Legacy/CSM/BIOS.

Limine, установленный на `vda1`, должен загружать ту же `/boot` FAT32:

```text
/dev/vda2
└── /boot
    ├── limine.conf
    ├── vmlinuz-6.18.44_1
    ├── initramfs-6.18.44_1.img
    └── grub/
```

Корневая система по-прежнему будет:

```text
/dev/vda3
```

через:

```text
root=UUID=<UUID-da-vda3>
```

---

## Результат

Один и тот же `/boot` FAT32 используется Limine и GRUB:

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
