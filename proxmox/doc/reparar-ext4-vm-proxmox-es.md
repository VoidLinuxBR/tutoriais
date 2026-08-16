# Reparar el sistema de archivos ext4 de una VM en Proxmox

Procedimiento para verificar y reparar el sistema de archivos ext4 de la partición raíz de una VM Proxmox.

## 1. Identificar a VM

Listar como VM:

```bash
qm list
```

Ver la configuración de la máquina virtual:

```bash
qm config 22248
```

Identifique el disco raíz. En este caso:

```text
scsi0: local-lvm:vm-22248-disk-0,iothread=1,size=50G
```

## 2. Apague la máquina virtual

```bash
qm shutdown 22248
```

Para comprobar:

```bash
qm status 22248
```

Debería regresar:

```text
status: stopped
```

## 3. Identifique el dispositivo de disco

```bash
pvesm path local-lvm:vm-22248-disk-0
```

Resultado:

```text
/dev/pve/vm-22248-disk-0
```

## 4. Verificar particiones

```bash
fdisk -l /dev/pve/vm-22248-disk-0
```

En este caso:

```text
/dev/pve/vm-22248-disk-0p1   BIOS boot
/dev/pve/vm-22248-disk-0p2   EFI System
/dev/pve/vm-22248-disk-0p3   Linux swap
/dev/pve/vm-22248-disk-0p4   Linux filesystem
```

La partición raíz era "p4".

## 5. Instale kpartx

Si no está instalado:

```bash
apt update
apt install kpartx
```

## 6. Mapear particiones de disco

```bash
kpartx -av /dev/pve/vm-22248-disk-0
```

Se crearán las siguientes asignaciones:

```text
/dev/mapper/pve-vm--22248--disk--0p1
/dev/mapper/pve-vm--22248--disk--0p2
/dev/mapper/pve-vm--22248--disk--0p3
/dev/mapper/pve-vm--22248--disk--0p4
```

## 7. Confirme el sistema de archivos.

```bash
blkid /dev/mapper/pve-vm--22248--disk--0p4
```

Confirmar que aparece:

```text
TYPE="ext4"
```

## 8. Verificar y reparar ext4

```bash
e2fsck -f -v /dev/mapper/pve-vm--22248--disk--0p4
```

`e2fsck` realizará los cinco pasos:

```text
Pass 1: Checking inodes, blocks, and sizes
Pass 2: Checking directory structure
Pass 3: Checking directory connectivity
Pass 4: Checking reference counts
Pass 5: Checking group summary information
```

Verifique el código de retorno:

```bash
echo $?
```

El resultado "0" indica que el sistema de archivos es consistente.

## 9. Eliminar asignaciones

Después de terminar `e2fsck`:

```bash
kpartx -dv /dev/pve/vm-22248-disk-0
```

Para comprobar:

```bash
lsblk | grep 22248
```

## 10. Inicie la máquina virtual

```bash
qm start 22248
```

Para comprobar:

```bash
qm status 22248
```

## RESUMEN

```bash
qm shutdown 22248
qm status 22248

pvesm path local-lvm:vm-22248-disk-0

fdisk -l /dev/pve/vm-22248-disk-0

apt update
apt install kpartx

kpartx -av /dev/pve/vm-22248-disk-0

blkid /dev/mapper/pve-vm--22248--disk--0p4

e2fsck -f -v /dev/mapper/pve-vm--22248--disk--0p4

echo $?

kpartx -dv /dev/pve/vm-22248-disk-0

lsblk | grep 22248

qm start 22248
```

## Importante

Nunca ejecute `e2fsck` en un sistema de archivos montado.

La máquina virtual debe estar apagada antes de la reparación.

En el ejemplo anterior, el sistema de archivos raíz era la partición `p4` del disco `scsi0` de 50 GB.
