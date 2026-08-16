# 在 Proxmox 中修復虛擬機器的 ext4 檔案系統

檢查並修復 Proxmox VM 根分割區的 ext4 檔案系統的過程。

## 1. 辨識虛擬機

列為虛擬機器：

```bash
qm list
```

查看虛擬機器配置：

```bash
qm config 22248
```

識別根磁碟。在這種情況下：

```text
scsi0: local-lvm:vm-22248-disk-0,iothread=1,size=50G
```

## 2. 關閉虛擬機

```bash
qm shutdown 22248
```

要檢查：

```bash
qm status 22248
```

應該返回：

```text
status: stopped
```

## 3.識別磁碟設備

```bash
pvesm path local-lvm:vm-22248-disk-0
```

結果：

```text
/dev/pve/vm-22248-disk-0
```

## 4.檢查分區

```bash
fdisk -l /dev/pve/vm-22248-disk-0
```

在這種情況下：

```text
/dev/pve/vm-22248-disk-0p1   BIOS boot
/dev/pve/vm-22248-disk-0p2   EFI System
/dev/pve/vm-22248-disk-0p3   Linux swap
/dev/pve/vm-22248-disk-0p4   Linux filesystem
```

根分區是“p4”。

## 5.安裝kpartx

如果沒有安裝：

```bash
apt update
apt install kpartx
```

## 6. 映射磁碟分割區

```bash
kpartx -av /dev/pve/vm-22248-disk-0
```

將建立以下映射：

```text
/dev/mapper/pve-vm--22248--disk--0p1
/dev/mapper/pve-vm--22248--disk--0p2
/dev/mapper/pve-vm--22248--disk--0p3
/dev/mapper/pve-vm--22248--disk--0p4
```

## 7. 確認文件系統

```bash
blkid /dev/mapper/pve-vm--22248--disk--0p4
```

確認它出現：

```text
TYPE="ext4"
```

## 8.檢查並修復ext4

```bash
e2fsck -f -v /dev/mapper/pve-vm--22248--disk--0p4
```

`e2fsck` 將執行五個步驟：

```text
Pass 1: Checking inodes, blocks, and sizes
Pass 2: Checking directory structure
Pass 3: Checking directory connectivity
Pass 4: Checking reference counts
Pass 5: Checking group summary information
```

檢查返回碼：

```bash
echo $?
```

結果“0”表示檔案系統是一致的。

## 9. 刪除映射

完成“e2fsck”後：

```bash
kpartx -dv /dev/pve/vm-22248-disk-0
```

要檢查：

```bash
lsblk | grep 22248
```

## 

```bash
qm start 22248
```

要檢查：

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

切勿在已安裝的檔案系統上執行“e2fsck”。

修復前必須關閉虛擬機器電源。

在上面的範例中，根檔案系統是 50 GB `scsi0` 磁碟的 `p4` 分割區。
