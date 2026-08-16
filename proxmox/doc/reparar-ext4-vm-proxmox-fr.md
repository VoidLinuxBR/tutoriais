# Réparer le système de fichiers ext4 d'une VM dans Proxmox

Procédure pour vérifier et réparer le système de fichiers ext4 de la partition racine d'une VM Proxmox.

## 1. Identifiez la VM

Lister en tant que VM :

```bash
qm list
```

Afficher la configuration de la VM :

```bash
qm config 22248
```

Identifiez le disque racine. Dans ce cas:

```text
scsi0: local-lvm:vm-22248-disk-0,iothread=1,size=50G
```

## 2. Arrêtez la VM

```bash
qm shutdown 22248
```

Pour vérifier :

```bash
qm status 22248
```

Devrait revenir :

```text
status: stopped
```

## 3. Identifiez le périphérique de disque

```bash
pvesm path local-lvm:vm-22248-disk-0
```

Résultat:

```text
/dev/pve/vm-22248-disk-0
```

## 4. Vérifiez les partitions

```bash
fdisk -l /dev/pve/vm-22248-disk-0
```

Dans ce cas:

```text
/dev/pve/vm-22248-disk-0p1   BIOS boot
/dev/pve/vm-22248-disk-0p2   EFI System
/dev/pve/vm-22248-disk-0p3   Linux swap
/dev/pve/vm-22248-disk-0p4   Linux filesystem
```

La partition racine était « p4 ».

## 5. Installez kpartx

S'il n'est pas installé :

```bash
apt update
apt install kpartx
```

## 6. Mapper les partitions de disque

```bash
kpartx -av /dev/pve/vm-22248-disk-0
```

Les mappages suivants seront créés :

```text
/dev/mapper/pve-vm--22248--disk--0p1
/dev/mapper/pve-vm--22248--disk--0p2
/dev/mapper/pve-vm--22248--disk--0p3
/dev/mapper/pve-vm--22248--disk--0p4
```

## 7. Confirmez le système de fichiers

```bash
blkid /dev/mapper/pve-vm--22248--disk--0p4
```

Confirmez qu'il apparaît :

```text
TYPE="ext4"
```

## 8. Vérifiez et réparez ext4

```bash
e2fsck -f -v /dev/mapper/pve-vm--22248--disk--0p4
```

`e2fsck` effectuera les cinq étapes :

```text
Pass 1: Checking inodes, blocks, and sizes
Pass 2: Checking directory structure
Pass 3: Checking directory connectivity
Pass 4: Checking reference counts
Pass 5: Checking group summary information
```

Vérifiez le code retour :

```bash
echo $?
```

Le résultat « 0 » indique que le système de fichiers est cohérent.

## 9. Supprimer les mappages

Après avoir terminé `e2fsck` :

```bash
kpartx -dv /dev/pve/vm-22248-disk-0
```

Pour vérifier :

```bash
lsblk | grep 22248
```

## 10. Démarrez la machine virtuelle

```bash
qm start 22248
```

Pour vérifier :

```bash
qm status 22248
```

## RÉSUMÉ

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

N'exécutez jamais `e2fsck` sur un système de fichiers monté.

La VM doit être éteinte avant la réparation.

Dans l'exemple ci-dessus, le système de fichiers racine était la partition « p4 » du disque « scsi0 » de 50 Go.
