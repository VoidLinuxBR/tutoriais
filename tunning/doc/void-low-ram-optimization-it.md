# Ottimizzazione della memoria e del disco (vecchio PC/scarsa RAM)

Questa guida mostra utili modifiche per macchine con poca RAM e HDD lento (ad esempio PC più vecchi con 2-4 GB di RAM).

Include:

- zram (swap compresso nella RAM)
- file di scambio
- scambiare la priorità
- Pianificatore del disco BFQ
- impostazioni del kernel (sysctl)
- Modifiche alla latenza I/O

---

# 1. Crea il file di scambio

Crea un file di scambio da 6 GB.

```bash
fallocate -l 6G /swapfile
```

Autorizzazione richiesta:

```bash
chmod 600 /swapfile
```

Crea area di scambio:

```bash
mkswap /swapfile
```

Attivare:

```bash
swapon /swapfile
```

Aggiungi a `/etc/fstab`:

```text
/swapfile none swap defaults,pri=20 0 0
```

---

# 2. Scambio di partizione (se esiste)

Aggiungi anche a `/etc/fstab`:

```text
UUID=SEU-UUID none swap defaults,pri=10 0 0
```

UUID lontano:

```bash
blkid
```

---

# 3. Configura zram (swap compresso nella RAM)

Carica modulo:

```bash
modprobe zram
```

Scegli il compressore veloce:

```bash
echo lz4 > /sys/block/zram0/comp_algorithm
```

Imposta dimensione (esempio: 1 GB):

```bash
echo 1G > /sys/block/zram0/disksize
```

Crea scambio:

```bash
mkswap /dev/zram0
```

Attiva con priorità più alta:

```bash
swapon -p 100 /dev/zram0
```

Per verificare:

```bash
zramctl
```

O

```bash
cat /proc/swaps
```

---

# 4. Attiva automaticamente zram all'avvio (runit)

Crea script:

```
/etc/runit/core-services/10-zram.sh
```

Contenuto:

```bash
#!/bin/sh

modprobe zram

echo lz4 > /sys/block/zram0/comp_algorithm
echo 1G > /sys/block/zram0/disksize

mkswap /dev/zram0
swapon -p 100 /dev/zram0
```

Autorizzazione:

```bash
chmod +x /etc/runit/core-services/10-zram.sh
```

---

# 5. Impostazioni del kernel (sysctl)

Modificare:

```
/etc/sysctl.conf
```

Per aggiungere:

```text
vm.swappiness=15
vm.vfs_cache_pressure=200
vm.dirty_background_ratio=5
vm.dirty_ratio=10
```

Fare domanda a:

```bash
sysctl -p
```

---

# 6. Verifica pagine enormi trasparenti

Visualizza stato:

```bash
cat /sys/kernel/mm/transparent_hugepage/enabled
```

Se non è impostato su "mai", disabilitalo.

Crea script:

```
/etc/runit/core-services/05-thp.sh
```

Contenuto:

```bash
#!/bin/sh
echo never > /sys/kernel/mm/transparent_hugepage/enabled
```

Autorizzazione:

```bash
chmod +x /etc/runit/core-services/05-thp.sh
```

---

# 7. Migliora la pianificazione del disco

Visualizza la pianificazione corrente:

```bash
cat /sys/block/sda/queue/scheduler
```

Passa a BFQ:

```bash
echo bfq > /sys/block/sda/queue/scheduler
```

---

# 8. Regolare i parametri BFQ

Visualizza parametri:

```bash
ls /sys/block/sda/queue/iosched
```

Abilita bassa latenza:

```bash
echo 1 > /sys/block/sda/queue/iosched/low_latency
```

Riduci l'attesa tra le richieste:

```bash
echo 0 > /sys/block/sda/queue/iosched/slice_idle
```

---

# 9. Applica BFQ automaticamente senza avvio

Crea script:

```
/etc/runit/core-services/03-bfq.sh
```

Contenuto:

```bash
#!/bin/sh

echo bfq > /sys/block/sda/queue/scheduler
echo 1 > /sys/block/sda/queue/iosched/low_latency
echo 0 > /sys/block/sda/queue/iosched/slice_idle
```

Autorizzazione:

```bash
chmod +x /etc/runit/core-services/03-bfq.sh
```

---

# 10. Controllare lo stato del sistema

Memoria:

```bash
free -h
```

Scambio:

```bash
cat /proc/swaps
```

zram:

```bash
zramctl
```

---

# Risultato atteso

Ordine di utilizzo della memoria:

```
RAM
 ↓
zram
 ↓
swapfile
 ↓
swap partition
```

Vantaggi:

- minore utilizzo dello scambio del disco
- latenza inferiore
- sistema più reattivo
- Migliore utilizzo della RAM su macchine con poca memoria

Particolarmente utile per:

- vecchi PC
- HDD lento
- sistemi con 2–4 GB di RAM
