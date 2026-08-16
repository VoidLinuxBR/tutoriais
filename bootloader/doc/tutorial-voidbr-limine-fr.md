# Limine no VoidBR — BIOS + UEFI

Tutoriel pour installer Limine sur VoidBR avec le support BIOS/Legacy et UEFI, en gardant également GRUB installé.

## 1. Mise en page pour une discothèque

Exemple utilisé :

```text
/dev/vda
├── /dev/vda1   2 MiB     BIOS Boot
├── /dev/vda2   512 MiB   FAT32   /boot
└── /dev/vda3   restante  ext4    /
```

« vda1 » est la partition de démarrage du BIOS.

`vda2` est FAT32 et sera utilisé comme `/boot`.

`vda3` contient le système racine.

Le `/boot` FAT32 contiendra le Limine, GRUB, les noyaux, initramfs et d'autres fichiers existant dans le `/boot` d'origine.

---

## 2. Installez Limine

```bash
sudo vinstall -S
sudo vinstall -y limine
```

Vérifiez-le:

```bash
limine --version
```

Fichiers importants :

```text
/usr/bin/limine
/usr/share/limine/BOOTX64.EFI
/usr/share/limine/limine-bios.sys
```

---

## 3. Installez efibootmgr

```bash
sudo vinstall -y efibootmgr
```

---

## 4. Montez root pour accéder à l'ancien /boot

Le `/boot` actuel se trouve à l'intérieur de la partition racine (`vda3`).

Créez un point de montage :

```bash
sudo mkdir -p /mnt/rootfs
```

Montez la racine :

```bash
sudo mount /dev/vda3 /mnt/rootfs
```

Vérifiez-le:

```bash
ls -lh /mnt/rootfs/boot
```

---

## 5. Montez temporairement la nouvelle partition /boot

Créez le point de montage :

```bash
sudo mkdir -p /mnt/newboot
```

Montez un FAT32 :

```bash
sudo mount /dev/vda2 /mnt/newboot
```

Vérifiez-le:

```bash
findmnt /mnt/newboot
```

Il devrait apparaître :

```text
/mnt/newboot  /dev/vda2  vfat
```

---

## 6. Copiez tous les anciens /boot

Puisque GRUB sera également maintenu, copiez tout le contenu de l'ancien `/boot` :

```bash
sudo cp -a /mnt/rootfs/boot/. /mnt/newboot/
```

Cela préserve par exemple :

```text
grub/
memtest86+/
vmlinuz-*
initramfs-*
config-*
memtest.bin
```

Les fichiers Limine qui existent déjà dans l'ancien `/boot` seront également copiés.

Vérifiez-le:

```bash
ls -lah /mnt/newboot
```

---

## 7. Démontez la nouvelle partition /boot

```bash
sudo umount /mnt/newboot
```

---

## 8. Montez FAT32 de manière permanente en tant que /boot

```bash
sudo mount /dev/vda2 /boot
```

Vérifiez-le:

```bash
findmnt /boot
```

Il devrait montrer :

```text
/boot  /dev/vda2  vfat
```

Vérifiez-le:

```bash
ls -lah /boot
```

L'ancien contenu de `/boot` doit être présent, y compris le répertoire :

```text
/boot/grub/
```

---

## 9. Obtenez l'UUID racine

L'UUID utilisé dans `root=UUID=` doit être l'UUID de la partition racine (`vda3`).

```bash
blkid /dev/vda3
```

Exemple:

```text
/dev/vda3: UUID="a128f5c1-eb0d-4dc3-a42d-131bde041284" TYPE="ext4"
```

---

## 10. Créez limine.conf

Créez le fichier directement dans le nouveau `/boot` FAT32 :

```bash
sudo nano /boot/limine.conf
```

Exemple:

```text
TIMEOUT: 3
VERBOSE: no

/VoidBR (Kernel 6.18.44_1)
    PROTOCOL: linux
    PATH: boot():/vmlinuz-6.18.44_1
    MODULE_PATH: boot():/initramfs-6.18.44_1.img
    CMDLINE: root=UUID=a128f5c1-eb0d-4dc3-a42d-131bde041284 rw loglevel=4
```

Remplacer:

```text
a128f5c1-eb0d-4dc3-a42d-131bde041284
```

par le véritable UUID de `vda3`.

Vérifiez-le:

```bash
cat /boot/limine.conf
```

---

## 11. Installez le micrologiciel UEFI

Créez le répertoire :

```bash
sudo mkdir -p /boot/EFI/limine
```

Copiez l'exécutable EFI :

```bash
sudo cp /usr/share/limine/BOOTX64.EFI /boot/EFI/limine/
```

Vérifiez-le:

```bash
ls -lh /boot/EFI/limine/BOOTX64.EFI
```

---

## 12. Installez le fichier Limine pour le BIOS

Copie:

```bash
sudo cp /usr/share/limine/limine-bios.sys /boot/
```

Vérifiez-le:

```bash
ls -lh /boot/limine-bios.sys
```

---

## 13. Installez Limine pour le BIOS

« vda1 » est la partition de démarrage du BIOS.

Installer:

```bash
sudo limine bios-install /dev/vda 1
```

L'installation doit se terminer par :

```text
Limine BIOS stages installed successfully.
```

---

## 14. Créez l'entrée UEFI

« vda2 » est la partition EFI et est montée en tant que « /boot ».

Créez l'entrée :

```bash
sudo efibootmgr -c \
    -d /dev/vda \
    -p 2 \
    -L "Limine" \
    -l '\EFI\limine\BOOTX64.EFI'
```

Vérifiez-le:

```bash
sudo efibootmgr -v
```

Quelque chose de similaire à :

```text
Boot0004* Limine HD(2,GPT,...)/\EFI\limine\BOOTX64.EFI
```

---

## 15. Configurer fstab

Obtenez l'UUID FAT32 :

```bash
blkid /dev/vda2
```

Exemple:

```text
/dev/vda2: UUID="1234-ABCD" TYPE="vfat"
```

Modifier:

```bash
sudo nano /etc/fstab
```

Remplacez l'entrée `vda2` par :

```text
UUID=1234-ABCD  /boot  vfat  defaults  0  2
```

L'ancienne entrée qui a monté « vda2 » dans « /boot/efi » doit être supprimée ou modifiée.

---

## 16. Test de poignard

Démonter:

```bash
sudo umount /boot
```

Monter en utilisant `fstab` :

```bash
sudo mount /boot
```

Vérifiez-le:

```bash
findmnt /boot
```

Il devrait montrer :

```text
/boot  /dev/vda2  vfat
```

---

## 17. Vérifiez la limite

```bash
ls -lh /boot/limine.conf
```

```bash
ls -lh /boot/limine-bios.sys
```

```bash
ls -lh /boot/EFI/limine/BOOTX64.EFI
```

Découvrez les noyaux :

```bash
ls -lh /boot/vmlinuz-*
```

Vérifiez les initramfs :

```bash
ls -lh /boot/initramfs-*
```

---

## 18. Découvrez GRUB

Le répertoire GRUB doit toujours être présent :

```bash
ls -lah /boot/grub
```

Les fichiers GRUB qui se trouvaient dans le « /boot » d'origine ont été conservés car tout le contenu a été copié en FAT32.

---

## 19. Conférence finale

```bash
findmnt /boot
```

Il devrait montrer :

```text
/boot  /dev/vda2  vfat
```

Découvrez Limine UEFI :

```bash
sudo efibootmgr -v | grep -i limine
```

Vérifiez la configuration :

```bash
cat /boot/limine.conf
```

Le noyau doit être localisé via :

```text
PATH: boot():/vmlinuz-6.18.44_1
```

E o initramfs :

```text
MODULE_PATH: boot():/initramfs-6.18.44_1.img
```

---

## 20. Structure finale

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

## 21. Test de l'UEFI

Découvrez le numéro d’entrée Limine :

```bash
sudo efibootmgr -v
```

En supposant qu'il s'agisse de « 0004 » :

```bash
sudo efibootmgr -n 0004
```

Vérifiez-le:

```bash
sudo efibootmgr | head -3
```

Il devrait apparaître :

```text
BootNext: 0004
```

Redémarrage:

```bash
sudo reboot
```

---

## 22. Tester le BIOS

Entrez le micrologiciel de la machine et sélectionnez le mode Legacy/CSM/BIOS.

Limine installé sur `vda1` doit charger le même `/boot` FAT32 :

```text
/dev/vda2
└── /boot
    ├── limine.conf
    ├── vmlinuz-6.18.44_1
    ├── initramfs-6.18.44_1.img
    └── grub/
```

Le système racine continuera à être :

```text
/dev/vda3
```

à travers:

```text
root=UUID=<UUID-da-vda3>
```

---

## Résultat

Le même `/boot` FAT32 est utilisé par Limine et GRUB :

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
