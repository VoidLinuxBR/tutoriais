# Optimización de memoria y disco (PC antigua/poca RAM)

Esta guía muestra ajustes útiles para máquinas con poca RAM y disco duro lento (por ejemplo, PC más antiguas con 2 a 4 GB de RAM).

Incluye:

- zram (intercambio comprimido en RAM)
- archivo de intercambio
- prioridad de intercambio
- scheduler de disco BFQ
- ajustes de kernel (sysctl)
- Ajustes de latencia de E/S

---

# 1. Crear archivo de intercambio

Cree un archivo de intercambio de 6 GB.

```bash
fallocate -l 6G /swapfile
```

Permiso requerido:

```bash
chmod 600 /swapfile
```

Crear área de intercambio:

```bash
mkswap /swapfile
```

Activar:

```bash
swapon /swapfile
```

Agregar a `/etc/fstab`:

```text
/swapfile none swap defaults,pri=20 0 0
```

---

# 2. Intercambio de partición (si existe)

Agregue también a `/etc/fstab`:

```text
UUID=SEU-UUID none swap defaults,pri=10 0 0
```

UUID lejano:

```bash
blkid
```

---

# 3. Configurar zram (intercambio comprimido en RAM)

Módulo de carga:

```bash
modprobe zram
```

Elija compresor rápido:

```bash
echo lz4 > /sys/block/zram0/comp_algorithm
```

Tamaño establecido (ejemplo: 1 GB):

```bash
echo 1G > /sys/block/zram0/disksize
```

Crear intercambio:

```bash
mkswap /dev/zram0
```

Activar con mayor prioridad:

```bash
swapon -p 100 /dev/zram0
```

Para comprobar:

```bash
zramctl
```

o

```bash
cat /proc/swaps
```

---

# 4. Activar automáticamente zram en el arranque (runit)

Crear guión:

```
/etc/runit/core-services/10-zram.sh
```

Contenido:

```bash
#!/bin/sh

modprobe zram

echo lz4 > /sys/block/zram0/comp_algorithm
echo 1G > /sys/block/zram0/disksize

mkswap /dev/zram0
swapon -p 100 /dev/zram0
```

Permiso:

```bash
chmod +x /etc/runit/core-services/10-zram.sh
```

---

# 5. Ajustes de kernel (sysctl)

Editar:

```
/etc/sysctl.conf
```

Para agregar:

```text
vm.swappiness=15
vm.vfs_cache_pressure=200
vm.dirty_background_ratio=5
vm.dirty_ratio=10
```

Aplicar:

```bash
sysctl -p
```

---

# 6. Verificar páginas enormes transparentes

Ver estado:

```bash
cat /sys/kernel/mm/transparent_hugepage/enabled
```

Si no está configurado en "nunca", desactívelo.

Crear guión:

```
/etc/runit/core-services/05-thp.sh
```

Contenido:

```bash
#!/bin/sh
echo never > /sys/kernel/mm/transparent_hugepage/enabled
```

Permiso:

```bash
chmod +x /etc/runit/core-services/05-thp.sh
```

---

# 7. Mejorar el programador de discos

Ver el programador actual:

```bash
cat /sys/block/sda/queue/scheduler
```

Cambiar a BFQ:

```bash
echo bfq > /sys/block/sda/queue/scheduler
```

---

# 8. Ajustar los parámetros de BFQ

Ver parámetros:

```bash
ls /sys/block/sda/queue/iosched
```

Habilitar baja latencia:

```bash
echo 1 > /sys/block/sda/queue/iosched/low_latency
```

Reducir la espera entre solicitudes:

```bash
echo 0 > /sys/block/sda/queue/iosched/slice_idle
```

---

# 9. Aplicar BFQ automaticamente no boot

Crear guión:

```
/etc/runit/core-services/03-bfq.sh
```

Contenido:

```bash
#!/bin/sh

echo bfq > /sys/block/sda/queue/scheduler
echo 1 > /sys/block/sda/queue/iosched/low_latency
echo 0 > /sys/block/sda/queue/iosched/slice_idle
```

Permiso:

```bash
chmod +x /etc/runit/core-services/03-bfq.sh
```

---

# 10. Verifique el estado del sistema

Memoria:

```bash
free -h
```

Intercambio:

```bash
cat /proc/swaps
```

zram:

```bash
zramctl
```

---

# Resultado esperado

Orden de uso de la memoria:

```
RAM
 ↓
zram
 ↓
swapfile
 ↓
swap partition
```

Beneficios:

- menor uso de intercambio de disco
- menor latencia
- sistema más responsivo
- Mejor uso de RAM en máquinas con poca memoria

Especialmente útil para:

- PCs antigos
- disco duro lento
- sistemas con 2 a 4 GB de RAM
