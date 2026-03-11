# Memory and Disk Optimization (old PC / low RAM)

This guide shows useful tweaks for machines with low RAM and slow HDD (e.g. older PCs with 2–4GB RAM).

Includes:

- zram (compressed swap in RAM)
- swapfile
- swap priority
- BFQ disk scheduler
- kernel settings (sysctl)
- I/O latency tweaks

---

# 1. Create swapfile

Create 6GB swap file.

```bash
fallocate -l 6G /swapfile
```

Required permission:

```bash
chmod 600 /swapfile
```

Create swap area:

```bash
mkswap /swapfile
```

Activate:

```bash
swapon /swapfile
```

Add to `/etc/fstab`:

```text
/swapfile none swap defaults,pri=20 0 0
```

---

# 2. Partition swap (if exists)

Also add to `/etc/fstab`:

```text
UUID=SEU-UUID none swap defaults,pri=10 0 0
```

Far UUID:

```bash
blkid
```

---

# 3. Configure zram (compressed swap in RAM)

Load module:

```bash
modprobe zram
```

Choose fast compressor:

```bash
echo lz4 > /sys/block/zram0/comp_algorithm
```

Set size (example: 1GB):

```bash
echo 1G > /sys/block/zram0/disksize
```

Create swap:

```bash
mkswap /dev/zram0
```

Activate with higher priority:

```bash
swapon -p 100 /dev/zram0
```

To check:

```bash
zramctl
```

or

```bash
cat /proc/swaps
```

---

# 4. Automatically activate zram at boot (runit)

Create script:

```
/etc/runit/core-services/10-zram.sh
```

Content:

```bash
#!/bin/sh

modprobe zram

echo lz4 > /sys/block/zram0/comp_algorithm
echo 1G > /sys/block/zram0/disksize

mkswap /dev/zram0
swapon -p 100 /dev/zram0
```

Permission:

```bash
chmod +x /etc/runit/core-services/10-zram.sh
```

---

# 5. Kernel settings (sysctl)

Edit:

```
/etc/sysctl.conf
```

To add:

```text
vm.swappiness=15
vm.vfs_cache_pressure=200
vm.dirty_background_ratio=5
vm.dirty_ratio=10
```

Apply:

```bash
sysctl -p
```

---

# 6. Verificar Transparent Huge Pages

View status:

```bash
cat /sys/kernel/mm/transparent_hugepage/enabled
```

If it is not set to `never`, disable it.

Create script:

```
/etc/runit/core-services/05-thp.sh
```

Content:

```bash
#!/bin/sh
echo never > /sys/kernel/mm/transparent_hugepage/enabled
```

Permission:

```bash
chmod +x /etc/runit/core-services/05-thp.sh
```

---

# 7. Improve disk scheduler

View current scheduler:

```bash
cat /sys/block/sda/queue/scheduler
```

Swap to BFQ:

```bash
echo bfq > /sys/block/sda/queue/scheduler
```

---

# 8. Adjust BFQ parameters

View parameters:

```bash
ls /sys/block/sda/queue/iosched
```

Enable Low Latency:

```bash
echo 1 > /sys/block/sda/queue/iosched/low_latency
```

Reduce waiting between requests:

```bash
echo 0 > /sys/block/sda/queue/iosched/slice_idle
```

---

# 9. Apply BFQ automatically without boot

Create script:

```
/etc/runit/core-services/03-bfq.sh
```

Content:

```bash
#!/bin/sh

echo bfq > /sys/block/sda/queue/scheduler
echo 1 > /sys/block/sda/queue/iosched/low_latency
echo 0 > /sys/block/sda/queue/iosched/slice_idle
```

Permission:

```bash
chmod +x /etc/runit/core-services/03-bfq.sh
```

---

# 10. Check system status

Memory:

```bash
free -h
```

Swap:

```bash
cat /proc/swaps
```

zram:

```bash
zramctl
```

---

# Expected result

Memory usage order:

```
RAM
 ↓
zram
 ↓
swapfile
 ↓
swap partition
```

Benefits:

- lower disk swap usage
- lower latency
- more responsive system
- Better RAM usage on low memory machines

Especially useful for:

- old PCs
- HDD lento
- systems with 2–4GB of RAM
