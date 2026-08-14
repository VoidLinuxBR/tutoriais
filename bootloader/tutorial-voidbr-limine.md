# Limine no VoidBR — BIOS + UEFI

Tutorial para instalar o Limine no VoidBR com suporte a BIOS/Legacy e UEFI, mantendo também o GRUB instalado.

## 1. Layout do disco

Exemplo utilizado:

```text
/dev/vda
├── /dev/vda1   2 MiB     BIOS Boot
├── /dev/vda2   512 MiB   FAT32   /boot
└── /dev/vda3   restante  ext4    /
```

A `vda1` é a partição BIOS Boot.

A `vda2` é FAT32 e será usada como `/boot`.

A `vda3` contém o sistema raiz.

O `/boot` FAT32 conterá os arquivos do Limine, GRUB, kernels, initramfs e demais arquivos existentes no `/boot` original.

---

## 2. Instalar o Limine

```bash
sudo vinstall -S
sudo vinstall -y limine
```

Confira:

```bash
limine --version
```

Arquivos importantes:

```text
/usr/bin/limine
/usr/share/limine/BOOTX64.EFI
/usr/share/limine/limine-bios.sys
```

---

## 3. Instalar o efibootmgr

```bash
sudo vinstall -y efibootmgr
```

---

## 4. Montar a raiz para acessar o /boot antigo

O `/boot` atual está dentro da partição raiz (`vda3`).

Crie um ponto de montagem:

```bash
sudo mkdir -p /mnt/rootfs
```

Monte a raiz:

```bash
sudo mount /dev/vda3 /mnt/rootfs
```

Confira:

```bash
ls -lh /mnt/rootfs/boot
```

---

## 5. Montar a nova partição /boot temporariamente

Crie o ponto de montagem:

```bash
sudo mkdir -p /mnt/newboot
```

Monte a FAT32:

```bash
sudo mount /dev/vda2 /mnt/newboot
```

Confira:

```bash
findmnt /mnt/newboot
```

Deve aparecer:

```text
/mnt/newboot  /dev/vda2  vfat
```

---

## 6. Copiar todo o /boot antigo

Como o GRUB também será mantido, copie todo o conteúdo do `/boot` antigo:

```bash
sudo cp -a /mnt/rootfs/boot/. /mnt/newboot/
```

Isso preserva, por exemplo:

```text
grub/
memtest86+/
vmlinuz-*
initramfs-*
config-*
memtest.bin
```

Também serão copiados os arquivos do Limine que já existirem no `/boot` antigo.

Confira:

```bash
ls -lah /mnt/newboot
```

---

## 7. Desmontar a nova partição /boot

```bash
sudo umount /mnt/newboot
```

---

## 8. Montar a FAT32 definitivamente como /boot

```bash
sudo mount /dev/vda2 /boot
```

Confira:

```bash
findmnt /boot
```

Deve mostrar:

```text
/boot  /dev/vda2  vfat
```

Confira:

```bash
ls -lah /boot
```

O conteúdo antigo do `/boot` deve estar presente, incluindo o diretório:

```text
/boot/grub/
```

---

## 9. Obter o UUID da raiz

O UUID utilizado em `root=UUID=` deve ser o UUID da partição raiz (`vda3`).

```bash
blkid /dev/vda3
```

Exemplo:

```text
/dev/vda3: UUID="a128f5c1-eb0d-4dc3-a42d-131bde041284" TYPE="ext4"
```

---

## 10. Criar o limine.conf

Crie o arquivo diretamente no novo `/boot` FAT32:

```bash
sudo nano /boot/limine.conf
```

Exemplo:

```text
TIMEOUT: 3
VERBOSE: no

/VoidBR (Kernel 6.18.44_1)
    PROTOCOL: linux
    PATH: boot():/vmlinuz-6.18.44_1
    MODULE_PATH: boot():/initramfs-6.18.44_1.img
    CMDLINE: root=UUID=a128f5c1-eb0d-4dc3-a42d-131bde041284 rw loglevel=4
```

Substitua:

```text
a128f5c1-eb0d-4dc3-a42d-131bde041284
```

pelo UUID real da `vda3`.

Confira:

```bash
cat /boot/limine.conf
```

---

## 11. Instalar o Limine UEFI

Crie o diretório:

```bash
sudo mkdir -p /boot/EFI/limine
```

Copie o executável EFI:

```bash
sudo cp /usr/share/limine/BOOTX64.EFI /boot/EFI/limine/
```

Confira:

```bash
ls -lh /boot/EFI/limine/BOOTX64.EFI
```

---

## 12. Instalar o arquivo do Limine para BIOS

Copie:

```bash
sudo cp /usr/share/limine/limine-bios.sys /boot/
```

Confira:

```bash
ls -lh /boot/limine-bios.sys
```

---

## 13. Instalar o Limine para BIOS

A `vda1` é a BIOS Boot Partition.

Instale:

```bash
sudo limine bios-install /dev/vda 1
```

A instalação deve terminar com:

```text
Limine BIOS stages installed successfully.
```

---

## 14. Criar a entrada UEFI

A `vda2` é a partição EFI e está montada como `/boot`.

Crie a entrada:

```bash
sudo efibootmgr -c \
    -d /dev/vda \
    -p 2 \
    -L "Limine" \
    -l '\EFI\limine\BOOTX64.EFI'
```

Confira:

```bash
sudo efibootmgr -v
```

Deve aparecer algo semelhante a:

```text
Boot0004* Limine HD(2,GPT,...)/\EFI\limine\BOOTX64.EFI
```

---

## 15. Configurar o fstab

Obtenha o UUID da FAT32:

```bash
blkid /dev/vda2
```

Exemplo:

```text
/dev/vda2: UUID="1234-ABCD" TYPE="vfat"
```

Edite:

```bash
sudo nano /etc/fstab
```

Altere a entrada da `vda2` para:

```text
UUID=1234-ABCD  /boot  vfat  defaults  0  2
```

A antiga entrada que montava a `vda2` em `/boot/efi` deve ser removida ou alterada.

---

## 16. Testar o fstab

Desmonte:

```bash
sudo umount /boot
```

Monte usando o `fstab`:

```bash
sudo mount /boot
```

Confira:

```bash
findmnt /boot
```

Deve mostrar:

```text
/boot  /dev/vda2  vfat
```

---

## 17. Conferir Limine

```bash
ls -lh /boot/limine.conf
```

```bash
ls -lh /boot/limine-bios.sys
```

```bash
ls -lh /boot/EFI/limine/BOOTX64.EFI
```

Confira os kernels:

```bash
ls -lh /boot/vmlinuz-*
```

Confira os initramfs:

```bash
ls -lh /boot/initramfs-*
```

---

## 18. Conferir o GRUB

O diretório do GRUB deve continuar presente:

```bash
ls -lah /boot/grub
```

Os arquivos do GRUB que estavam no `/boot` original foram preservados porque todo o conteúdo foi copiado para a FAT32.

---

## 19. Conferência final

```bash
findmnt /boot
```

Deve mostrar:

```text
/boot  /dev/vda2  vfat
```

Confira o Limine UEFI:

```bash
sudo efibootmgr -v | grep -i limine
```

Confira a configuração:

```bash
cat /boot/limine.conf
```

O kernel deve estar sendo localizado através de:

```text
PATH: boot():/vmlinuz-6.18.44_1
```

E o initramfs:

```text
MODULE_PATH: boot():/initramfs-6.18.44_1.img
```

---

## 20. Estrutura final

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

## 21. Testar UEFI

Descubra o número da entrada Limine:

```bash
sudo efibootmgr -v
```

Supondo que seja `0004`:

```bash
sudo efibootmgr -n 0004
```

Confira:

```bash
sudo efibootmgr | head -3
```

Deve aparecer:

```text
BootNext: 0004
```

Reinicie:

```bash
sudo reboot
```

---

## 22. Testar BIOS

Entre no firmware da máquina e selecione o modo Legacy/CSM/BIOS.

O Limine instalado na `vda1` deverá carregar o mesmo `/boot` FAT32:

```text
/dev/vda2
└── /boot
    ├── limine.conf
    ├── vmlinuz-6.18.44_1
    ├── initramfs-6.18.44_1.img
    └── grub/
```

O sistema raiz continuará sendo:

```text
/dev/vda3
```

através do:

```text
root=UUID=<UUID-da-vda3>
```

---

## Resultado

O mesmo `/boot` FAT32 é utilizado pelo Limine e pelo GRUB:

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
