# Reparar filesystem ext4 de uma VM no Proxmox

Procedimento para verificar e reparar o filesystem ext4 da partição raiz de uma VM Proxmox.

## 1. Identificar a VM

Listar as VMs:

```bash
qm list
```

Ver a configuração da VM:

```bash
qm config 22248
```

Identificar o disco raiz. Neste caso:

```text
scsi0: local-lvm:vm-22248-disk-0,iothread=1,size=50G
```

## 2. Desligar a VM

```bash
qm shutdown 22248
```

Verificar:

```bash
qm status 22248
```

Deve retornar:

```text
status: stopped
```

## 3. Identificar o dispositivo do disco

```bash
pvesm path local-lvm:vm-22248-disk-0
```

Resultado:

```text
/dev/pve/vm-22248-disk-0
```

## 4. Verificar as partições

```bash
fdisk -l /dev/pve/vm-22248-disk-0
```

Neste caso:

```text
/dev/pve/vm-22248-disk-0p1   BIOS boot
/dev/pve/vm-22248-disk-0p2   EFI System
/dev/pve/vm-22248-disk-0p3   Linux swap
/dev/pve/vm-22248-disk-0p4   Linux filesystem
```

A partição raiz era a `p4`.

## 5. Instalar o kpartx

Caso não esteja instalado:

```bash
apt update
apt install kpartx
```

## 6. Mapear as partições do disco

```bash
kpartx -av /dev/pve/vm-22248-disk-0
```

Serão criados os mapeamentos:

```text
/dev/mapper/pve-vm--22248--disk--0p1
/dev/mapper/pve-vm--22248--disk--0p2
/dev/mapper/pve-vm--22248--disk--0p3
/dev/mapper/pve-vm--22248--disk--0p4
```

## 7. Confirmar o filesystem

```bash
blkid /dev/mapper/pve-vm--22248--disk--0p4
```

Confirmar que aparece:

```text
TYPE="ext4"
```

## 8. Verificar e reparar o ext4

```bash
e2fsck -f -v /dev/mapper/pve-vm--22248--disk--0p4
```

O `e2fsck` executará as cinco etapas:

```text
Pass 1: Checking inodes, blocks, and sizes
Pass 2: Checking directory structure
Pass 3: Checking directory connectivity
Pass 4: Checking reference counts
Pass 5: Checking group summary information
```

Verificar o código de retorno:

```bash
echo $?
```

Resultado `0` indica que o filesystem está consistente.

## 9. Remover os mapeamentos

Após finalizar o `e2fsck`:

```bash
kpartx -dv /dev/pve/vm-22248-disk-0
```

Verificar:

```bash
lsblk | grep 22248
```

## 10. Iniciar a VM

```bash
qm start 22248
```

Verificar:

```bash
qm status 22248
```

## Resumo

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

Nunca execute `e2fsck` em um filesystem montado.

A VM deve estar desligada antes do reparo.

No exemplo acima, o filesystem raiz era a partição `p4` do disco `scsi0` de 50 GB.
