# Otimização de Memória e Disco (PC antigo / pouca RAM)

Este guia mostra ajustes úteis para máquinas com pouca RAM e HDD lento (ex: PCs antigos com 2–4GB de RAM).

Inclui:

- zram (swap comprimido na RAM)
- swapfile
- prioridade de swap
- scheduler de disco BFQ
- ajustes de kernel (sysctl)
- tweaks de latência de I/O

---

# 1. Criar swapfile

Criar arquivo de swap de 6GB.

```bash
fallocate -l 6G /swapfile
```

Permissão obrigatória:

```bash
chmod 600 /swapfile
```

Criar área swap:

```bash
mkswap /swapfile
```

Ativar:

```bash
swapon /swapfile
```

Adicionar ao `/etc/fstab`:

```text
/swapfile none swap defaults,pri=20 0 0
```

---

# 2. Swap de partição (se existir)

Adicionar também no `/etc/fstab`:

```text
UUID=SEU-UUID none swap defaults,pri=10 0 0
```

Ver UUID:

```bash
blkid
```

---

# 3. Configurar zram (swap comprimido na RAM)

Carregar módulo:

```bash
modprobe zram
```

Escolher compressor rápido:

```bash
echo lz4 > /sys/block/zram0/comp_algorithm
```

Definir tamanho (exemplo: 1GB):

```bash
echo 1G > /sys/block/zram0/disksize
```

Criar swap:

```bash
mkswap /dev/zram0
```

Ativar com prioridade maior:

```bash
swapon -p 100 /dev/zram0
```

Verificar:

```bash
zramctl
```

ou

```bash
cat /proc/swaps
```

---

# 4. Ativar zram automaticamente no boot (runit)

Criar script:

```
/etc/runit/core-services/10-zram.sh
```

Conteúdo:

```bash
#!/bin/sh

modprobe zram

echo lz4 > /sys/block/zram0/comp_algorithm
echo 1G > /sys/block/zram0/disksize

mkswap /dev/zram0
swapon -p 100 /dev/zram0
```

Permissão:

```bash
chmod +x /etc/runit/core-services/10-zram.sh
```

---

# 5. Ajustes de kernel (sysctl)

Editar:

```
/etc/sysctl.conf
```

Adicionar:

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

# 6. Verificar Transparent Huge Pages

Ver estado:

```bash
cat /sys/kernel/mm/transparent_hugepage/enabled
```

Se não estiver em `never`, desativar.

Criar script:

```
/etc/runit/core-services/05-thp.sh
```

Conteúdo:

```bash
#!/bin/sh
echo never > /sys/kernel/mm/transparent_hugepage/enabled
```

Permissão:

```bash
chmod +x /etc/runit/core-services/05-thp.sh
```

---

# 7. Melhorar scheduler de disco

Ver scheduler atual:

```bash
cat /sys/block/sda/queue/scheduler
```

Trocar para BFQ:

```bash
echo bfq > /sys/block/sda/queue/scheduler
```

---

# 8. Ajustar parâmetros do BFQ

Ver parâmetros:

```bash
ls /sys/block/sda/queue/iosched
```

Ativar baixa latência:

```bash
echo 1 > /sys/block/sda/queue/iosched/low_latency
```

Reduzir espera entre requisições:

```bash
echo 0 > /sys/block/sda/queue/iosched/slice_idle
```

---

# 9. Aplicar BFQ automaticamente no boot

Criar script:

```
/etc/runit/core-services/03-bfq.sh
```

Conteúdo:

```bash
#!/bin/sh

echo bfq > /sys/block/sda/queue/scheduler
echo 1 > /sys/block/sda/queue/iosched/low_latency
echo 0 > /sys/block/sda/queue/iosched/slice_idle
```

Permissão:

```bash
chmod +x /etc/runit/core-services/03-bfq.sh
```

---

# 10. Verificar estado do sistema

Memória:

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

# Resultado esperado

Ordem de uso da memória:

```
RAM
 ↓
zram
 ↓
swapfile
 ↓
swap partition
```

Benefícios:

- menor uso de swap em disco
- menor latência
- sistema mais responsivo
- melhor uso de RAM em máquinas com pouca memória

Especialmente útil para:

- PCs antigos
- HDD lento
- sistemas com 2–4GB de RAM
