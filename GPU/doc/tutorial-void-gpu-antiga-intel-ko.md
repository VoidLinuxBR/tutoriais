# 전체 튜토리얼: Linux의 Intel GPU(Void Linux, XFCE 및 기존 하드웨어)

이 가이드에서는 Linux, 특히 GMA 3100, GMA X3100, HD 2000/3000 등과 같은 구형 하드웨어에서 Intel GPU를 식별, 구성 및 안정화하는 방법을 설명합니다.

---

# 1. Intel GPU 식별

실행하다:

```bash
lspci -k | grep -A3 VGA
```

또는:

```bash
inxi -G
```

출력 예:

```
Device: Intel 82G33/G31 Express Integrated Graphics
Driver: i915
Display: X.Org modesetting
OpenGL: 2.1 Mesa i915
```

---

# 2. 드라이버 Intel(Linux 없음)

모든 Intel GPU는 다음을 사용합니다.

```
kernel driver: i915
```

독점 드라이버를 설치할 필요가 없습니다.

Xorg 드라이버에는 두 가지가 있습니다.

- 모드설정 → 권장
- xf86-video-intel → 대부분의 경우 더 이상 사용되지 않음

어느 것이 사용 중인지 확인하세요.

```bash
inxi -G
```

표시되는 경우:

```
driver: modesetting
```

맞습니다.

---

# 3. 구형 Intel GPU(GMA 950, 3000, 3100, X3100, X4500)의 경우

이러한 GPU에는 다음과 같은 제한 사항이 있습니다.

- OpenGL 2.1 최대
- DRI3 불안정
- 작곡가가 충돌을 일으킴
- 최신 가속이 제대로 작동하지 않습니다

---

# 4. 안정적인 구성 권장

만들다:

```bash
sudo mkdir -p /etc/X11/xorg.conf.d
sudo nano /etc/X11/xorg.conf.d/20-intel.conf
```

콘텐츠:

```conf
Section "Device"
    Identifier "Intel Graphics"
    Driver "modesetting"
    Option "DRI" "2"
EndSection
```

---

# 5. XFCE 작성기 비활성화(이전 GPU에서는 필수)

실행하다:

```bash
xfconf-query -c xfwm4 -p /general/use_compositing -s false
```

확인하다:

```bash
xfconf-query -c xfwm4 -p /general/use_compositing
```

예상 결과:

```
false
```

---

# 6. 구형 Intel GPU용 Firefox 설정

만들다:

```bash
nano ~/.mozilla/firefox/*.default-release/user.js
```

콘텐츠:

```js
user_pref("layers.acceleration.disabled", true);
user_pref("gfx.webrender.force-disabled", true);
user_pref("gfx.xrender.enabled", true);
user_pref("media.ffmpeg.vaapi.enabled", false);
```

이렇게 하면 충돌이 방지됩니다.

---

# 7. OpenGL 지원 확인

실행하다:

```bash
glxinfo | grep "OpenGL version"
```

예:

```
OpenGL version string: 2.1 Mesa 25.3.3
```

이는 GMA 3100이 지원하는 최대값입니다.

---

# 8. 활성 가속도 확인

실행하다:

```bash
glxinfo | grep "direct rendering"
```

올바른 결과:

```
direct rendering: Yes
```

---

# 9. 로드된 드라이버 확인

실행하다:

```bash
lsmod | grep i915
```

결과:

```
i915
```

---

# 10. Xorg 상태 확인

실행하다:

```bash
grep -i driver /var/log/Xorg.0.log
```

올바른 결과:

```
modesetting
```

---

# 11. 아주 오래된 하드웨어에서는 xf86-video-intel을 사용하지 마세요

설치된 경우 제거:

```bash
sudo xbps-remove xf86-video-intel
```

모드 설정 드라이버가 더 안정적입니다.

---

# 12. DRI 확인

실행하다:

```bash
glxinfo | grep DRI
```

구형 GPU의 경우 다음을 사용하십시오.

```
DRI2
```

---

# 13. 해상도 확인

실행하다:

```bash
xrandr
```

---

# 14. 일반적인 문제 및 해결 방법

## XFCE 진입 시 재부팅

작곡가 비활성화:

```bash
xfconf-query -c xfwm4 -p /general/use_compositing -s false
```

---

## Firefox 충돌

user.js를 통한 가속 비활성화(6단계)

---

## 검은 화면

xf86-video-intel을 제거하십시오:

```bash
sudo xbps-remove xf86-video-intel
```

---

# 15. 전체 그래픽 시스템 점검

실행하다:

```bash
inxi -G
glxinfo | grep OpenGL
lsmod | grep i915
xfconf-query -c xfwm4 -p /general/use_compositing
```

---

# 16. Intel GPU 및 OpenGL 지원

| GPU | 오픈GL |
|----|---------|
| GMA 950 | 2.1 |
| GMA 3100 | 2.1 |
| X3100 | 2.1 |
| X4500 | 2.1 |
| HD 2000 | 3.3 |
| HD 3000 | 3.3 |
| HD4000 | 4.0 |
| UHD 620 | 4.6 |

---

# 17. 결론

구형 Intel GPU의 경우 다음을 사용하십시오.

- 드라이버 커널: i915
- 드라이버 Xorg: 모드 설정
- DRI2
- 작곡가 비활성화됨
- Firefox 가속이 비활성화되었습니다.

시스템은 안정적이고 기능적일 것입니다.
