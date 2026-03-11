# 記憶體和磁碟優化（舊電腦/低 RAM）

本指南展示了對 RAM 較低且 HDD 較慢的機器（例如具有 2-4GB RAM 的舊電腦）的有用調整。

包括：

- zram（RAM 中的壓縮交換）
- 交換文件
- 交換優先權
- BFQ 磁碟調度器
- 內核設定（sysctl）
- I/O 延遲調整

---

# 1. 建立交換文件

建立 6GB 交換檔案。

```bash
fallocate -l 6G /swapfile
```

所需權限：

```bash
chmod 600 /swapfile
```

建立交換區：

```bash
mkswap /swapfile
```

啟用設定:

```bash
swapon /swapfile
```

加到`/etc/fstab`：

```text
/swapfile none swap defaults,pri=20 0 0
```

---

# 2. 分區交換（如果存在）

也要加入到`/etc/fstab`：

```text
UUID=SEU-UUID none swap defaults,pri=10 0 0
```

遠端UUID：

```bash
blkid
```

---

# 3.配置zram（RAM中的壓縮交換區）

加載模組：

```bash
modprobe zram
```

選擇快速壓縮機：

```bash
echo lz4 > /sys/block/zram0/comp_algorithm
```

設定大小（例如：1GB）：

```bash
echo 1G > /sys/block/zram0/disksize
```

創建交換：

```bash
mkswap /dev/zram0
```

以更高優先權激活：

```bash
swapon -p 100 /dev/zram0
```

要檢查：

```bash
zramctl
```

或者

```bash
cat /proc/swaps
```

---

# 4.開機自動啟動zram（runit）

建立腳本：

```
/etc/runit/core-services/10-zram.sh
```

內容：

```bash
#!/bin/sh

modprobe zram

echo lz4 > /sys/block/zram0/comp_algorithm
echo 1G > /sys/block/zram0/disksize

mkswap /dev/zram0
swapon -p 100 /dev/zram0
```

允許：

```bash
chmod +x /etc/runit/core-services/10-zram.sh
```

---

# 5. 核心設定（sysctl）

編輯：

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

申請：

```bash
sysctl -p
```

---

# 6.Verificar透明大頁面

查看狀態：

```bash
cat /sys/kernel/mm/transparent_hugepage/enabled
```

如果它沒有設定為“never”，則停用它。

建立腳本：

```
/etc/runit/core-services/05-thp.sh
```

內容：

```bash
#!/bin/sh
echo never > /sys/kernel/mm/transparent_hugepage/enabled
```

允許：

```bash
chmod +x /etc/runit/core-services/05-thp.sh
```

---

# 7.改進磁碟調度程序

查看目前調度程序：

```bash
cat /sys/block/sda/queue/scheduler
```

交換到 BFQ：

```bash
echo bfq > /sys/block/sda/queue/scheduler
```

---

# 8.調整BFQ參數

查看參數：

```bash
ls /sys/block/sda/queue/iosched
```



```bash
echo 1 > /sys/block/sda/queue/iosched/low_latency
```

減少請求之間的等待：

```bash
echo 0 > /sys/block/sda/queue/iosched/slice_idle
```

---

# 9.無需開機自動應用BFQ

建立腳本：

```
/etc/runit/core-services/03-bfq.sh
```

內容：

```bash
#!/bin/sh

echo bfq > /sys/block/sda/queue/scheduler
echo 1 > /sys/block/sda/queue/iosched/low_latency
echo 0 > /sys/block/sda/queue/iosched/slice_idle
```

允許：

```bash
chmod +x /etc/runit/core-services/03-bfq.sh
```

---

# 10.檢查系統狀態

記憶：

```bash
free -h
```

交換：

```bash
cat /proc/swaps
```

茲拉姆：

```bash
zramctl
```

---

# 預期結果

記憶體使用順序：

```
RAM
 ↓
zram
 ↓
swapfile
 ↓
swap partition
```

好處：

- 降低磁碟交換使用率
- 更低的延遲
- 反應更快的系統
- 在低記憶體機器上更好地使用 RAM

特別適用於：

- 舊電腦
- HDD 慢速
- 具有 2–4GB RAM 的系統
