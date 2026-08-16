# Limine no VoidBR — BIOS + UEFI

Tutorial per installare Limine su VoidBR con supporto BIOS/Legacy e UEFI, mantenendo anche GRUB installato.

## 1. Layout della discoteca

Esempio utilizzato:

```text
/dev/vda
├── /dev/vda1   2 MiB     BIOS Boot
├── /dev/vda2   512 MiB   FAT32   /boot
└── /dev/vda3   restante  ext4    /
```

"vda1" è la partizione di avvio del BIOS.

"vda2" è FAT32 e verrà utilizzato come "/boot".

"vda3" contiene il sistema root.

Il file "/boot" FAT32 conterrà Limine, GRUB, i kernel, initramfs e altri file esistenti nel "/boot" originale.

---

## 2. Installa Limine

```bash
sudo vinstall -S
sudo vinstall -y limine
```

Controlla:

```bash
limine --version
```

File importanti:

```text
/usr/bin/limine
/usr/share/limine/BOOTX64.EFI
/usr/share/limine/limine-bios.sys
```

---

## 3. Installa efibootmgr

```bash
sudo vinstall -y efibootmgr
```

---

## 4. Montare root per accedere al vecchio /boot

L'attuale `/boot` si trova all'interno della partizione root (`vda3`).

Crea un punto di montaggio:

```bash
sudo mkdir -p /mnt/rootfs
```

Montare la radice:

```bash
sudo mount /dev/vda3 /mnt/rootfs
```

Controlla:

```bash
ls -lh /mnt/rootfs/boot
```

---

## 5. Montare temporaneamente la nuova partizione /boot

Crea il punto di montaggio:

```bash
sudo mkdir -p /mnt/newboot
```

Montare un FAT32:

```bash
sudo mount /dev/vda2 /mnt/newboot
```

Controlla:

```bash
findmnt /mnt/newboot
```

Dovrebbe apparire:

```text
/mnt/newboot  /dev/vda2  vfat
```

---

## 6. Copia tutto il vecchio /boot

Dato che verrà mantenuto anche GRUB, copia tutto il contenuto del vecchio `/boot`:

```bash
sudo cp -a /mnt/rootfs/boot/. /mnt/newboot/
```

Ciò preserva, ad esempio:

```text
grub/
memtest86+/
vmlinuz-*
initramfs-*
config-*
memtest.bin
```

Verranno copiati anche i file Limine già esistenti nel vecchio `/boot`.

Controlla:

```bash
ls -lah /mnt/newboot
```

---

## 7. Smontare la nuova partizione /boot

```bash
sudo umount /mnt/newboot
```

---

## 8. Montare FAT32 permanentemente come /boot

```bash
sudo mount /dev/vda2 /boot
```

Controlla:

```bash
findmnt /boot
```

Dovrebbe mostrare:

```text
/boot  /dev/vda2  vfat
```

Controlla:

```bash
ls -lah /boot
```

Deve essere presente il vecchio contenuto di `/boot`, inclusa la directory:

```text
/boot/grub/
```

---

## 9. Ottieni l'UUID di root

L'UUID utilizzato in `root=UUID=` deve essere l'UUID della partizione root (`vda3`).

```bash
blkid /dev/vda3
```

Esempio:

```text
/dev/vda3: UUID="a128f5c1-eb0d-4dc3-a42d-131bde041284" TYPE="ext4"
```

---

## 10. Crea limine.conf

Crea il file direttamente nel nuovo `/boot` FAT32:

```bash
sudo nano /boot/limine.conf
```

Esempio:

```text
TIMEOUT: 3
VERBOSE: no

/VoidBR (Kernel 6.18.44_1)
    PROTOCOL: linux
    PATH: boot():/vmlinuz-6.18.44_1
    MODULE_PATH: boot():/initramfs-6.18.44_1.img
    CMDLINE: root=UUID=a128f5c1-eb0d-4dc3-a42d-131bde041284 rw loglevel=4
```

Sostituire:

```text
a128f5c1-eb0d-4dc3-a42d-131bde041284
```

dal vero UUID di "vda3".

Controlla:

```bash
cat /boot/limine.conf
```

---

## 11. Installa il firmware UEFI

Crea la directory:

```bash
sudo mkdir -p /boot/EFI/limine
```

Copia l'eseguibile EFI:

```bash
sudo cp /usr/share/limine/BOOTX64.EFI /boot/EFI/limine/
```

Controlla:

```bash
ls -lh /boot/EFI/limine/BOOTX64.EFI
```

---

## 12. Installa il file Limine per il BIOS

Copia:

```bash
sudo cp /usr/share/limine/limine-bios.sys /boot/
```

Controlla:

```bash
ls -lh /boot/limine-bios.sys
```

---

## 13. Installa Limine per il BIOS

"vda1" è la partizione di avvio del BIOS.

Installare:

```bash
sudo limine bios-install /dev/vda 1
```

L'installazione dovrebbe terminare con:

```text
Limine BIOS stages installed successfully.
```

---

## 14. Crea la voce UEFI

"vda2" è la partizione EFI ed è montata come "/boot".

Crea la voce:

```bash
sudo efibootmgr -c \
    -d /dev/vda \
    -p 2 \
    -L "Limine" \
    -l '\EFI\limine\BOOTX64.EFI'
```

Controlla:

```bash
sudo efibootmgr -v
```

Qualcosa di simile a:

```text
Boot0004* Limine HD(2,GPT,...)/\EFI\limine\BOOTX64.EFI
```

---

## 15. Configura fstab

Ottieni l'UUID FAT32:

```bash
blkid /dev/vda2
```

Esempio:

```text
/dev/vda2: UUID="1234-ABCD" TYPE="vfat"
```

Modificare:

```bash
sudo nano /etc/fstab
```

Cambia la voce "vda2" in:

```text
UUID=1234-ABCD  /boot  vfat  defaults  0  2
```

La vecchia voce che montava `vda2` in `/boot/efi` deve essere rimossa o modificata.

---

## 16. Testar o fstab

Smontare:

```bash
sudo umount /boot
```

Montare utilizzando `fstab`:

```bash
sudo mount /boot
```

Controlla:

```bash
findmnt /boot
```

Dovrebbe mostrare:

```text
/boot  /dev/vda2  vfat
```

---

## 17. Controlla Limine

```bash
ls -lh /boot/limine.conf
```

```bash
ls -lh /boot/limine-bios.sys
```

```bash
ls -lh /boot/EFI/limine/BOOTX64.EFI
```

Controlla i kernel:

```bash
ls -lh /boot/vmlinuz-*
```

Controlla gli initramfs:

```bash
ls -lh /boot/initramfs-*
```

---

## 18. Dai un'occhiata a GRUB

La directory GRUB deve essere ancora presente:

```bash
ls -lah /boot/grub
```

I file GRUB presenti nel `/boot` originale sono stati conservati perché tutto il contenuto è stato copiato in FAT32.

---

## 19. Conferenza finale

```bash
findmnt /boot
```

Dovrebbe mostrare:

```text
/boot  /dev/vda2  vfat
```

Dai un'occhiata a Limine UEFI:

```bash
sudo efibootmgr -v | grep -i limine
```

Controlla la configurazione:

```bash
cat /boot/limine.conf
```

Il kernel deve essere localizzato tramite:

```text
PATH: boot():/vmlinuz-6.18.44_1
```

E o initramfs:

```text
MODULE_PATH: boot():/initramfs-6.18.44_1.img
```

---

## 20. Struttura finale

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

## 21. Testare UEFI

Scopri il numero di ingresso di Limine:

```bash
sudo efibootmgr -v
```

Supponendo che sia "0004":

```bash
sudo efibootmgr -n 0004
```

Controlla:

```bash
sudo efibootmgr | head -3
```

Dovrebbe apparire:

```text
BootNext: 0004
```

Ricomincia:

```bash
sudo reboot
```

---

## 22. Testare il BIOS

Immettere il firmware della macchina e selezionare la modalità Legacy/CSM/BIOS.

Limine installato su `vda1` deve caricare lo stesso `/boot` FAT32:

```text
/dev/vda2
└── /boot
    ├── limine.conf
    ├── vmlinuz-6.18.44_1
    ├── initramfs-6.18.44_1.img
    └── grub/
```

L'apparato radicale continuerà ad essere:

```text
/dev/vda3
```

Attraverso:

```text
root=UUID=<UUID-da-vda3>
```

---

## Risultato

Lo stesso FAT32 `/boot` è utilizzato da Limine e GRUB:

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
