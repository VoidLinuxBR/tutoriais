# 内存和磁盘优化（旧电脑/低 RAM）

本指南展示了对 RAM 较低且 HDD 较慢的机器（例如具有 2-4GB RAM 的旧电脑）的有用调整。

包括：

- zram（RAM 中的压缩交换）
- 交换文件
- 交换优先级
- BFQ 磁盘调度器
- 内核设置（sysctl）
- I/O 延迟调整

---

# 1. 创建交换文件

创建 6GB 交换文件。

```bash
fallocate -l 6G /swapfile
```

所需权限：

```bash
chmod 600 /swapfile
```

创建交换区：

```bash
mkswap /swapfile
```

激活：

```bash
swapon /swapfile
```

添加到`/etc/fstab`：

```text
/swapfile none swap defaults,pri=20 0 0
```

---

# 2. 分区交换（如果存在）

还要添加到`/etc/fstab`：

```text
UUID=SEU-UUID none swap defaults,pri=10 0 0
```

远端UUID：

```bash
blkid
```

---

# 3.配置zram（RAM中的压缩交换区）

加载模块：

```bash
modprobe zram
```

选择快速压缩机：

```bash
echo lz4 > /sys/block/zram0/comp_algorithm
```

设置大小（例如：1GB）：

```bash
echo 1G > /sys/block/zram0/disksize
```

创建交换：

```bash
mkswap /dev/zram0
```

以更高优先级激活：

```bash
swapon -p 100 /dev/zram0
```

要检查：

```bash
zramctl
```

或者

```bash
cat /proc/swaps
```

---

# 4.开机自动激活zram（runit）

创建脚本：

```
/etc/runit/core-services/10-zram.sh
```

内容：

```bash
#!/bin/sh

modprobe zram

echo lz4 > /sys/block/zram0/comp_algorithm
echo 1G > /sys/block/zram0/disksize

mkswap /dev/zram0
swapon -p 100 /dev/zram0
```

允许：

```bash
chmod +x /etc/runit/core-services/10-zram.sh
```

---

# 5. 内核设置（sysctl）

编辑：

```
/etc/sysctl.conf
```

添加：

```text
vm.swappiness=15
vm.vfs_cache_pressure=200
vm.dirty_background_ratio=5
vm.dirty_ratio=10
```

申请：

```bash
sysctl -p
```

---

# 6.Verificar透明大页面

查看状态：

```bash
cat /sys/kernel/mm/transparent_hugepage/enabled
```

如果它没有设置为“never”，则禁用它。

创建脚本：

```
/etc/runit/core-services/05-thp.sh
```

内容：

```bash
#!/bin/sh
echo never > /sys/kernel/mm/transparent_hugepage/enabled
```

允许：

```bash
chmod +x /etc/runit/core-services/05-thp.sh
```

---

# 7.改进磁盘调度程序

查看当前调度程序：

```bash
cat /sys/block/sda/queue/scheduler
```

交换到 BFQ：

```bash
echo bfq > /sys/block/sda/queue/scheduler
```

---

# 8.调整BFQ参数

查看参数：

```bash
ls /sys/block/sda/queue/iosched
```

启用低延迟：

```bash
echo 1 > /sys/block/sda/queue/iosched/low_latency
```

减少请求之间的等待：

```bash
echo 0 > /sys/block/sda/queue/iosched/slice_idle
```

---

# 9.无需开机自动应用BFQ

创建脚本：

```
/etc/runit/core-services/03-bfq.sh
```

内容：

```bash
#!/bin/sh

echo bfq > /sys/block/sda/queue/scheduler
echo 1 > /sys/block/sda/queue/iosched/low_latency
echo 0 > /sys/block/sda/queue/iosched/slice_idle
```

允许：

```bash
chmod +x /etc/runit/core-services/03-bfq.sh
```

---

# 10.检查系统状态

记忆：

```bash
free -h
```

交换：

```bash
cat /proc/swaps
```

兹拉姆：

```bash
zramctl
```

---

# 预期结果

内存使用顺序：

```
RAM
 ↓
zram
 ↓
swapfile
 ↓
swap partition
```

好处：

- 降低磁盘交换使用率
- 更低的延迟
- 反应更快的系统
- 在低内存机器上更好地使用 RAM

特别适用于：

- 旧电脑
- HDD 慢速
- 具有 2–4GB RAM 的系统
