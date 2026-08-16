# Proxmox で VM の ext4 ファイルシステムを修復する

Proxmox VM のルート パーティションの ext4 ファイルシステムをチェックして修復する手順。

## 1.VMを特定する

VM としての Listar:

```bash
qm list
```

VM 構成を表示します。

```bash
qm config 22248
```

ルートディスクを特定します。この場合：

```text
scsi0: local-lvm:vm-22248-disk-0,iothread=1,size=50G
```

## 2. VM をシャットダウンします。

```bash
qm shutdown 22248
```

確認するには:

```bash
qm status 22248
```

返されるべきもの:

```text
status: stopped
```

## 3. ディスクデバイスを特定する

```bash
pvesm path local-lvm:vm-22248-disk-0
```

結果：

```text
/dev/pve/vm-22248-disk-0
```

## 4. パーティションを確認する

```bash
fdisk -l /dev/pve/vm-22248-disk-0
```

この場合：

```text
/dev/pve/vm-22248-disk-0p1   BIOS boot
/dev/pve/vm-22248-disk-0p2   EFI System
/dev/pve/vm-22248-disk-0p3   Linux swap
/dev/pve/vm-22248-disk-0p4   Linux filesystem
```

ルートパーティションは「p4」でした。

## 5.kpartxをインストールする

インストールされていない場合:

```bash
apt update
apt install kpartx
```

## 6. ディスクパーティションのマッピング

```bash
kpartx -av /dev/pve/vm-22248-disk-0
```

次のマッピングが作成されます。

```text
/dev/mapper/pve-vm--22248--disk--0p1
/dev/mapper/pve-vm--22248--disk--0p2
/dev/mapper/pve-vm--22248--disk--0p3
/dev/mapper/pve-vm--22248--disk--0p4
```

## 7. ファイルシステムの確認

```bash
blkid /dev/mapper/pve-vm--22248--disk--0p4
```

表示されることを確認します。

```text
TYPE="ext4"
```

## 8. ext4 の確認と修復

```bash
e2fsck -f -v /dev/mapper/pve-vm--22248--disk--0p4
```

「e2fsck」は次の 5 つのステップを実行します。

```text
Pass 1: Checking inodes, blocks, and sizes
Pass 2: Checking directory structure
Pass 3: Checking directory connectivity
Pass 4: Checking reference counts
Pass 5: Checking group summary information
```

戻りコードを確認してください:

```bash
echo $?
```

結果「0」は、ファイルシステムが一貫していることを示します。

## 9. マッピングを削除する

`e2fsck` の終了後:

```bash
kpartx -dv /dev/pve/vm-22248-disk-0
```

確認するには:

```bash
lsblk | grep 22248
```

## 10.VMを起動します

```bash
qm start 22248
```

確認するには:

```bash
qm status 22248
```

## まとめ

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

## 重要

マウントされたファイルシステム上では決して「e2fsck」を実行しないでください。

修復する前に VM の電源をオフにする必要があります。

上の例では、ルート ファイルシステムは 50 GB の「scsi0」ディスクの「p4」パーティションでした。
