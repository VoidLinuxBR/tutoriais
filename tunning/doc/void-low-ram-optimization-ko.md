# 메모리 및 디스크 최적화(기존 PC / RAM 부족)

이 가이드에서는 RAM이 낮고 HDD가 느린 컴퓨터(예: 2~4GB RAM이 있는 구형 PC)에 대한 유용한 조정 사항을 보여줍니다.

포함:

- zram(RAM의 압축 스왑)
- 스왑 파일
- 스왑 우선순위
- BFQ 디스크 스케줄러
- 커널 설정(sysctl)
- I/O 지연 시간 조정

---

# 1. 스왑파일 생성

6GB 스왑 파일을 만듭니다.

```bash
fallocate -l 6G /swapfile
```

필요한 권한:

```bash
chmod 600 /swapfile
```

스왑 영역 만들기:

```bash
mkswap /swapfile
```

활성화:

```bash
swapon /swapfile
```

`/etc/fstab`에 추가:

```text
/swapfile none swap defaults,pri=20 0 0
```

---

# 2. 파티션 스왑(존재하는 경우)

또한 `/etc/fstab`에 추가하십시오:

```text
UUID=SEU-UUID none swap defaults,pri=10 0 0
```

원거리 UUID:

```bash
blkid
```

---

# 3. zram 구성(RAM의 압축 스왑)

모듈 로드:

```bash
modprobe zram
```

빠른 압축기 선택:

```bash
echo lz4 > /sys/block/zram0/comp_algorithm
```

크기 설정(예: 1GB):

```bash
echo 1G > /sys/block/zram0/disksize
```

스왑 생성:

```bash
mkswap /dev/zram0
```

더 높은 우선순위로 활성화:

```bash
swapon -p 100 /dev/zram0
```

확인하려면:

```bash
zramctl
```

또는

```bash
cat /proc/swaps
```

---

# 4. 부팅 시 자동으로 zram 활성화(runit)

스크립트 만들기:

```
/etc/runit/core-services/10-zram.sh
```

콘텐츠:

```bash
#!/bin/sh

modprobe zram

echo lz4 > /sys/block/zram0/comp_algorithm
echo 1G > /sys/block/zram0/disksize

mkswap /dev/zram0
swapon -p 100 /dev/zram0
```

허가:

```bash
chmod +x /etc/runit/core-services/10-zram.sh
```

---

# 5. 커널 설정(sysctl)

편집하다:

```
/etc/sysctl.conf
```

추가하려면:

```text
vm.swappiness=15
vm.vfs_cache_pressure=200
vm.dirty_background_ratio=5
vm.dirty_ratio=10
```

적용하다:

```bash
sysctl -p
```

---

# 6. Verificar 투명 거대 페이지

상태 보기:

```bash
cat /sys/kernel/mm/transparent_hugepage/enabled
```

'never'로 설정되어 있지 않으면 비활성화하세요.

스크립트 만들기:

```
/etc/runit/core-services/05-thp.sh
```

콘텐츠:

```bash
#!/bin/sh
echo never > /sys/kernel/mm/transparent_hugepage/enabled
```

허가:

```bash
chmod +x /etc/runit/core-services/05-thp.sh
```

---

# 7. 디스크 스케줄러 개선

현재 스케줄러 보기:

```bash
cat /sys/block/sda/queue/scheduler
```

BFQ로 교환:

```bash
echo bfq > /sys/block/sda/queue/scheduler
```

---

# 8. BFQ 매개변수 조정

매개변수 보기:

```bash
ls /sys/block/sda/queue/iosched
```

낮은 대기 시간 활성화:

```bash
echo 1 > /sys/block/sda/queue/iosched/low_latency
```

요청 간 대기 시간을 줄입니다.

```bash
echo 0 > /sys/block/sda/queue/iosched/slice_idle
```

---

# 9. 부팅하지 않고 자동으로 BFQ 적용

스크립트 만들기:

```
/etc/runit/core-services/03-bfq.sh
```

콘텐츠:

```bash
#!/bin/sh

echo bfq > /sys/block/sda/queue/scheduler
echo 1 > /sys/block/sda/queue/iosched/low_latency
echo 0 > /sys/block/sda/queue/iosched/slice_idle
```

허가:

```bash
chmod +x /etc/runit/core-services/03-bfq.sh
```

---

# 10. 시스템 상태 확인

메모리:

```bash
free -h
```

교환:

```bash
cat /proc/swaps
```

zram:

```bash
zramctl
```

---

# 예상되는 결과

메모리 사용 순서:

```
RAM
 ↓
zram
 ↓
swapfile
 ↓
swap partition
```

이익:

- 디스크 스왑 사용량 감소
- 낮은 대기 시간
- 더욱 반응성이 뛰어난 시스템
- 메모리가 적은 시스템에서 더 나은 RAM 사용량

특히 다음과 같은 경우에 유용합니다.

- 오래된 PC
- HDD 렌토
- 2~4GB RAM을 갖춘 시스템
