# Оптимизация памяти и диска (старый компьютер/недостаток оперативной памяти)

В этом руководстве показаны полезные настройки для компьютеров с небольшим объемом оперативной памяти и медленным жестким диском (например, старые компьютеры с 2–4 ГБ оперативной памяти).

Включает:

- zram (сжатый своп в оперативной памяти)
- файл подкачки
- приоритет обмена
- Планировщик дисков BFQ
- настройки ядра (sysctl)
- Настройки задержки ввода-вывода

---

# 1. Создать файл подкачки

Создайте файл подкачки размером 6 ГБ.

```bash
fallocate -l 6G /swapfile
```

Требуемое разрешение:

```bash
chmod 600 /swapfile
```

Создать область подкачки:

```bash
mkswap /swapfile
```

Активировать:

```bash
swapon /swapfile
```

Добавьте в /etc/fstab:

```text
/swapfile none swap defaults,pri=20 0 0
```

---

# 2. Замена раздела (если существует)

Также добавьте в /etc/fstab:

```text
UUID=SEU-UUID none swap defaults,pri=10 0 0
```

Дальний UUID:

```bash
blkid
```

---

# 3. Настроить zram (сжатый своп в оперативной памяти)

Загрузочный модуль:

```bash
modprobe zram
```

Выберите быстрый компрессор:

```bash
echo lz4 > /sys/block/zram0/comp_algorithm
```

Установить размер (пример: 1 ГБ):

```bash
echo 1G > /sys/block/zram0/disksize
```

Создать обмен:

```bash
mkswap /dev/zram0
```

Активировать с более высоким приоритетом:

```bash
swapon -p 100 /dev/zram0
```

Чтобы проверить:

```bash
zramctl
```

или

```bash
cat /proc/swaps
```

---

# 4. Автоматически активировать zram при загрузке (runit)

Создать скрипт:

```
/etc/runit/core-services/10-zram.sh
```

Содержание:

```bash
#!/bin/sh

modprobe zram

echo lz4 > /sys/block/zram0/comp_algorithm
echo 1G > /sys/block/zram0/disksize

mkswap /dev/zram0
swapon -p 100 /dev/zram0
```

Разрешение:

```bash
chmod +x /etc/runit/core-services/10-zram.sh
```

---

# 5. Настройки ядра (sysctl)

Редактировать:

```
/etc/sysctl.conf
```

Чтобы добавить:

```text
vm.swappiness=15
vm.vfs_cache_pressure=200
vm.dirty_background_ratio=5
vm.dirty_ratio=10
```

Применять:

```bash
sysctl -p
```

---

# 6. Verificar Прозрачные огромные страницы

Посмотреть статус:

```bash
cat /sys/kernel/mm/transparent_hugepage/enabled
```

Если для него не установлено значение «никогда», отключите его.

Создать скрипт:

```
/etc/runit/core-services/05-thp.sh
```

Содержание:

```bash
#!/bin/sh
echo never > /sys/kernel/mm/transparent_hugepage/enabled
```

Разрешение:

```bash
chmod +x /etc/runit/core-services/05-thp.sh
```

---

# 7. Улучшите планировщик дисков.

Посмотреть текущий планировщик:

```bash
cat /sys/block/sda/queue/scheduler
```

Обмен на БФК:

```bash
echo bfq > /sys/block/sda/queue/scheduler
```

---

# 8. Настройте параметры BFQ

Просмотр параметров:

```bash
ls /sys/block/sda/queue/iosched
```

Включить низкую задержку:

```bash
echo 1 > /sys/block/sda/queue/iosched/low_latency
```

Уменьшите ожидание между запросами:

```bash
echo 0 > /sys/block/sda/queue/iosched/slice_idle
```

---

# 9. Автоматически применять BFQ без загрузки.

Создать скрипт:

```
/etc/runit/core-services/03-bfq.sh
```

Содержание:

```bash
#!/bin/sh

echo bfq > /sys/block/sda/queue/scheduler
echo 1 > /sys/block/sda/queue/iosched/low_latency
echo 0 > /sys/block/sda/queue/iosched/slice_idle
```

Разрешение:

```bash
chmod +x /etc/runit/core-services/03-bfq.sh
```

---

# 10. Проверьте состояние системы.

Память:

```bash
free -h
```

Менять:

```bash
cat /proc/swaps
```

зрам:

```bash
zramctl
```

---

# Ожидаемый результат

Порядок использования памяти:

```
RAM
 ↓
zram
 ↓
swapfile
 ↓
swap partition
```

Преимущества:

- меньшее использование подкачки дисков
- более низкая задержка
- более отзывчивая система
- Лучшее использование оперативной памяти на машинах с низким объемом памяти

Особенно полезно для:

- старые компьютеры
- HDD Ленто
- системы с 2–4 ГБ ОЗУ
