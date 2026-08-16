# Limine no VoidBR — BIOS + UEFI

Tutorial para instalar Limine en VoidBR con soporte BIOS/Legacy y UEFI, manteniendo también GRUB instalado.

## 1. Diseño de discoteca.

Ejemplo utilizado:

```text
/dev/vda
├── /dev/vda1   2 MiB     BIOS Boot
├── /dev/vda2   512 MiB   FAT32   /boot
└── /dev/vda3   restante  ext4    /
```

`vda1` es la partición de arranque del BIOS.

`vda2` es FAT32 y se usará como `/boot`.

`vda3` contiene el sistema raíz.

El FAT32 `/boot` contendrá Limine, GRUB, kernels, initramfs y otros archivos existentes en el `/boot` original.

---

## 2. Instalar Limine

```bash
sudo vinstall -S
sudo vinstall -y limine
```

Échale un vistazo:

```bash
limine --version
```

Archivos importantes:

```text
/usr/bin/limine
/usr/share/limine/BOOTX64.EFI
/usr/share/limine/limine-bios.sys
```

---

## 3. Instale efibootmgr

```bash
sudo vinstall -y efibootmgr
```

---

## 4. Monte root para acceder al antiguo /boot

El `/boot` actual está dentro de la partición raíz (`vda3`).

Crea un punto de montaje:

```bash
sudo mkdir -p /mnt/rootfs
```

Montar la raíz:

```bash
sudo mount /dev/vda3 /mnt/rootfs
```

Échale un vistazo:

```bash
ls -lh /mnt/rootfs/boot
```

---

## 5. Monte la nueva partición /boot temporalmente

Crea el punto de montaje:

```bash
sudo mkdir -p /mnt/newboot
```

Monte en FAT32:

```bash
sudo mount /dev/vda2 /mnt/newboot
```

Échale un vistazo:

```bash
findmnt /mnt/newboot
```

Debería aparecer:

```text
/mnt/newboot  /dev/vda2  vfat
```

---

## 6. Copie todo el antiguo /boot

Dado que GRUB también se mantendrá, copie todo el contenido del antiguo `/boot`:

```bash
sudo cp -a /mnt/rootfs/boot/. /mnt/newboot/
```

Esto preserva, por ejemplo:

```text
grub/
memtest86+/
vmlinuz-*
initramfs-*
config-*
memtest.bin
```

También se copiarán los archivos Limine que ya existen en el antiguo `/boot`.

Échale un vistazo:

```bash
ls -lah /mnt/newboot
```

---

## 7. Desmonte la nueva partición /boot

```bash
sudo umount /mnt/newboot
```

---

## 8. Monte FAT32 permanentemente como /boot

```bash
sudo mount /dev/vda2 /boot
```

Échale un vistazo:

```bash
findmnt /boot
```

Debería mostrar:

```text
/boot  /dev/vda2  vfat
```

Échale un vistazo:

```bash
ls -lah /boot
```

El contenido antiguo de `/boot` debe estar presente, incluido el directorio:

```text
/boot/grub/
```

---

## 9. Obtenga el UUID raíz

El UUID utilizado en `root=UUID=` debe ser el UUID de la partición raíz (`vda3`).

```bash
blkid /dev/vda3
```

Ejemplo:

```text
/dev/vda3: UUID="a128f5c1-eb0d-4dc3-a42d-131bde041284" TYPE="ext4"
```

---

## 10. Crea limine.conf

Cree el archivo directamente en el nuevo `/boot` FAT32:

```bash
sudo nano /boot/limine.conf
```

Ejemplo:

```text
TIMEOUT: 3
VERBOSE: no

/VoidBR (Kernel 6.18.44_1)
    PROTOCOL: linux
    PATH: boot():/vmlinuz-6.18.44_1
    MODULE_PATH: boot():/initramfs-6.18.44_1.img
    CMDLINE: root=UUID=a128f5c1-eb0d-4dc3-a42d-131bde041284 rw loglevel=4
```

Reemplazar:

```text
a128f5c1-eb0d-4dc3-a42d-131bde041284
```

por el UUID real de `vda3`.

Échale un vistazo:

```bash
cat /boot/limine.conf
```

---

## 11. Instale el firmware UEFI

Crea el directorio:

```bash
sudo mkdir -p /boot/EFI/limine
```

Copie el ejecutable EFI:

```bash
sudo cp /usr/share/limine/BOOTX64.EFI /boot/EFI/limine/
```

Échale un vistazo:

```bash
ls -lh /boot/EFI/limine/BOOTX64.EFI
```

---

## 12. Instale el archivo Limine para BIOS

Copiar:

```bash
sudo cp /usr/share/limine/limine-bios.sys /boot/
```

Échale un vistazo:

```bash
ls -lh /boot/limine-bios.sys
```

---

## 13. Instale Limine para BIOS

`vda1` es la partición de arranque del BIOS.

Instalar:

```bash
sudo limine bios-install /dev/vda 1
```

La instalación debería finalizar con:

```text
Limine BIOS stages installed successfully.
```

---

## 14. Crea la entrada UEFI.

`vda2` es la partición EFI y está montada como `/boot`.

Crea la entrada:

```bash
sudo efibootmgr -c \
    -d /dev/vda \
    -p 2 \
    -L "Limine" \
    -l '\EFI\limine\BOOTX64.EFI'
```

Échale un vistazo:

```bash
sudo efibootmgr -v
```

Algo parecido a:

```text
Boot0004* Limine HD(2,GPT,...)/\EFI\limine\BOOTX64.EFI
```

---

## 15. Configurar fstab

Obtenga el UUID FAT32:

```bash
blkid /dev/vda2
```

Ejemplo:

```text
/dev/vda2: UUID="1234-ABCD" TYPE="vfat"
```

Editar:

```bash
sudo nano /etc/fstab
```

Cambie la entrada `vda2` a:

```text
UUID=1234-ABCD  /boot  vfat  defaults  0  2
```

La entrada anterior que montaba `vda2` en `/boot/efi` debe eliminarse o cambiarse.

---

## 16. Testar de puñalada

Desmontar:

```bash
sudo umount /boot
```

Montar usando `fstab`:

```bash
sudo mount /boot
```

Échale un vistazo:

```bash
findmnt /boot
```

Debería mostrar:

```text
/boot  /dev/vda2  vfat
```

---

## 17. Verificar la línea

```bash
ls -lh /boot/limine.conf
```

```bash
ls -lh /boot/limine-bios.sys
```

```bash
ls -lh /boot/EFI/limine/BOOTX64.EFI
```

Mira los núcleos:

```bash
ls -lh /boot/vmlinuz-*
```

Verifique los initramfs:

```bash
ls -lh /boot/initramfs-*
```

---

## 18. Echa un vistazo a GRUB

El directorio GRUB aún debe estar presente:

```bash
ls -lah /boot/grub
```

Los archivos GRUB que estaban en el `/boot` original se conservaron porque todo el contenido se copió a FAT32.

---

## 19. Conferencia final

```bash
findmnt /boot
```

Debería mostrar:

```text
/boot  /dev/vda2  vfat
```

Echa un vistazo a Limine UEFI:

```bash
sudo efibootmgr -v | grep -i limine
```

Verifique la configuración:

```bash
cat /boot/limine.conf
```

El kernel debe localizarse a través de:

```text
PATH: boot():/vmlinuz-6.18.44_1
```

E o initramfs:

```text
MODULE_PATH: boot():/initramfs-6.18.44_1.img
```

---

## 20. Estructura definitiva

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

## 21. Prueba de UEFI

Descubra el número de entrada de Limine:

```bash
sudo efibootmgr -v
```

Suponiendo que sea `0004`:

```bash
sudo efibootmgr -n 0004
```

Échale un vistazo:

```bash
sudo efibootmgr | head -3
```

Debería aparecer:

```text
BootNext: 0004
```

Reanudar:

```bash
sudo reboot
```

---

## 22. Pruebe el BIOS

Ingrese el firmware de la máquina y seleccione el modo Legacy/CSM/BIOS.

Limine instalado en `vda1` debe cargar el mismo `/boot` FAT32:

```text
/dev/vda2
└── /boot
    ├── limine.conf
    ├── vmlinuz-6.18.44_1
    ├── initramfs-6.18.44_1.img
    └── grub/
```

El sistema raíz seguirá siendo:

```text
/dev/vda3
```

a través de:

```text
root=UUID=<UUID-da-vda3>
```

---

## Resultado

Limine y GRUB utilizan el mismo FAT32 `/boot`:

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
