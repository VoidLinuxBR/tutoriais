# 🧩 VOID LINUX 튜토리얼 — EXT4, XFS, JFS 또는 BTRFS(하위 볼륨), LUKS, 최대 절전 모드 및 ZRAM을 사용한 하이브리드 설치(UEFI + BIOS)
### 개정 및 검증된 버전 — 올바른 파티션 + 범용 부팅

이 가이드는 기존, 신규 또는 문제가 있는 모든 유형의 시스템을 부팅할 수 있는 완전한 **하이브리드** Void Linux를 설치합니다.

- 💾 **최신 UEFI**(일반 입력 및 폴백 포함)
- 🧮 **BIOS/레거시**(완전한 호환성)
- 🧰 **BIOS 부팅이 포함된 GPT(EF02)** — 기존 하드웨어에 대한 최대 지원
- 🚀 **하위 볼륨이 있는 Btrfs**(선택 사항), 미리 만들어진 스냅샷
- 🔐 **GRUB과 완벽하게 호환되는 LUKS1**
- 🌙 **스왑 파일을 통한 실제 최대 절전 모드**
- 🧊 **성능을 위해 구성된 ZRAM**
- 🧱 **EXT4, XFS, JFS 및 BTRFS에 대한 전체 지원**
- 💡 **Initramfs/GRUB 자동 구성(LUKS + 재개)**

😀 **타협 없음, GRUB 재설치 없음, 시간 낭비 없음.**
😀 **NVRAM이 지워진 머신에서도 부팅이 보장됩니다(BOOTX64.EFI 폴백).**

---

# ▶️ 1. Bootar o Live ISO

제안: 뛰어난 호환성을 위해 glibc 버전을 사용하세요.
- 다음에서 iso를 다운로드하세요.
```
https://repo-default.voidlinux.org/live/current/void-live-x86_64-20250202-base.iso
```
- 또는 다음에서 최신 버전을 찾으세요.
```
https://voidlinux.org/download/
```

1. 루트로 로그인합니다.
```bash
로그인 : 루트
비밀번호 : voidlinux
```

2. 쉘을 *sh*에서 *bash*로 전환합니다.
*대시/sh* 많은 스크립트에서 사용하는 여러 기능을 **구현하지 않습니다**.
```bash
세게 때리다
```

3. 키보드 레이아웃을 **ABNT2**로 변경하여 악센트와 기호가 올바르게 매핑되도록 합니다.
```bash
로드키 br-abnt2
```

4. 터미널에 붙여넣기(선택 사항) - 색상, user@host:path 및 마지막 명령 상태(✔/✘)가 포함된 프롬프트가 표시됩니다. 유용하고 아름답습니다.
```bash
내보내기 PS1='\[\e[1;32m\]\u\[\e[1;33m\]@\[\e[1;36m\]\h\[\e[1;31m\]:\w \
$([[ $? -eq 0 ]] && echo -e "\e[1;32m✔" || echo -e "\e[1;31m✘$?") \
\[\e[0m\]\$ '
```

# ▶️ 2. 인터넷에 연결
- **Wi-Fi** *(케이블을 사용하는 경우 이 단계 건너뛰기)*:
```bash
wpa_passphrase "SSID" "비밀번호" > wifi.conf
wpa_supplicant -B -i wlan0 -c wifi.conf
dhcpcd wlan0
```

1. 연결을 테스트합니다.
```bash
핑 -c3 8.8.8.8
ping -c3 repo-default.voidlinux.org
```

2. 필수 패키지를 설치합니다:
⚠️ **중요:**
```bash
xbps-install -Sy xbps parted jfsutils xfsprogs nano zstd xz bash-completion
```
---

# ▶️ 3. 디스크 식별
1. 사용 가능한 디스크를 나열하고 장치 이름을 기록해 둡니다(예: `/dev/sda`, `/dev/vda`, `/dev/nvme0n1`).
```bash
fdisk -l | grep -E '^(디스크|디스크) '
```

# ▶️ 4. 튜토리얼에 사용되는 변수를 정의합니다:
⚠️ **중요:**

1. **사용 전** 장치 정의:
> 1. **가정** 튜토리얼에서는 `/dev/sda`(일반) 또는 `/dev/nvme0n1`(nvme)
> 2. 디스크에 따라 **조정**(**하나** 또는 **다른** 모델 선택)

**일반** 디스크의 경우(예: /dev/sda)
```bash
장치 내보내기=/dev/sda
DEV_BIOS=${DEVICE}1 내보내기
DEV_EFI=${DEVICE}2 내보내기
DEV_ROOT=${DEVICE}3 내보내기
DEV_LUKS=/dev/mapper/cryptroot 내보내기
```
**NVMe** 디스크(예: /dev/nvme0n1)의 경우 파티션 접미사가 변경됩니다(`p`).
```bash
장치 내보내기=/dev/nvme0n1
DEV_BIOS=${DEVICE}p1 내보내기
DEV_EFI=${DEVICE}p2 내보내기
DEV_RAIZ=${DEVICE}p3 내보내기
DEV_LUKS=/dev/mapper/cryptroot 내보내기
```

> 😀 **참고:**
> DEVICE → 전체 디스크
DEV_BIOS → BIOS 부팅 파티션(1~2MiB, FS 없음, 마운트되지 않음)
DEV_EFI → EFI 파티션(FAT32)
DEV_ROOT → 루트 파티션(일반 또는 LUKS)
DEV_LUKS → LUKS 매핑(/dev/mapper/cryptroot)

- 👉 여기서 디스크의 해부학적 구조를 정의합니다. 가이드의 다른 모든 내용은 이러한 변수를 따릅니다.
- 🔎 이것이 왜 필요한가요?
처음에 모든 것을 선언하면 다음 프로세스에서 오타가 방지되기 때문입니다.

2. **KEYMAP** 및 **TIMEZONE**을 정의합니다(필요에 따라 변경).
```bash
KEYMAP=br-abnt2 내보내기
```
```bash
수출 TIMEZONE=미국/Sao_Paulo
```

---

# ▶️ 5. 파티션 디스크
> - BIOS 파티션 **반드시**가 첫 번째여야 합니다. 이를 통해 오래된 마더보드, 문제가 있는 부트로더 및 디스크의 첫 번째 영역에 부팅 코드가 필요한 BIOS와의 호환성이 향상됩니다.
> - ESP는 문제 없이 나중에 올 수 있습니다. UEFI는 위치에 신경 쓰지 않습니다.

### 이상적이고 올바른 순서:

- 1️⃣ BIOS 부팅(EF02)
- 2️⃣ ESP (EFI 시스템, FAT32)
- 3️⃣ Btrfs/Ext4/Xfs/Jfs (루트)

### parted를 사용한 파티션(자동)
> 여기서는 **DEVICE**가 이미 정의되어 있으므로 "마법의" 변수가 없습니다.
```
와이프 -a "${DEVICE}"
parted --script "${DEVICE}" -- \
mklabel gpt \
mkpart 기본 1MiB 2MiB 이름 1 BIOS 세트 1 bios_grub on \
mkpart 기본 fat32 2MiB 514MiB 이름 2 EFI 세트 2 esp on \
mkpart 기본 514MiB 100% 이름 3 ROOT \
정렬-검사 최적 1 \
정렬-검사 최적 2 \
정렬 확인 최적 1
parted --script "${DEVICE}" -- 인쇄
```
> - 파티션 1 → BIOS 부팅(bios_grub, FS 없음, 마운트되지 않음)
> - 파티션 2 → EFI(FAT32)
> - 파티션 3 → ROOT(나중에 LUKS 유무에 관계없이 EXT4/XFS/JFS/BTRFS로 포맷할 예정입니다)
> - FS가 묶이는 것을 피하기 위해 FS를 정확하게 지정하지 않고 mkpart 기본 514MiB 100%를 사용했습니다. 나중에 FS를 선택하세요.
---

# ▶️ 6. 설치 모드(NORMAL 또는 LUKS)를 선택하세요.
⚠️ **중요:**
> 아래 두 블록 중 하나만 선택하세요.
**두 단계를 모두 실행하지 마세요**.

1. 일반 설치**(LUKS 제외)**
```bash
내보내기 DISK="${DEV_RAIZ}"
```
- DISK를 실제 장치 /dev/sda3으로 설정합니다.

2. **LUKS를 사용한 설치**(암호화된 루트)
```
# LUKS1(GRUB 호환)의 루트 파티션만 암호화합니다. 전체 디스크는 암호화하지 않습니다.
# YES로 확인하여 파티션을 암호화합니다.
cryptsetup luksFormat --type luks1 "${DEV_RAIZ}"

# 암호로 파티션을 엽니다.
cryptsetup "${DEV_RAIZ}" 암호화 루트 열기

# 이제부터 실제 루트는 매핑된 장치입니다.
내보내기 DISK="${DEV_LUKS}"
```
- LUKS는 전체 디스크가 아닌 /dev/sda3 위에 위치합니다.
- 시스템은 /dev/mapper/cryptroot에 설치됩니다.

👉 이제부터는 모든 것이 $DISK.를 사용합니다.

---

# ▶️ 7. 파일 시스템(FS) 생성 및 루트 마운트
⚠️ **중요:**
> 아래 두 블록 중 하나만 선택하세요.

1. **EXT4**
```
mkfs.ext4 -F "${DISK}" -L 루트
마운트 -v "${DISK}" /mnt
```
2. **XFS**
```
mkfs.xfs -f "${DISK}"
마운트 -v "${DISK}" /mnt
```
3. **JFS**
```
mkfs.jfs -f "${DISK}"
마운트 -v "${DISK}" /mnt
```
4. **간단한 BTRFS**
```
mkfs.btrfs -f "${DISK}" -L 루트
마운트 -v "${DISK}" /mnt
```
5. **BTRFS com subvolumes**
```
mkfs.btrfs -f "${DISK}" -L 루트

${DISK} /mnt 마운트
btrfs 하위 볼륨 생성 /mnt/@
btrfs 하위 볼륨 생성 /mnt/@home
btrfs 하위 볼륨 생성 /mnt/@log
btrfs 하위 볼륨 생성 /mnt/@cache
btrfs 하위 볼륨 생성 /mnt/@snapshots
마운트 해제 /mnt

mount -o defaults,noatime,ssd,compress=zstd:3,discard=async,space_cache=v2,commit=300,subvol=/@ ${DISK} /mnt
mkdir -p /mnt/{boot/efi,home,var/log,var/cache,.snapshots,swap}

mount -o defaults,noatime,ssd,compress=zstd:3,discard=async,space_cache=v2,commit=300,subvol=/@home ${DISK} /mnt/home
mount -o defaults,noatime,ssd,compress=zstd:3,discard=async,space_cache=v2,commit=300,subvol=/@cache ${DISK} /mnt/var/cache
mount -o defaults,noatime,ssd,compress=zstd:3,discard=async,space_cache=v2,commit=300,subvol=/@log ${DISK} /mnt/var/log
mount -o defaults,noatime,ssd,compress=zstd:3,discard=async,space_cache=v2,commit=300,subvol=/@snapshots ${DISK} /mnt/.snapshots
```
---

# ▶️ 8. ESP(EFI) 준비 및 조립
```
mkfs.fat -F32 -I "${DEV_EFI}"
mkdir -p /mnt/boot/efi
마운트 -v "${DEV_EFI}" /mnt/boot/efi
```
>💡 BIOS 파티션(${DEV_BIOS})에는 파일 시스템이 없고, 포맷되지 않으며, 마운트되지 않습니다.
---

# ▶️ 9. 설치 o Void Linux no chroot

1. chroot(/mnt)에서 사용할 저장소 키(XBPS 키)를 복사합니다.
```
mkdir -p /A{tc, 증기/xbps/xbps)
cp -rpaf /var/db/xbps/keys/*.plist /mnt/var/db/xbps/keys/
cp -fpa /etc/resolv.conf /mnt/etc/resolv.conf
```

2. 새로 마운트된 디스크에 기본 시스템을 설치합니다.
```
xbps-설치 -Sy -R https://repo-default.voidlinux.org/current \
-r /mnt \
기본 시스템 btrfs-progs cryptsetup grub grub-x86_64-efi dracut linux \
리눅스 헤더 리눅스 펌웨어 리눅스 펌웨어 네트워크 glibc-로케일 \
xtools dhcpcd openssh vim nano grc zstd xz bash-completion vpm vsv \
socklog-void wget net-tools tmate ncurses jfsutils xfsprogs duf tree eza chrony
```
---

# ▶️ 10. Gerar fstab no /mnt (chroot)
```bash
# /mnt/etc/fstab에 fstab을 생성합니다.
xgenfstab -U /mnt > /mnt/etc/fstab
```

```bash
# 올바르게 생성되었는지 확인
고양이 /mnt/etc/fstab
```

# ▶️ 11. chroot를 사용하여 설치된 시스템에 액세스합니다.

1. 크로잇이 아닌 고용:
```
xchroot /mnt /bin/bash
```
---

# ▶️ 12. 초기 설정 (chroot에서)
```
# 호스트 이름 구성 - 호스트 이름을 정의합니다.
echo void > /etc/호스트 이름

# 시간대 구성 - 시간대를 정의합니다.
ln -sfv /usr/share/zoneinfo/"${TIMEZONE}" /etc/localtime

# 로케일 구성
-i -e 's/^#\(en_.Utf-8 UTF-8\)/)/' \
-E 's/^#\pt_br.br.utf-8 UTF-8\)/' \' \
/etc/default/libc-locales

# 로케일 생성
xbps-재구성 -f glibc-로케일

# /var/service 심볼릭 링크에서 발생할 수 있는 오류를 수정합니다(중요):
rm -f /var/서비스
ln -sf /etc/runit/runsvdir/default /var/service

# 일부 서비스 활성화
ln -sf /etc/sv/dbus /var/service/
ln -sf /etc/sv/dhcpcd /var/service/
ln -sf /etc/sv/sshd /var/service/
ln -sf /etc/sv/nanoklogd /var/service/
ln -sf /etc/sv/socklog-unix /var/service/
ln -sf /etc/sv/chronyd /var/service/

# sudo - 휠 그룹 구성(선택 사항이지만 권장됨)
고양이 << 'EOF' > /etc/sudoers.d/g_wheel
%wheel ALL=(ALL:ALL) NOPASSWD: ALL
EOF
#필수권한
chmod 440 /etc/sudoers.d/g_wheel
```

## 사용자 생성 및 구성

⚠️ **중요:** 아래에서 실제 사용자 이름을 정의하세요.
```bash
NEWUSER=your_user_here로 내보내기
```

홈 디렉토리, 기본 그룹 및 Bash 쉘을 사용하여 사용자를 작성하십시오.
```bash
useradd -m -G 오디오, 비디오, 휠, tty -s /bin/bash ${NEWUSER}
```

사용자 비밀번호 설정(***중요***)
```bash
비밀번호 ${NEWUSER}
```

루트 사용자 비밀번호 설정(***중요***)
```bash
비밀번호 루트
```

루트 사용자의 기본 셸을 Bash로 변경
```bash
chsh -s /bin/bash 루트
```
---

# ▶️ 13. UUID 구성
⚠️ **중요:**
- 파티션의 UUID를 가져옵니다.
```
UUID_LUKS=$(blkid -s UUID -o 값 "${DEV_RAIZ}")
UUID_ROOT=$(blkid -s UUID -o 값 "${DISK}")
UUID_EFI=$(blkid -s UUID -o 값 "${DEV_EFI}")
```
---

# ▶️ 14. 최대 절전 모드를 지원하는 스왑 파일 생성(선택 사항)

### 중요 사항
```
- Btrfs의 스왑 파일은 항상 **prealloc**으로 표시되며 이는 정상입니다.
- RAM의 전체 크기일 필요는 없습니다.
- 대부분의 경우 최대 절전 모드에는 60%면 충분합니다.
- 무거운 하중의 경우 → 70% 또는 80%를 사용합니다.
```

1. 최적의 스왑 파일 크기를 자동으로 계산합니다.
```
# 최대 절전 모드에 대한 최신 권장 사항: 전체 RAM의 60%
SWAP_GB=$(LC_ALL=C awk '/MemTotal/ {print int($2 * 0.60 / 1024 / 1024)}' /proc/meminfo)
echo "권장 스왑 파일: ${SWAP_GB}G"
```
- 또는 원하는 크기를 수동으로 설정합니다.
```
SWAP_GB=4
echo "사용자 정의 스왑 파일: ${SWAP_GB}G"
```
2. 스왑 파일용 디렉터리 생성
```
mkdir -p /스왑
swapoff -a 2>/dev/null
rm -f /스왑/스왑 파일
```
3. COW 비활성화(Btrfs에 필요)
```
chattr +C /스왑
```
4. 이전에 정의된 크기로 스왑 파일을 생성합니다.
```
fallocate -l ${SWAP_GB}G /swap/swapfile
chmod 600 /스왑/스왑 파일
```
5. 스왑 파일 포맷 및 스왑 활성화
```
mkswap /스왑/스왑 파일
스왑온 /스왑/스왑 파일
```
6. 오프셋 가져오기:
```
# filefrag용 패키지를 설치합니다.
xbps-설치 -Sy e2fsprogs

# 오프셋을 구한다
offset=$(filefrag -v /swap/swapfile | awk '/^ *0:/{print $4}')
```
---

# ▶️ 15. GRUB 구성
⚠️ **중요:**
> 이 블록은 스마트합니다.
- LUKS를 사용하고 있는지 자동으로 감지
- 최대 절전 모드로 스왑 파일을 생성했는지 감지합니다.
- 아무것도 복제하지 않고 /etc/default/grub 조정
- 누락된 경우에만 필요한 라인을 생성합니다.
- 꼭 필요하지 않다면 아무 것도 변경하지 마세요.

아래 블록을 정확하게 사용하십시오.
```
HAS_RESUME=거짓
HAS_LUKS=거짓

[[ -n "${offset}" ]] && HAS_RESUME=true
[[ "${DISK}" = "${DEV_LUKS}" ]] && HAS_LUKS=true

# 안전을 위해 오래된 줄을 제거합니다
sed -i '/^[[:space:]]*GRUB_CMDLINE_LINUX_DEFAULT=/d' /etc/default/grub

# GRUB_CMDLINE_LINUX

# 기본 값
BASE="로그레벨=4"

# 요약 추가
$HAS_RESUME인 경우; 그 다음에
BASE="$BASE 이력서=UUID=${UUID_ROOT} 이력서_오프셋=${offset}"
BE

# LUKS 추가
$HAS_LUKS인 경우; 그 다음에
grep -q '^GRUB_ENABLE_CRYPTODISK=y' /etc/default/grub || echo 'GRUB_ENABLE_CRYPTODISK=y' >> /etc/default/grub
grep -q '^GRUB_PRELOAD_MODULES=' /etc/default/grub || echo 'GRUB_PRELOAD_MODULES="luks cryptodisk gcry_rijndael"' >> /etc/default/grub
BASE="$BASE rd.luks.uuid=${UUID_LUKS} rd.luks.name=${UUID_LUKS}=cryptroot 루트=/dev/mapper/cryptroot"
BE

# 마지막 줄을 올바르게 다시 만듭니다.
echo "GRUB_CMDLINE_LINUX_DEFAULT=\"${BASE}\"" >> /etc/default/grub
```
---

# ▶️ 16. initrd 다시 만들기
⚠️ **중요:**
```
모드=(/usr/lib/모듈/*)
KVER=$(기본 이름 "${mods[0]}")
에코 ${KVER}
dracut --force --kver ${KVER}
```
---

# ▶️ 17. 부팅 시 비밀번호를 두 번 묻는 것을 방지하기 위해 키 파일 생성(LUKS만 해당)
> 시스템이 LUKS를 사용하지 않는 경우 이 단계를 건너뜁니다.
```
if [ "${DISK}" = "${DEV_LUKS}" ]; 그 다음에
echo "LUKS 감지됨: 자동 잠금 해제를 위한 키 파일 생성 중..."

# 보안 키 파일 생성
dd if=/dev/urandom of=/boot/volume.key bs=64 개수=1
chmod 000 /boot/volume.key

# LUKS에 키 파일을 추가합니다(현재 비밀번호를 묻습니다)
cryptsetup luksAddKey "${DEV_RAIZ}" /boot/volume.key

# /etc/crypttab 구성
고양이 << EOF >> /etc/crypttab
cryptroot ${DEV_RAIZ} /boot/volume.key luks
EOF

# initramfs에 키 파일과 crypttab을 포함합니다.
mkdir -p /etc/dracut.conf.d
고양이 << EOF >> /etc/dracut.conf.d/10-crypt.conf
install_items+=" /boot/volume.key /etc/crypttab "
EOF

# 키파일 지원으로 initramfs를 재생성합니다.
xbps-재구성 -fa
또 다른
echo "LUKS가 없는 시스템: 키 파일 생성을 건너뛰는 중입니다."
BE
```

# ▶️ 18. **BIOS** 및 **UEFI**(실제 하이브리드)에 GRUB 설치
1. BIOS용 GRUB 설치(레거시)
```
grub-install --target=i386-pc ${DEVICE}
```
2. UEFI용 GRUB 설치
```
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=void
```
3. UEFI 폴백(범용 부팅)을 생성합니다. 이 파일은 NVRAM이 지워진 경우에도 부팅을 보장합니다.
```
mkdir -p /boot/efi/EFI/BOOT
cp -f /boot/efi/EFI/void/grubx64.efi /boot/efi/EFI/BOOT/BOOTX64.EFI
```
4. 최종 GRUB 파일 생성
```
grub-mkconfig -o /boot/grub/grub.cfg
```

---

# ▶️ 19. 맞춤형 사용자 설정:

1. 환경 설정:

```
# /etc/xbps.d/00-repository-main.conf 사용자 정의
mkdir -pv /etc/xbps.d
고양이 << 'EOF' >> /etc/xbps.d/00-repository-main.conf
저장소=https://repo-fastly.voidlinux.org/current
#repository=https://repo-fastly.voidlinux.org/current/nonfree
#repository=https://repo-fastly.voidlinux.org/current/multilib
#repository=https://repo-fastly.voidlinux.org/current/multilib/nonfree

저장소=https://void.chililinux.com/voidlinux/current
#repository=https://void.chililinux.com/voidlinux/current/extras
#repository=https://void.chililinux.com/voidlinux/current/nonfree
#repository=https://void.chililinux.com/voidlinux/current/multilib
#repository=https://void.chililinux.com/voidlinux/current/multilib/nonfree
EOF

# /etc/rc.conf를 사용자 정의합니다. 콘솔의 기본 시간대, 키보드 레이아웃 및 글꼴을 설정합니다. 필요에 따라 변경하세요.
고양이 << EOF >> /etc/rc.conf
TIMEZONE="${TIMEZONE}"
KEYMAP="${KEYMAP}"
글꼴=Lat2-Terminus16
EOF

# 루트의 .bashrc 사용자 정의
wget --quiet --no-check-certificate \
-O /etc//skel/.bashrc \
"https://raw.githubusercontent.com/voidlinux-br/void-install/refs/heads/main/.bashrc"
chown 루트:루트 /etc/skel/.bashrc
chmod 644 /etc/skel/.bashrc

고양이 << 'EOF' > /etc/skel/.bash_profile
# ~/.bash_profile — .bashrc를 Void에 로드합니다.

# .bashrc가 존재하면 로드합니다.
if [ -f ~/.bashrc ]; 그 다음에
소스 ~/.bashrc
BE
EOF

# 루트와 사용자에게 복사
/root "/home/${NEWUSER}"의 d에 대해; 하다
cp -f /etc/skel/.bash_profile "$d/"
cp -f /etc/skel/.bashrc "$d/"
완료

chown "${NEWUSER}:${NEWUSER}" "/home/${NEWUSER}/.bash_profile" "/home/${NEWUSER}/.bashrc"
chmod 644 "/home/${NEWUSER}/.bash_profile" "/home/${NEWUSER}/.bashrc"

# 사용자 정의 svlogtail 다운로드
wget --quiet --no-check-certificate \
-O /usr/bin/svlogtail\
"https://raw.githubusercontent.com/voidlinux-br/void-install/refs/heads/main/svlogtail"
chmod +x /usr/bin/svlogtail
```

2. SSH 구성(선택 사항이지만 권장됨):
```
mkdir -pv /etc/ssh/sshd_config.d/
고양이 << 'EOF' > /etc/ssh/sshd_config.d/10-custom.conf
PermitTTY 예
PrintMotd 예
PrintLastLog 예
배너 /etc/issue.net

PermitRoot로그인 예
KbdInteractiveAuthentication 예
X11전달 예
Pubkey인증 예
PubkeyAcceptedKeyTypes=+ssh-rsa
AuthorizedKeys파일 .ssh/authorized_keys
비밀번호인증 예
ChallengeResponse인증 예
PAM을 사용하세요 예

하위 시스템 SFTP 내부 SFTP
EOF
```
---

# ▶️ 20. ZRAM 활성화(선택 사항)
Void Linux는 zramen 서비스를 사용하여 ZRAM을 활성화하고 SSD 스왑 사용량을 줄이고 로드 시 성능을 향상시키는 압축 메모리 블록을 생성합니다.
1. 지라멘 설치
```
xbps-설치 -Sy zramen
```
2. ZRAM 구성(권장 구성):
```
고양이 << 'EOF' > /etc/zramen.conf
zram_fraction=0.5
zram_devices=1
zram_algorithm=zstd
EOF
```
3. runit에서 서비스 활성화
```
ln -s
```
> ZRAM은 부팅할 때마다 자동으로 활성화됩니다.

---

# ▶️ 21. 설치 완료
1. Sair do chroot:
```
출구
```
2. /mnt(하위 볼륨 및 /boot/efi)에 마운트된 모든 파티션을 마운트 해제합니다.
```
umount -R /mnt
```
3. chroot 내에서 활성화된 스왑 파일이나 스왑 파티션을 비활성화합니다:
```
스왑오프 -a
```
4. 실제 부팅을 테스트하려면 물리적 머신이나 VM을 다시 시작하세요.
```
재부팅
```
> 설치 미디어를 제거하고 새로 설치된 디스크로 부팅하는 것을 잊지 마십시오.
즐기다!

---

# 🎉 완전한 하이브리드 미래 지향적 시스템
- 부팅 BIOS + UEFI
- 폴백 UEFI
- 스냅샷이 포함된 Btrfs(Snapper/Timeshift 지원)
- 스왑 파일을 사용한 실제 최대 절전 모드
- 성능을 위한 Zram

이 SSD는 **지구상의 모든 기계**를 부팅합니다.

# 면책조항

```
이 튜토리얼은 무료입니다. 원하는 대로 사용, 복사, 수정 및 재배포할 수 있습니다.
콘텐츠는 **MIT 라이선스**에 따라 제공되며 자체 라이선스에 따라 오픈 소스 소프트웨어에서 파생된 코드 조각이나 명령이 포함될 수 있습니다.

어떠한 보증도 제공되지 않습니다. 여기에 있는 모든 내용은 "있는 그대로" 제공됩니다.
자신의 책임하에 사용하십시오. 저자, 기여자, Void Linux 모두 이 자료의 사용으로 인한 손실, 손상, 시스템 오류 또는 결과에 대해 책임을 지지 않습니다.

원한다면 소스 코드를 얻고, 검토하고, 조정하고, 이 튜토리얼의 자신만의 버전을 생성할 수 있습니다.
```

