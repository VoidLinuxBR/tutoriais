# Восстановить файловую систему ext4 виртуальной машины в Proxmox

Процедура проверки и восстановления файловой системы ext4 корневого раздела виртуальной машины Proxmox.

## 1. Определить виртуальную машину

Listar как виртуальные машины:

```bash
qm list
```

Просмотр конфигурации виртуальной машины:

```bash
qm config 22248
```

Определите корневой диск. В этом случае:

```text
scsi0: local-lvm:vm-22248-disk-0,iothread=1,size=50G
```

## 2. Выключите виртуальную машину

```bash
qm shutdown 22248
```

Чтобы проверить:

```bash
qm status 22248
```

Должно вернуться:

```text
status: stopped
```

## 3. Определите дисковое устройство

```bash
pvesm path local-lvm:vm-22248-disk-0
```

Результат:

```text
/dev/pve/vm-22248-disk-0
```

## 4. Проверьте разделы

```bash
fdisk -l /dev/pve/vm-22248-disk-0
```

В этом случае:

```text
/dev/pve/vm-22248-disk-0p1   BIOS boot
/dev/pve/vm-22248-disk-0p2   EFI System
/dev/pve/vm-22248-disk-0p3   Linux swap
/dev/pve/vm-22248-disk-0p4   Linux filesystem
```

Корневой раздел был `p4`.

## 5. Установите kpartx

Если не установлено:

```bash
apt update
apt install kpartx
```

## 6. Сопоставить разделы диска

```bash
kpartx -av /dev/pve/vm-22248-disk-0
```

Будут созданы следующие сопоставления:

```text
/dev/mapper/pve-vm--22248--disk--0p1
/dev/mapper/pve-vm--22248--disk--0p2
/dev/mapper/pve-vm--22248--disk--0p3
/dev/mapper/pve-vm--22248--disk--0p4
```

## 7. Подтвердите файловую систему

```bash
blkid /dev/mapper/pve-vm--22248--disk--0p4
```

Подтвердите появление:

```text
TYPE="ext4"
```

## 8. Проверьте и исправьте ext4.

```bash
e2fsck -f -v /dev/mapper/pve-vm--22248--disk--0p4
```

`e2fsck` выполнит пять шагов:

```text
Pass 1: Checking inodes, blocks, and sizes
Pass 2: Checking directory structure
Pass 3: Checking directory connectivity
Pass 4: Checking reference counts
Pass 5: Checking group summary information
```

Проверьте код возврата:

```bash
echo $?
```

Результат `0` указывает на то, что файловая система согласована.

## 9. Удаление сопоставлений

После завершения `e2fsck`:

```bash
kpartx -dv /dev/pve/vm-22248-disk-0
```

Чтобы проверить:

```bash
lsblk | grep 22248
```

## 10. Запустите виртуальную машину

```bash
qm start 22248
```

Чтобы проверить:

```bash
qm status 22248
```

## КРАТКОЕ СОДЕРЖАНИЕ

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

## Важный

Никогда не запускайте e2fsck в смонтированной файловой системе.

Перед ремонтом виртуальную машину необходимо выключить.

В приведенном выше примере корневой файловой системой был раздел p4 диска scsi0 объемом 50 ГБ.
