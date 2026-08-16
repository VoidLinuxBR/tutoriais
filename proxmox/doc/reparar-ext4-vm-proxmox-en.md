# Repair ext4 filesystem of a VM in Proxmox

Procedure to check and repair the ext4 filesystem of the root partition of a Proxmox VM.

## 1. Identify VM

Listar as VMs:

```bash
qm list
```

View VM configuration:

```bash
qm config 22248
```

Identify the root disk. In this case:

```text
scsi0: local-lvm:vm-22248-disk-0,iothread=1,size=50G
```

## 2. Shut down the VM

```bash
qm shutdown 22248
```

To check:

```bash
qm status 22248
```

Should return:

```text
status: stopped
```

## 3. Identify the disk device

```bash
pvesm path local-lvm:vm-22248-disk-0
```

Result:

```text
/dev/pve/vm-22248-disk-0
```

## 4. Check partitions

```bash
fdisk -l /dev/pve/vm-22248-disk-0
```

In this case:

```text
/dev/pve/vm-22248-disk-0p1   BIOS boot
/dev/pve/vm-22248-disk-0p2   EFI System
/dev/pve/vm-22248-disk-0p3   Linux swap
/dev/pve/vm-22248-disk-0p4   Linux filesystem
```

The root partition was `p4`.

## 5. Install kpartx

If not installed:

```bash
apt update
apt install kpartx
```

## 6. Map disk partitions

```bash
kpartx -av /dev/pve/vm-22248-disk-0
```

The following mappings will be created:

```text
/dev/mapper/pve-vm--22248--disk--0p1
/dev/mapper/pve-vm--22248--disk--0p2
/dev/mapper/pve-vm--22248--disk--0p3
/dev/mapper/pve-vm--22248--disk--0p4
```

## 7. Confirm the filesystem

```bash
blkid /dev/mapper/pve-vm--22248--disk--0p4
```

Confirm that it appears:

```text
TYPE="ext4"
```

## 8. Check and repair ext4

```bash
e2fsck -f -v /dev/mapper/pve-vm--22248--disk--0p4
```

`e2fsck` will perform the five steps:

```text
Pass 1: Checking inodes, blocks, and sizes
Pass 2: Checking directory structure
Pass 3: Checking directory connectivity
Pass 4: Checking reference counts
Pass 5: Checking group summary information
```

Check return code:

```bash
echo $?
```

Result `0` indicates that the filesystem is consistent.

## 9. Remove mappings

After finishing `e2fsck`:

```bash
kpartx -dv /dev/pve/vm-22248-disk-0
```

To check:

```bash
lsblk | grep 22248
```

## 10. Start the VM

```bash
qm start 22248
```

To check:

```bash
qm status 22248
```

## SUMMARY

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

## Important

Never run `e2fsck` on a mounted filesystem.

The VM must be powered off before repair.

In the example above, the root filesystem was the `p4` partition of the 50 GB `scsi0` disk.
