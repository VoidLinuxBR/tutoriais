# 在 Proxmox 中修复虚拟机的 ext4 文件系统

检查和修复 Proxmox VM 根分区的 ext4 文件系统的过程。

## 1. 识别虚拟机

列为虚拟机：

```bash
qm list
```

查看虚拟机配置：

```bash
qm config 22248
```

识别根磁盘。在这种情况下：

```text
scsi0: local-lvm:vm-22248-disk-0,iothread=1,size=50G
```

## 2. 关闭虚拟机

```bash
qm shutdown 22248
```

要检查：

```bash
qm status 22248
```

应该返回：

```text
status: stopped
```

## 3.识别磁盘设备

```bash
pvesm path local-lvm:vm-22248-disk-0
```

结果：

```text
/dev/pve/vm-22248-disk-0
```

## 4.检查分区

```bash
fdisk -l /dev/pve/vm-22248-disk-0
```

在这种情况下：

```text
/dev/pve/vm-22248-disk-0p1   BIOS boot
/dev/pve/vm-22248-disk-0p2   EFI System
/dev/pve/vm-22248-disk-0p3   Linux swap
/dev/pve/vm-22248-disk-0p4   Linux filesystem
```

根分区是“p4”。

## 5.安装kpartx

如果没有安装：

```bash
apt update
apt install kpartx
```

## 6. 映射磁盘分区

```bash
kpartx -av /dev/pve/vm-22248-disk-0
```

将创建以下映射：

```text
/dev/mapper/pve-vm--22248--disk--0p1
/dev/mapper/pve-vm--22248--disk--0p2
/dev/mapper/pve-vm--22248--disk--0p3
/dev/mapper/pve-vm--22248--disk--0p4
```

## 7. 确认文件系统

```bash
blkid /dev/mapper/pve-vm--22248--disk--0p4
```

确认它出现：

```text
TYPE="ext4"
```

## 8.检查并修复ext4

```bash
e2fsck -f -v /dev/mapper/pve-vm--22248--disk--0p4
```

`e2fsck` 将执行五个步骤：

```text
Pass 1: Checking inodes, blocks, and sizes
Pass 2: Checking directory structure
Pass 3: Checking directory connectivity
Pass 4: Checking reference counts
Pass 5: Checking group summary information
```

检查返回码：

```bash
echo $?
```

结果“0”表示文件系统是一致的。

## 9. 删除映射

完成“e2fsck”后：

```bash
kpartx -dv /dev/pve/vm-22248-disk-0
```

要检查：

```bash
lsblk | grep 22248
```

## 10.启动虚拟机

```bash
qm start 22248
```

要检查：

```bash
qm status 22248
```

## 概括

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

## 重要的

切勿在已安装的文件系统上运行“e2fsck”。

修复前必须关闭虚拟机电源。

在上面的示例中，根文件系统是 50 GB `scsi0` 磁盘的 `p4` 分区。
