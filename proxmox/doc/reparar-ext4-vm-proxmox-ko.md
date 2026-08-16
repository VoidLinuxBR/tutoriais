# Proxmox에서 VM의 ext4 파일 시스템 복구

Proxmox VM 루트 파티션의 ext4 파일 시스템을 확인하고 복구하는 절차입니다.

## 1. VM 식별

VM으로 리스타:

```bash
qm list
```

VM 구성 보기:

```bash
qm config 22248
```

루트 디스크를 식별합니다. 이 경우:

```text
scsi0: local-lvm:vm-22248-disk-0,iothread=1,size=50G
```

## 2. VM 종료

```bash
qm shutdown 22248
```

확인하려면:

```bash
qm status 22248
```

다음을 반환해야 합니다.

```text
status: stopped
```

## 3. 디스크 장치 식별

```bash
pvesm path local-lvm:vm-22248-disk-0
```

결과:

```text
/dev/pve/vm-22248-disk-0
```

## 4. 파티션 확인

```bash
fdisk -l /dev/pve/vm-22248-disk-0
```

이 경우:

```text
/dev/pve/vm-22248-disk-0p1   BIOS boot
/dev/pve/vm-22248-disk-0p2   EFI System
/dev/pve/vm-22248-disk-0p3   Linux swap
/dev/pve/vm-22248-disk-0p4   Linux filesystem
```

루트 파티션은 'p4'였습니다.

## 5. kpartx 설치

설치되지 않은 경우:

```bash
apt update
apt install kpartx
```

## 6. 디스크 파티션 매핑

```bash
kpartx -av /dev/pve/vm-22248-disk-0
```

다음 매핑이 생성됩니다.

```text
/dev/mapper/pve-vm--22248--disk--0p1
/dev/mapper/pve-vm--22248--disk--0p2
/dev/mapper/pve-vm--22248--disk--0p3
/dev/mapper/pve-vm--22248--disk--0p4
```

## 7. 파일 시스템 확인

```bash
blkid /dev/mapper/pve-vm--22248--disk--0p4
```

나타나는지 확인하십시오.

```text
TYPE="ext4"
```

## 8. ext4 점검 및 수리

```bash
e2fsck -f -v /dev/mapper/pve-vm--22248--disk--0p4
```

`e2fsck`는 5가지 단계를 수행합니다:

```text
Pass 1: Checking inodes, blocks, and sizes
Pass 2: Checking directory structure
Pass 3: Checking directory connectivity
Pass 4: Checking reference counts
Pass 5: Checking group summary information
```

반환 코드 확인:

```bash
echo $?
```

결과 '0'은 파일 시스템이 일관적임을 나타냅니다.

## 9. 매핑 제거

`e2fsck`를 마친 후:

```bash
kpartx -dv /dev/pve/vm-22248-disk-0
```

확인하려면:

```bash
lsblk | grep 22248
```

## 10. VM 시작

```bash
qm start 22248
```

확인하려면:

```bash
qm status 22248
```

## 요약

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

## 중요한

마운트된 파일 시스템에서는 `e2fsck`를 실행하지 마십시오.

수리하기 전에 VM의 전원을 꺼야 합니다.

위의 예에서 루트 파일 시스템은 50GB `scsi0` 디스크의 `p4` 파티션이었습니다.
