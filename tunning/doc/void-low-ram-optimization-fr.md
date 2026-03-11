# Optimisation de la mémoire et du disque (ancien PC / RAM faible)

Ce guide présente des ajustements utiles pour les machines avec peu de RAM et un disque dur lent (par exemple, les PC plus anciens avec 2 à 4 Go de RAM).

Comprend :

- zram (swap compressé dans la RAM)
- fichier d'échange
- priorité d'échange
- Planificateur de disque BFQ
- paramètres du noyau (sysctl)
- Ajustements de latence d'E/S

---

# 1. Créer un fichier d'échange

Créez un fichier d'échange de 6 Go.

```bash
fallocate -l 6G /swapfile
```

Autorisation requise :

```bash
chmod 600 /swapfile
```

Créer une zone d'échange :

```bash
mkswap /swapfile
```

Activer:

```bash
swapon /swapfile
```

Ajouter à `/etc/fstab` :

```text
/swapfile none swap defaults,pri=20 0 0
```

---

# 2. Échange de partition (si existant)

Ajoutez également à `/etc/fstab` :

```text
UUID=SEU-UUID none swap defaults,pri=10 0 0
```

UUID lointain :

```bash
blkid
```

---

# 3. Configurez zram (swap compressé dans la RAM)

Module de chargement :

```bash
modprobe zram
```

Choisissez un compresseur rapide :

```bash
echo lz4 > /sys/block/zram0/comp_algorithm
```

Taille définie (exemple : 1 Go) :

```bash
echo 1G > /sys/block/zram0/disksize
```

Créer un échange :

```bash
mkswap /dev/zram0
```

Activer avec une priorité plus élevée :

```bash
swapon -p 100 /dev/zram0
```

Pour vérifier :

```bash
zramctl
```

ou

```bash
cat /proc/swaps
```

---

# 4. Activer automatiquement zram au démarrage (runit)

Créer un script :

```
/etc/runit/core-services/10-zram.sh
```

Contenu:

```bash
#!/bin/sh

modprobe zram

echo lz4 > /sys/block/zram0/comp_algorithm
echo 1G > /sys/block/zram0/disksize

mkswap /dev/zram0
swapon -p 100 /dev/zram0
```

Autorisation:

```bash
chmod +x /etc/runit/core-services/10-zram.sh
```

---

# 5. Paramètres du noyau (sysctl)

Modifier:

```
/etc/sysctl.conf
```

Pour ajouter :

```text
vm.swappiness=15
vm.vfs_cache_pressure=200
vm.dirty_background_ratio=5
vm.dirty_ratio=10
```

Appliquer:

```bash
sysctl -p
```

---

# 6. Vérifier les énormes pages transparentes

Afficher l'état :

```bash
cat /sys/kernel/mm/transparent_hugepage/enabled
```

S'il n'est pas défini sur « jamais », désactivez-le.

Créer un script :

```
/etc/runit/core-services/05-thp.sh
```

Contenu:

```bash
#!/bin/sh
echo never > /sys/kernel/mm/transparent_hugepage/enabled
```

Autorisation:

```bash
chmod +x /etc/runit/core-services/05-thp.sh
```

---

# 7. Améliorer le planificateur de disque

Afficher le planificateur actuel :

```bash
cat /sys/block/sda/queue/scheduler
```

Passer à BFQ :

```bash
echo bfq > /sys/block/sda/queue/scheduler
```

---

# 8. Ajustez les paramètres BFQ

Afficher les paramètres :

```bash
ls /sys/block/sda/queue/iosched
```

Activer la faible latence :

```bash
echo 1 > /sys/block/sda/queue/iosched/low_latency
```

Réduisez l’attente entre les demandes :

```bash
echo 0 > /sys/block/sda/queue/iosched/slice_idle
```

---

# 9. Appliquer BFQ automatiquement sans démarrage

Créer un script :

```
/etc/runit/core-services/03-bfq.sh
```

Contenu:

```bash
#!/bin/sh

echo bfq > /sys/block/sda/queue/scheduler
echo 1 > /sys/block/sda/queue/iosched/low_latency
echo 0 > /sys/block/sda/queue/iosched/slice_idle
```

Autorisation:

```bash
chmod +x /etc/runit/core-services/03-bfq.sh
```

---

# 10. Vérifiez l'état du système

Mémoire:

```bash
free -h
```

Échanger:

```bash
cat /proc/swaps
```

zram :

```bash
zramctl
```

---

# Résultat attendu

Ordre d'utilisation de la mémoire :

```
RAM
 ↓
zram
 ↓
swapfile
 ↓
swap partition
```

Avantages:

- utilisation réduite du swap de disque
- latence inférieure
- système plus réactif
- Meilleure utilisation de la RAM sur les machines à faible mémoire

Particulièrement utile pour :

- vieux PC
- Lento du disque dur
- systèmes avec 2 à 4 Go de RAM
