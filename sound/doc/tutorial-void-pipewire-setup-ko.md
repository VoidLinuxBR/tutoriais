# PipeWire + WirePlumber no Void Linux(2025)
실제 Void Linux 패키지만을 기반으로 한 기능적이고 간단한 가이드입니다.

공식 문서:
https://docs.voidlinux.org/config/media/pipewire.html

---

# 1. 올바른 패키지 설치

```
sudo xbps-install -S \
  pipewire \
  wireplumber \
  alsa-pipewire \
  alsa-lib \
  alsa-utils \
  alsa-plugins \
  alsa-firmware \
  pavucontrol \
  pulsemixer
```

이는 PipeWire on Void에 **유일하게 필요한 패키지**입니다.

---

# 2. 구성

## 2.1 ALSA → PipeWire 통합
```
sudo mkdir -p /etc/alsa/conf.d
sudo ln -sf /usr/share/alsa/alsa.conf.d/50-pipewire.conf /etc/alsa/conf.d
sudo ln -sf /usr/share/alsa/alsa.conf.d/99-pipewire-default.conf /etc/alsa/conf.d
mkdir -p ~/.config/pipewire/pipewire.conf.d
mkdir -p ~/.config/pipewire/pipewire-pulse.conf.d
```

## 2.2 파이프와이어 펄스 서버 활성화(PulseAudio 호환)
```
sudo mkdir -p /etc/pipewire/pipewire.conf.d
sudo ln -sf /usr/share/examples/pipewire/20-pipewire-pulse.conf /etc/pipewire/pipewire.conf.d/
```

## 2.3 세션에서 PipeWire 자동 시작 활성화
```
mkdir -p ~/.config/autostart
ln -sf /usr/share/applications/pipewire.desktop ~/.config/autostart/
ln -sf /usr/share/applications/pipewire-pulse.desktop ~/.config/autostart/
ln -sf /usr/share/applications/wireplumber.desktop ~/.config/autostart/
```
---

# 3. DE/WM에 의한 자동 시작

## 3.1 데스크탑 환경(혼자 실행)
이러한 DE는 PipeWire를 자동으로 시작합니다.
**할 일이 없습니다.**

- XFCE
- LXQt
- LXDE
- KDE 플라즈마(X11 및 Wayland)
- 그놈(X11 및 Wayland)
- 계피
- 죽음
- 계발

PipeWire는 개입 없이 DBus를 통해 작동됩니다.

---

## 3.2 창 관리자(수동으로 시작해야 함)

### i3 — `~/.xinitrc`

```
export XDG_RUNTIME_DIR="/run/user/$(id -u)"

dbus-run-session -- sh -c '
  pipewire &
  wireplumber &
  pipewire-pulse &
  exec i3
'
```

### bspwm — `~/.xinitrc`

```
export XDG_RUNTIME_DIR="/run/user/$(id -u)"

dbus-run-session -- sh -c '
  pipewire &
  wireplumber &
  pipewire-pulse &
  exec bspwm
'
```

### dwm — `~/.xinitrc`

```
export XDG_RUNTIME_DIR="/run/user/$(id -u)"

dbus-run-session -- sh -c '
  pipewire &
  wireplumber &
  pipewire-pulse &
  exec dwm
'
```

### 오픈박스 — `~/.xinitrc`

```
export XDG_RUNTIME_DIR="/run/user/$(id -u)"

dbus-run-session -- sh -c '
  pipewire &
  wireplumber &
  pipewire-pulse &
  exec openbox-session
'
```

### 플럭스박스 / IceWM / JWM — `~/.xinitrc`

```
export XDG_RUNTIME_DIR="/run/user/$(id -u)"

dbus-run-session -- sh -c '
  pipewire &
  wireplumber &
  pipewire-pulse &
  exec fluxbox
'
```

---

## 3.3 Wayland WM

### 스웨이 — `~/.config/sway/config`

```
exec pipewire
exec wireplumber
exec pipewire-pulse
```

### 웨이파이어

```
exec pipewire
exec wireplumber
exec pipewire-pulse
```

### 하이프랜드

```
exec-once = pipewire
exec-once = wireplumber
exec-once = pipewire-pulse
```

---

# 4. 필수 그룹(elogind가 없으면 무효)

```
sudo usermod -aG audio,video $USER
```

로그아웃/로그인.

---

# 5. 일반 테스트

파이프와이어:
```
ps aux | grep pipewire
```

와이어배관공:
```
ps aux | grep wireplumber
```

PulseAudio 호환:
```
pactl info | grep "Server Name"
```

예상되는:
```
Server Name: PulseAudio (on PipeWire 0.3.x)
```

이음매:
```
speaker-test -t wav -c 2
```

---

# 6. 문제 해결

이전 PulseAudio 구성을 제거하십시오.
```
rm -rf ~/.config/pulse ~/.pulse
```

DBus를 확인하세요.
```
ps aux | grep dbus
```

PipeWire를 수동으로 시작합니다.
```
dbus-run-session -- pipewire
```

PulseAudio에 대해 불평하는 애플리케이션:
```
pipewire-pulse &
```

---

# 7. 간략한 요약

설치하다:
```
sudo xbps-install pipewire wireplumber alsa-pipewire alsa-utils alsa-plugins alsa-lib alsa-firmware
```

자동으로 지원되는 DE:
- XFCE, LXQt, LXDE, GNOME, KDE, 시나몬, MATE, 계몽

구성이 필요한 WM:
- i3, bspwm, dwm, Openbox, Fluxbox, sway, wayfire, 하이프랜드

PipeWire는 세션별로 작동합니다(runit를 사용하지 않음).

---

# 끝

