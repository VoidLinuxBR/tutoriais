# メモリとディスクの最適化 (古い PC / RAM が少ない)

このガイドでは、RAM が少なく HDD が遅いマシン (例: 2 ～ 4GB RAM を搭載した古い PC) に役立つ調整を示します。

含まれるもの:

- zram (RAM 内の圧縮スワップ)
- スワップファイル
- スワップ優先度
- BFQディスクスケジューラ
- カーネル設定 (sysctl)
- I/O レイテンシの調整

---

# 1.スワップファイルの作成

6GBのスワップファイルを作成します。

```bash
fallocate -l 6G /swapfile
```

必要な許可:

```bash
chmod 600 /swapfile
```

スワップ領域を作成します。

```bash
mkswap /swapfile
```

活性化：

```bash
swapon /swapfile
```

`/etc/fstab` に追加します。

```text
/swapfile none swap defaults,pri=20 0 0
```

---

# 2. パーティションのスワップ (存在する場合)

`/etc/fstab` にも追加します。

```text
UUID=SEU-UUID none swap defaults,pri=10 0 0
```

遠方の UUID:

```bash
blkid
```

---

# 3. zram (RAM 内の圧縮スワップ) を構成します。

ロードモジュール:

```bash
modprobe zram
```

高速コンプレッサーを選択します。

```bash
echo lz4 > /sys/block/zram0/comp_algorithm
```

設定サイズ (例: 1GB):

```bash
echo 1G > /sys/block/zram0/disksize
```

スワップを作成します。

```bash
mkswap /dev/zram0
```

より高い優先度でアクティブ化します。

```bash
swapon -p 100 /dev/zram0
```

確認するには:

```bash
zramctl
```

または

```bash
cat /proc/swaps
```

---

# 4. 起動時に zram を自動的にアクティブ化する (runit)

スクリプトを作成します。

```
/etc/runit/core-services/10-zram.sh
```

コンテンツ：

```bash
#!/bin/sh

modprobe zram

echo lz4 > /sys/block/zram0/comp_algorithm
echo 1G > /sys/block/zram0/disksize

mkswap /dev/zram0
swapon -p 100 /dev/zram0
```

許可：

```bash
chmod +x /etc/runit/core-services/10-zram.sh
```

---

# 5. カーネル設定(sysctl)

編集：

```
/etc/sysctl.conf
```

追加するには:

```text
vm.swappiness=15
vm.vfs_cache_pressure=200
vm.dirty_background_ratio=5
vm.dirty_ratio=10
```

適用する：

```bash
sysctl -p
```

---

# 6. Verificar の透明な巨大ページ

ステータスの表示:

```bash
cat /sys/kernel/mm/transparent_hugepage/enabled
```

「never」に設定されていない場合は、無効にします。

スクリプトを作成します。

```
/etc/runit/core-services/05-thp.sh
```

コンテンツ：

```bash
#!/bin/sh
echo never > /sys/kernel/mm/transparent_hugepage/enabled
```

許可：

```bash
chmod +x /etc/runit/core-services/05-thp.sh
```

---

# 7. ディスクスケジューラの改善

現在のスケジューラを表示します。

```bash
cat /sys/block/sda/queue/scheduler
```

BFQ に切り替えます:

```bash
echo bfq > /sys/block/sda/queue/scheduler
```

---

# 8. BFQパラメータを調整する

パラメータを表示します。

```bash
ls /sys/block/sda/queue/iosched
```

低遅延を有効にする:

```bash
echo 1 > /sys/block/sda/queue/iosched/low_latency
```

リクエスト間の待機を削減します。

```bash
echo 0 > /sys/block/sda/queue/iosched/slice_idle
```

---

# 9. ブートせずに BFQ を自動的に適用する

スクリプトを作成します。

```
/etc/runit/core-services/03-bfq.sh
```

コンテンツ：

```bash
#!/bin/sh

echo bfq > /sys/block/sda/queue/scheduler
echo 1 > /sys/block/sda/queue/iosched/low_latency
echo 0 > /sys/block/sda/queue/iosched/slice_idle
```

許可：

```bash
chmod +x /etc/runit/core-services/03-bfq.sh
```

---

# 10. システムステータスを確認する

メモリ：

```bash
free -h
```

スワップ：

```bash
cat /proc/swaps
```

ズラム:

```bash
zramctl
```

---

# 期待される結果

メモリ使用順序:

```
RAM
 ↓
zram
 ↓
swapfile
 ↓
swap partition
```

利点：

- ディスクスワップ使用量の削減
- 待ち時間の短縮
- より応答性の高いシステム
- メモリの少ないマシンでの RAM 使用率の向上

特に次の場合に役立ちます。

- 古いパソコン
- HDDレント
- 2 ～ 4 GB の RAM を搭載したシステム
