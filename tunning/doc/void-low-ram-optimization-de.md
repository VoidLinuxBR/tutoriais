# Speicher- und Festplattenoptimierung (alter PC / wenig RAM)

Diese Anleitung zeigt nützliche Optimierungen für Maschinen mit wenig RAM und langsamer Festplatte (z. B. ältere PCs mit 2–4 GB RAM).

Beinhaltet:

- zram (komprimierter Swap im RAM)
- Auslagerungsdatei
- Priorität tauschen
- BFQ-Festplattenplaner
- Kernel-Einstellungen (sysctl)
- Optimierungen der E/A-Latenz

---

# 1. Auslagerungsdatei erstellen

Erstellen Sie eine 6-GB-Auslagerungsdatei.

```bash
fallocate -l 6G /swapfile
```

Erforderliche Erlaubnis:

```bash
chmod 600 /swapfile
```

Swap-Bereich erstellen:

```bash
mkswap /swapfile
```

Aktivieren:

```bash
swapon /swapfile
```

Zu „/etc/fstab“ hinzufügen:

```text
/swapfile none swap defaults,pri=20 0 0
```

---

# 2. Partitionsaustausch (falls vorhanden)

Fügen Sie außerdem zu „/etc/fstab“ hinzu:

```text
UUID=SEU-UUID none swap defaults,pri=10 0 0
```

Far UUID:

```bash
blkid
```

---

# 3. Konfigurieren Sie zram (komprimierter Swap im RAM)

Modul laden:

```bash
modprobe zram
```

Wählen Sie einen schnellen Kompressor:

```bash
echo lz4 > /sys/block/zram0/comp_algorithm
```

Größe festlegen (Beispiel: 1 GB):

```bash
echo 1G > /sys/block/zram0/disksize
```

Tausch erstellen:

```bash
mkswap /dev/zram0
```

Mit höherer Priorität aktivieren:

```bash
swapon -p 100 /dev/zram0
```

Zur Überprüfung:

```bash
zramctl
```

oder

```bash
cat /proc/swaps
```

---

# 4. Zram beim Booten automatisch aktivieren (runit)

Skript erstellen:

```
/etc/runit/core-services/10-zram.sh
```

Inhalt:

```bash
#!/bin/sh

modprobe zram

echo lz4 > /sys/block/zram0/comp_algorithm
echo 1G > /sys/block/zram0/disksize

mkswap /dev/zram0
swapon -p 100 /dev/zram0
```

Erlaubnis:

```bash
chmod +x /etc/runit/core-services/10-zram.sh
```

---

# 5. Kernel-Einstellungen (sysctl)

Bearbeiten:

```
/etc/sysctl.conf
```

Hinzufügen:

```text
vm.swappiness=15
vm.vfs_cache_pressure=200
vm.dirty_background_ratio=5
vm.dirty_ratio=10
```

Anwenden:

```bash
sysctl -p
```

---

# 6. Überprüfen Sie transparente riesige Seiten

Status anzeigen:

```bash
cat /sys/kernel/mm/transparent_hugepage/enabled
```

Wenn es nicht auf „Nie“ eingestellt ist, deaktivieren Sie es.

Skript erstellen:

```
/etc/runit/core-services/05-thp.sh
```

Inhalt:

```bash
#!/bin/sh
echo never > /sys/kernel/mm/transparent_hugepage/enabled
```

Erlaubnis:

```bash
chmod +x /etc/runit/core-services/05-thp.sh
```

---

# 7. Verbessern Sie den Festplattenplaner

Aktuellen Planer anzeigen:

```bash
cat /sys/block/sda/queue/scheduler
```

Zu BFQ wechseln:

```bash
echo bfq > /sys/block/sda/queue/scheduler
```

---

# 8. Passen Sie die BFQ-Parameter an

Parameter anzeigen:

```bash
ls /sys/block/sda/queue/iosched
```

Niedrige Latenz aktivieren:

```bash
echo 1 > /sys/block/sda/queue/iosched/low_latency
```

Reduzieren Sie die Wartezeit zwischen Anfragen:

```bash
echo 0 > /sys/block/sda/queue/iosched/slice_idle
```

---

# 9. Wenden Sie BFQ automatisch ohne Booten an

Skript erstellen:

```
/etc/runit/core-services/03-bfq.sh
```

Inhalt:

```bash
#!/bin/sh

echo bfq > /sys/block/sda/queue/scheduler
echo 1 > /sys/block/sda/queue/iosched/low_latency
echo 0 > /sys/block/sda/queue/iosched/slice_idle
```

Erlaubnis:

```bash
chmod +x /etc/runit/core-services/03-bfq.sh
```

---

# 10. Überprüfen Sie den Systemstatus

Erinnerung:

```bash
free -h
```

Tauschen:

```bash
cat /proc/swaps
```

zram:

```bash
zramctl
```

---

# Erwartetes Ergebnis

Reihenfolge der Speichernutzung:

```
RAM
 ↓
zram
 ↓
swapfile
 ↓
swap partition
```

Vorteile:

- geringere Festplatten-Swap-Nutzung
- geringere Latenz
- reaktionsfähigeres System
- Bessere RAM-Nutzung auf Maschinen mit wenig Arbeitsspeicher

Besonders nützlich für:

- alte PCs
- HDD-Lento
- Systeme mit 2–4 GB RAM
