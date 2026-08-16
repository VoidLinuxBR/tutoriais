# Limine no VoidBR – BIOS + UEFI

Tutorial zur Installation von Limine auf VoidBR mit BIOS/Legacy- und UEFI-Unterstützung, wobei auch GRUB installiert bleibt.

## 1. Disco-Layout

Verwendetes Beispiel:

```text
/dev/vda
├── /dev/vda1   2 MiB     BIOS Boot
├── /dev/vda2   512 MiB   FAT32   /boot
└── /dev/vda3   restante  ext4    /
```

„vda1“ ist die BIOS-Boot-Partition.

„vda2“ ist FAT32 und wird als „/boot“ verwendet.

„vda3“ enthält das Root-System.

Das FAT32 „/boot“ enthält die Limine-, GRUB-, Kernel-, Initramfs- und andere Dateien, die im ursprünglichen „/boot“ vorhanden sind.

---

## 2. Limine installieren

```bash
sudo vinstall -S
sudo vinstall -y limine
```

Hör zu:

```bash
limine --version
```

Wichtige Dateien:

```text
/usr/bin/limine
/usr/share/limine/BOOTX64.EFI
/usr/share/limine/limine-bios.sys
```

---

## 3. Installieren Sie efibootmgr

```bash
sudo vinstall -y efibootmgr
```

---

## 4. Mounten Sie root, um auf das alte /boot zuzugreifen

Das aktuelle „/boot“ befindet sich in der Root-Partition („vda3“).

Erstellen Sie einen Mountpunkt:

```bash
sudo mkdir -p /mnt/rootfs
```

Mounten Sie die Wurzel:

```bash
sudo mount /dev/vda3 /mnt/rootfs
```

Hör zu:

```bash
ls -lh /mnt/rootfs/boot
```

---

## 5. Mounten Sie die neue /boot-Partition vorübergehend

Erstellen Sie den Mount-Punkt:

```bash
sudo mkdir -p /mnt/newboot
```

FAT32-Montage:

```bash
sudo mount /dev/vda2 /mnt/newboot
```

Hör zu:

```bash
findmnt /mnt/newboot
```

Es sollte erscheinen:

```text
/mnt/newboot  /dev/vda2  vfat
```

---

## 6. Kopieren Sie alle alten /boot

Da auch GRUB beibehalten wird, kopieren Sie den gesamten Inhalt des alten „/boot“:

```bash
sudo cp -a /mnt/rootfs/boot/. /mnt/newboot/
```

Dadurch bleiben beispielsweise erhalten:

```text
grub/
memtest86+/
vmlinuz-*
initramfs-*
config-*
memtest.bin
```

Limine-Dateien, die bereits im alten „/boot“ vorhanden sind, werden ebenfalls kopiert.

Hör zu:

```bash
ls -lah /mnt/newboot
```

---

## 7. Hängen Sie die neue /boot-Partition aus

```bash
sudo umount /mnt/newboot
```

---

## 8. Mounten Sie FAT32 dauerhaft als /boot

```bash
sudo mount /dev/vda2 /boot
```

Hör zu:

```bash
findmnt /boot
```

Es sollte Folgendes zeigen:

```text
/boot  /dev/vda2  vfat
```

Hör zu:

```bash
ls -lah /boot
```

Der alte Inhalt von „/boot“ muss vorhanden sein, einschließlich des Verzeichnisses:

```text
/boot/grub/
```

---

## 9. Holen Sie sich die Root-UUID

Die in „root=UUID=“ verwendete UUID muss die UUID der Root-Partition („vda3“) sein.

```bash
blkid /dev/vda3
```

Beispiel:

```text
/dev/vda3: UUID="a128f5c1-eb0d-4dc3-a42d-131bde041284" TYPE="ext4"
```

---

## 10. Erstellen Sie limine.conf

Erstellen Sie die Datei direkt im neuen „/boot“ FAT32:

```bash
sudo nano /boot/limine.conf
```

Beispiel:

```text
TIMEOUT: 3
VERBOSE: no

/VoidBR (Kernel 6.18.44_1)
    PROTOCOL: linux
    PATH: boot():/vmlinuz-6.18.44_1
    MODULE_PATH: boot():/initramfs-6.18.44_1.img
    CMDLINE: root=UUID=a128f5c1-eb0d-4dc3-a42d-131bde041284 rw loglevel=4
```

Ersetzen:

```text
a128f5c1-eb0d-4dc3-a42d-131bde041284
```

durch die echte UUID von „vda3“.

Hör zu:

```bash
cat /boot/limine.conf
```

---

## 11. Installieren Sie die UEFI-Firmware

Erstellen Sie das Verzeichnis:

```bash
sudo mkdir -p /boot/EFI/limine
```

Kopieren Sie die ausführbare EFI-Datei:

```bash
sudo cp /usr/share/limine/BOOTX64.EFI /boot/EFI/limine/
```

Hör zu:

```bash
ls -lh /boot/EFI/limine/BOOTX64.EFI
```

---

## 12. Installieren Sie die Limine-Datei für das BIOS

Kopie:

```bash
sudo cp /usr/share/limine/limine-bios.sys /boot/
```

Hör zu:

```bash
ls -lh /boot/limine-bios.sys
```

---

## 13. Installieren Sie Limine für das BIOS

„vda1“ ist die BIOS-Boot-Partition.

Installieren:

```bash
sudo limine bios-install /dev/vda 1
```

Die Installation sollte mit Folgendem enden:

```text
Limine BIOS stages installed successfully.
```

---

## 14. Erstellen Sie den UEFI-Eintrag

„vda2“ ist die EFI-Partition und wird als „/boot“ gemountet.

Erstellen Sie den Eintrag:

```bash
sudo efibootmgr -c \
    -d /dev/vda \
    -p 2 \
    -L "Limine" \
    -l '\EFI\limine\BOOTX64.EFI'
```

Hör zu:

```bash
sudo efibootmgr -v
```

Etwas Ähnliches wie:

```text
Boot0004* Limine HD(2,GPT,...)/\EFI\limine\BOOTX64.EFI
```

---

## 15. Konfigurieren Sie fstab

Holen Sie sich die FAT32-UUID:

```bash
blkid /dev/vda2
```

Beispiel:

```text
/dev/vda2: UUID="1234-ABCD" TYPE="vfat"
```

Bearbeiten:

```bash
sudo nano /etc/fstab
```

Ändern Sie den Eintrag „vda2“ in:

```text
UUID=1234-ABCD  /boot  vfat  defaults  0  2
```

Der alte Eintrag, der „vda2“ in „/boot/efi“ gemountet hat, muss entfernt oder geändert werden.

---

## 16. Testar of fstab

Zerlegen:

```bash
sudo umount /boot
```

Mounten mit „fstab“:

```bash
sudo mount /boot
```

Hör zu:

```bash
findmnt /boot
```

Es sollte Folgendes zeigen:

```text
/boot  /dev/vda2  vfat
```

---

## 17. Limin prüfen

```bash
ls -lh /boot/limine.conf
```

```bash
ls -lh /boot/limine-bios.sys
```

```bash
ls -lh /boot/EFI/limine/BOOTX64.EFI
```

Schauen Sie sich die Kernel an:

```bash
ls -lh /boot/vmlinuz-*
```

Überprüfen Sie die Initramfs:

```bash
ls -lh /boot/initramfs-*
```

---

## 18. Schauen Sie sich GRUB an

Das GRUB-Verzeichnis muss noch vorhanden sein:

```bash
ls -lah /boot/grub
```

Die GRUB-Dateien, die sich im ursprünglichen „/boot“ befanden, blieben erhalten, da der gesamte Inhalt nach FAT32 kopiert wurde.

---

## 19. Abschlusskonferenz

```bash
findmnt /boot
```

Es sollte Folgendes zeigen:

```text
/boot  /dev/vda2  vfat
```

Schauen Sie sich Limine UEFI an:

```bash
sudo efibootmgr -v | grep -i limine
```

Überprüfen Sie die Konfiguration:

```bash
cat /boot/limine.conf
```

Der Kernel muss gefunden werden über:

```text
PATH: boot():/vmlinuz-6.18.44_1
```

E o initramfs:

```text
MODULE_PATH: boot():/initramfs-6.18.44_1.img
```

---

## 20. Endgültige Struktur

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

## 21. UEFI testen

Finden Sie die Limine-Eintragsnummer heraus:

```bash
sudo efibootmgr -v
```

Angenommen, es ist „0004“:

```bash
sudo efibootmgr -n 0004
```

Hör zu:

```bash
sudo efibootmgr | head -3
```

Es sollte erscheinen:

```text
BootNext: 0004
```

Neustart:

```bash
sudo reboot
```

---

## 22. Testen Sie das BIOS

Geben Sie die Firmware des Geräts ein und wählen Sie den Legacy/CSM/BIOS-Modus.

Limine, das auf „vda1“ installiert ist, muss dasselbe „/boot“ FAT32 laden:

```text
/dev/vda2
└── /boot
    ├── limine.conf
    ├── vmlinuz-6.18.44_1
    ├── initramfs-6.18.44_1.img
    └── grub/
```

Das Root-System wird weiterhin sein:

```text
/dev/vda3
```

durch:

```text
root=UUID=<UUID-da-vda3>
```

---

## Ergebnis

Das gleiche FAT32 „/boot“ wird von Limine und GRUB verwendet:

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
