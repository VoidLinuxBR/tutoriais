# 🐧 Void Linux + GNOME — 튜토리얼

> ⚠️ **중요 — 시작하기 전에 읽어보세요**
>
> 이 튜토리얼은 **명시적으로 표시된** 경우를 제외하고 **`루트`로 실행해서는 안 됩니다**.
>
> 모든 명령은 **일반 사용자**가 필요할 때 `sudo`를 사용하여 실행하도록 설계되었습니다.
>
> `root`로 로그인하여 전체 튜토리얼을 실행합니다.
> - 권한 논리를 깨뜨림
> - `sudo` 구성과 같은 단계를 무효화합니다.
> - 소리 없는 오류나 예상치 못한 동작이 발생할 수 있습니다.
>
> 👉 **추천**
> 방금 시스템을 설치하고 `root`로 로그인한 경우:
>
> 1. 일반 사용자 생성
> 2. 이 사용자로 로그인
> 3. 정상적으로 튜토리얼을 따르세요.
>
> Unix/Linux 시스템의 기본 규칙:
>
> **`root`는 예외입니다. 일반 사용자가 원칙입니다.**

---

## 0. 루트 비밀번호를 묻지 않도록 sudo - 휠 그룹 구성 -
```
sudo tee -a /etc/sudoers.d/g_wheel >/dev/null << EOF
%wheel ALL=(ALL:ALL) NOPASSWD: ALL
EOF

#필수권한
sudo chmod 440 /etc/sudoers.d/g_wheel
```

## 1. 시스템 업데이트
```
sudo xbps-install -Syu
```

## 2. 전체 그놈 설치(메타 패키지)
```
sudo xbps-install -y gnome \
그놈 아이콘 테마 \
서류 \
네트워크 관리자 애플릿 \
확장 관리자 \
노틸러스 \
노틸러스-서류-확장 \
노틸러스-그놈-콘솔-확장 \
노틸러스-그놈-터미널-확장 \
그놈 터미널 \
아크 테마 \
파이어폭스 \
파이어폭스-i18n-pt-BR \
xarchiver \
그놈 디스크 유틸리티 \
갈라진 \
gvfs\
p7zip \
압축을 푼다 \
에그\
noto-글꼴-이모지 \
htop
```

## 3. GDM(공식 디스플레이 관리자) 설치
```
sudo xbps-install -y gdm
```

## 4. 디스플레이 드라이버

### 인텔
```
sudo xbps-install -y mesa-dri linux-firmware-intel
```

### 새로운 AMD(amdgpu)
```
sudo xbps-install -y mesa-dri xf86-video-amdgpu
```

### 오래된 AMD
```
sudo xbps-install -y mesa-dri xf86-video-ati
```

### Nvidia(오픈 드라이버)
```
sudo xbps-install -y 메사-누보-dri
```

## 5. PipeWire(Modern Void Sound) 설치
```
sudo xbps-install -y \
파이프와이어 \
전선 배관공 \
alsa-플러그인-pulseaudio \
alsa-파이프와이어 \
libjack-pipewire \
펄스 오디오 유틸리티 \
alsa-utils \
파부컨트롤
```

## 6. ALSA → PipeWire 통합
```
sudo mkdir -p /etc/alsa/conf.d
sudo ln -sf /usr/share/alsa/alsa.conf.d/50-pipewire.conf /etc/alsa/conf.d
sudo ln -sf /usr/share/alsa/alsa.conf.d/99-pipewire-default.conf /etc/alsa/conf.d
```

## 7. 파이프와이어 펄스 서버 활성화(PulseAudio 호환)
```
sudo mkdir -p /etc/pipewire/pipewire.conf.d
sudo ln -sf /usr/share/examples/pipewire/20-pipewire-pulse.conf /etc/pipewire/pipewire.conf.d/
```

## 8. 세션에서 PipeWire 자동 시작 활성화
```
mkdir -p ~/.config/autostart
ln -sf /usr/share/applications/pipewire.desktop ~/.config/autostart/
ln -sf /usr/share/applications/pipewire-pulse.desktop ~/.config/autostart/
ln -sf /usr/share/applications/wireplumber.desktop ~/.config/autostart/
```

## 9. (선택 사항) startx용 .xinitrc 생성
```
고양이 <<EOF > ~/.xinitrc
#!/bin/sh
setxkbmap -layout br -variant abnt2 &
그놈 세션 실행
EOF
```

## 10. 시간대 구성 - 시간대를 정의합니다.
```
sudo ln -sfv /usr/share/zoneinfo/America/Sao_Paulo /etc/localtime
```

## 11. 로케일 구성
```
Sudo -i -i -e 's/s/#\(en_.us.utf-8 utf-8\)/' -e 's/^#\pt_br.br.br.
```

## 12. /etc/rc.conf를 사용자 정의합니다. 콘솔의 기본 시간대, 키보드 레이아웃 및 글꼴을 설정합니다. 필요에 따라 변경하세요.
```
sudo tee -a /etc/rc.conf >/dev/null << EOF
TIMEZONE="미국/Sao_Paulo"
키맵="br-abnt2"
글꼴=Lat2-Terminus16
EOF
```

## 13. /etc/locale.conf를 사용자 정의합니다. 언어를 설정합니다. 필요에 따라 변경하세요.
```
sudo tee /etc/locale.conf >/dev/null << EOF
LANG=pt_BR.UTF-8
언어=pt_BR.UTF-8
LC_COLLATE=pt_BR.UTF-8
EOF
```

## 14. 재구성
```
sudo xbps-재구성 -fa
```

## 15. 필수 서비스 활성화(runit)
```
sudo ln -s /etc/sv/dbus /var/service/
sudo ln -s /etc/sv/elogind /var/service/
sudo ln -s /etc/sv/polkitd /var/service/
sudo ln -s /etc/sv/NetworkManager /var/service/
sudo ln -s /etc/sv/gdm /var/service/
```

## 마무리
- GDM 사용 → 시스템이 GNOME에서 직접 시작됩니다.
- GDM이 없으면 → `startx`를 사용하십시오(`.xinitrc`가 존재하는 경우).
