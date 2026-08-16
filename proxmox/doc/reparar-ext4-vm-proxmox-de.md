# Ext4-Dateisystem einer VM in Proxmox reparieren

Vorgehensweise zum Überprüfen und Reparieren des ext4-Dateisystems der Root-Partition einer Proxmox-VM.

## 1. VM identifizieren

Als VMs auflisten:

```bash
qm list
```

VM-Konfiguration anzeigen:

```bash
qm config 22248
```

Identifizieren Sie die Root-Festplatte. In diesem Fall:

```text
scsi0: local-lvm:vm-22248-disk-0,iothread=1,size=50G
```

## 2. Fahren Sie die VM herunter

```bash
qm shutdown 22248
```

Zur Überprüfung:

```bash
qm status 22248
```

Sollte zurückkommen:

```text
status: stopped
```

## 3. Identifizieren Sie das Festplattengerät

```bash
pvesm path local-lvm:vm-22248-disk-0
```

Ergebnis:

```text
/dev/pve/vm-22248-disk-0
```

## 4. Überprüfen Sie die Partitionen

```bash
fdisk -l /dev/pve/vm-22248-disk-0
```

In diesem Fall:

```text
/dev/pve/vm-22248-disk-0p1   BIOS boot
/dev/pve/vm-22248-disk-0p2   EFI System
/dev/pve/vm-22248-disk-0p3   Linux swap
/dev/pve/vm-22248-disk-0p4   Linux filesystem
```

Die Root-Partition war „p4“.

## 5. Installieren Sie kpartx

Falls nicht installiert:

```bash
apt update
apt install kpartx
```

## 6. Ordnen Sie Festplattenpartitionen zu

```bash
kpartx -av /dev/pve/vm-22248-disk-0
```

Die folgenden Zuordnungen werden erstellt:

```text
/dev/mapper/pve-vm--22248--disk--0p1
/dev/mapper/pve-vm--22248--disk--0p2
/dev/mapper/pve-vm--22248--disk--0p3
/dev/mapper/pve-vm--22248--disk--0p4
```

## 7. Bestätigen Sie das Dateisystem

```bash
blkid /dev/mapper/pve-vm--22248--disk--0p4
```

Bestätigen Sie, dass Folgendes angezeigt wird:

```text
TYPE="ext4"
```

## 8. Überprüfen und reparieren Sie ext4

```bash
e2fsck -f -v /dev/mapper/pve-vm--22248--disk--0p4
```

„e2fsck“ führt die fünf Schritte aus:

```text
Pass 1: Checking inodes, blocks, and sizes
Pass 2: Checking directory structure
Pass 3: Checking directory connectivity
Pass 4: Checking reference counts
Pass 5: Checking group summary information
```

Rückgabecode prüfen:

```bash
echo $?
```

Das Ergebnis „0“ zeigt an, dass das Dateisystem konsistent ist.

## 9. Zuordnungen entfernen

Nach Abschluss von „e2fsck“:

```bash
kpartx -dv /dev/pve/vm-22248-disk-0
```

Zur Überprüfung:

```bash
lsblk | grep 22248
```

## 10. Starten Sie die VM

```bash
qm start 22248
```

Zur Überprüfung:

```bash
qm status 22248
```

## ZUSAMMENFASSUNG

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

## Wichtig

Führen Sie „e2fsck“ niemals auf einem gemounteten Dateisystem aus.

Die VM muss vor der Reparatur ausgeschaltet werden.

Im obigen Beispiel war das Root-Dateisystem die „p4“-Partition der 50-GB-Festplatte „scsi0“.
