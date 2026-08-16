# Ripara il filesystem ext4 di una VM in Proxmox

Procedura per controllare e riparare il filesystem ext4 della partizione root di una VM Proxmox.

## 1. Identificare la VM

Listar come VM:

```bash
qm list
```

Visualizza la configurazione della VM:

```bash
qm config 22248
```

Identificare il disco di root. In questo caso:

```text
scsi0: local-lvm:vm-22248-disk-0,iothread=1,size=50G
```

## 2. Arrestare la VM

```bash
qm shutdown 22248
```

Per verificare:

```bash
qm status 22248
```

Dovrebbe restituire:

```text
status: stopped
```

## 3. Identificare il dispositivo disco

```bash
pvesm path local-lvm:vm-22248-disk-0
```

Risultato:

```text
/dev/pve/vm-22248-disk-0
```

## 4. Controllare le partizioni

```bash
fdisk -l /dev/pve/vm-22248-disk-0
```

In questo caso:

```text
/dev/pve/vm-22248-disk-0p1   BIOS boot
/dev/pve/vm-22248-disk-0p2   EFI System
/dev/pve/vm-22248-disk-0p3   Linux swap
/dev/pve/vm-22248-disk-0p4   Linux filesystem
```

La partizione root era "p4".

## 5. Installa kpartx

Se non installato:

```bash
apt update
apt install kpartx
```

## 6. Mappare le partizioni del disco

```bash
kpartx -av /dev/pve/vm-22248-disk-0
```

Verranno create le seguenti mappature:

```text
/dev/mapper/pve-vm--22248--disk--0p1
/dev/mapper/pve-vm--22248--disk--0p2
/dev/mapper/pve-vm--22248--disk--0p3
/dev/mapper/pve-vm--22248--disk--0p4
```

## 7. Confermare il file system

```bash
blkid /dev/mapper/pve-vm--22248--disk--0p4
```

Conferma che appaia:

```text
TYPE="ext4"
```

## 8. Controllare e riparare ext4

```bash
e2fsck -f -v /dev/mapper/pve-vm--22248--disk--0p4
```

`e2fsck` eseguirà i cinque passaggi:

```text
Pass 1: Checking inodes, blocks, and sizes
Pass 2: Checking directory structure
Pass 3: Checking directory connectivity
Pass 4: Checking reference counts
Pass 5: Checking group summary information
```

Controlla il codice di reso:

```bash
echo $?
```

Il risultato "0" indica che il filesystem è coerente.

## 9. Rimuovere le mappature

Dopo aver terminato "e2fsck":

```bash
kpartx -dv /dev/pve/vm-22248-disk-0
```

Per verificare:

```bash
lsblk | grep 22248
```

## 10. Avviare la VM

```bash
qm start 22248
```

Per verificare:

```bash
qm status 22248
```

## RIEPILOGO

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

Non eseguire mai `e2fsck` su un filesystem montato.

La VM deve essere spenta prima della riparazione.

Nell'esempio sopra, il filesystem root era la partizione "p4" del disco "scsi0" da 50 GB.
