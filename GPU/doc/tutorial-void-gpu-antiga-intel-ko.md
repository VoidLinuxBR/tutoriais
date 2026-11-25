# 완전한 튜토리얼: 리눅스에서 인텔 GPU (Void Linux, XFCE 및 오래된 하드웨어)

이 가이드는 특히 GMA 3100, GMA X3100, HD 2000/3000 등과 같은 오래된 하드웨어를 포함하여 Linux에서 Intel GPU를 식별, 설정 및 안정화하는 방법을 설명합니다.

> **주의:** 구형 인텔 GPU는 두 그룹으로 나뉘며, 설정이 달라집니다:
>
> - **A 그룹 – GMA (950, 3000, 3100, X3100, X4500):** 안정성 단계를 따릅니다 (DRI2, 컴포지터 및 파이어폭스 가속 비활성화). 하드웨어를 통한 유용한 비디오 디코딩은 지원하지 않습니다.
> - **B 그룹 – Intel HD (1세대 코어/아이언레이크, HD 2000/3000/샌디 브리지, HD 2500/4000/아이비 브리지, 하스웰):** 일반적으로 안정성 조정이 필요하지 않으며, **`i965` 드라이버를 사용하여 비디오 가속(VA-API)을 사용할 수 있고 사용해야 합니다** (7절 참조).
>
> 섹션 1을 사용하여 GPU가 어느 그룹에 속하는지 확인하세요.

---

# 1. Intel GPU 확인

실행:

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

# 2. 리눅스용 인텔 드라이버

모든 인텔 GPU는 다음을 사용합니다:

```
kernel driver: i915
```

당신은 독점 드라이버를 설치할 필요가 없습니다. 인텔에서는 **모든 드라이버가 자유롭습니다**.

## 2.1 i915, i965 및 iHD: 혼동하지 마세요

서로 다른 층들이고, 서로 대안이라고 생각하는 것이 흔합니다:

| 이름 | 무엇인가 | 어디서 오는가 |
|------|---------|----------|
| `i915` | **커널** 드라이버 (비디오/모드 설정) | 이미 커널에 포함되어 있음, 아무 것도 설치할 필요 없음 |
| `i965` | GPU용 **VA-API** 드라이버(비디오 디코딩, ~Coffee Lake까지 지원) | `libva-intel-driver` 패키지 |
| `iHD` | Broadwell(5세대) 이상용 **VA-API** 드라이버 | `intel-media-driver` 패키지 |
| `crocus` / `iris` | Mesa **OpenGL** 드라이버 (crocus: Gen4에서 Haswell까지; iris: Broadwell 이상) | `mesa-dri` 패키지 |

즉: HD 2000/3000을 사용하는 사람은 `i915` (커널) **와** `i965` (VA-API)를 동시에 사용합니다.

## 2.2 드라이버 Xorg

가능한 두 개의 Xorg 드라이버가 있습니다:

- modesetting → 권장
- xf86-video-intel → 대부분의 경우 구식

어떤 것이 사용 중인지 확인하세요:

```bash
inxi -G
```

보여주면:

```
driver: modesetting
```

맞습니다.

---

# 3. 구형 인텔 GPU용 (GMA 950, 3000, 3100, X3100, X4500)

이 GPU에는 제한이 있습니다:

- OpenGL 2.1 최대
- 불안정한 DRI3
- 작곡가가 멈춤을 유발함
- 현대 가속이 제대로 작동하지 않습니다

---

# 4. 권장 안정 구성

만들다:

```bash
sudo mkdir -p /etc/X11/xorg.conf.d
sudo nano /etc/X11/xorg.conf.d/20-intel.conf
```

내용:

```conf
Section "Device"
    Identifier "Intel Graphics"
    Driver "modesetting"
    Option "DRI" "2"
EndSection
```

---

# 5. XFCE 합성기 비활성화(구형 GPU에서는 필수)

실행:

```bash
xfconf-query -c xfwm4 -p /general/use_compositing -s false
```

확인:

```bash
xfconf-query -c xfwm4 -p /general/use_compositing
```

예상 결과:

```
false
```

---

# 6. 구형 Intel GPU용 Firefox 설정 (그룹 A – GMA만 해당)

> **GMA 950/3000/3100/X3100/X4500 전용**입니다. 만약 귀하의 GPU가 Intel HD(HD 2000/3000 이상)라면, **이 단계를 적용하지 마십시오**: 이 단계는 비디오 가속을 비활성화합니다. 바로 7장으로 이동하세요.

만들다:

```bash
nano ~/.mozilla/firefox/*.default-release/user.js
```

내용:

```js
user_pref("layers.acceleration.disabled", true);
user_pref("gfx.webrender.force-disabled", true);
user_pref("gfx.xrender.enabled", true);
user_pref("media.ffmpeg.vaapi.enabled", false);
```

이것은 멈춤을 방지합니다.

---

# 7. 인텔 HD용 비디오 가속(VA-API) (B 그룹)

VA-API를 사용하면 GPU가 비디오를 디코딩하고 CPU는 자유로워집니다. 이는 2세대/3세대 Core i3/i5 노트북으로 YouTube를 볼 때 큰 차이를 만듭니다.

## 7.1 각 GPU가 하드웨어에서 디코딩하는 것

| GPU | 가속화된 코덱 | VA-API 드라이버 |
|-----|----------------------|---------------|
| GMA 950/3000/3100/X3100 | 없음 | — |
| GMA X4500 | MPEG-2 | `i965` |
| HD 그래픽스 (아이언레이크, 1세대 코어) | MPEG-2, H.264, VC-1 | `i965` |
| HD 2000/3000 (샌디 브리지) | MPEG-2, H.264, VC-1 | `i965` |
| HD 2500/4000 (아이비 브리지) | MPEG-2, H.264, VC-1, JPEG | `i965` |
| HD 4200–5200 (Haswell) | MPEG-2, H.264, VC-1, JPEG | `i965` |
| 브로드웰 이상 | + 세대별 VP8/HEVC/VP9/AV1 | `iHD` |

**Haswell 이전의 GPU는 VP9 또는 AV1을 디코딩하지 않습니다.** YouTube는 기본적으로 VP9/AV1을 사용하므로, 이러한 경우 브라우저에 H.264를 강제로 사용하도록 하는 확장 프로그램을 설치하세요(예: *h264ify* 또는 *enhanced-h264ify*). 설치하지 않으면 비디오가 다시 CPU에 의해 디코딩됩니다.

## 7.2 패키지 설치

```bash
sudo xbps-install -S mesa-dri libva-intel-driver libva-utils intel-gpu-tools
```

사용자가 `video` 그룹에 있는지 확인하십시오(세션에서 나갔다가 다시 들어갑니다):

```bash
sudo usermod -aG video $USER
```

## 7.3 올바른 VA-API 드라이버 강제 설정

`~/.profile`에 추가하세요 (또는 `/etc/environment`에, `export`는 제외하고):

```bash
export LIBVA_DRIVER_NAME=i965
```

## 7.4 테스트

```bash
vainfo
```

기대 결과(요약):

```
vainfo: Driver version: Intel i965 driver for Intel(R) Sandybridge Mobile
      VAProfileH264Main               : VAEntrypointVLD
      VAProfileH264High               : VAEntrypointVLD
```

`VAEntrypointVLD` = 하드웨어 디코딩 가능.

## 7.5 파이어폭스

`about:config`에서:

```
media.ffmpeg.vaapi.enabled = true
```

최신 버전의 Firefox에서 디코딩이 자동으로 켜지지 않으면, 또한:

```
media.hardware-video-decoding.force-enabled = true
```

섹션 6에서 `user.js`를 적용했다면 제거하십시오. Firefox를 재시작한 후 `about:support` 섹션의 **미디어**에서 H264가 하드웨어 디코딩으로 표시되는지 확인하십시오.

## 7.6 GPU가 작동 중인지 확인

YouTube에서 비디오를 열고 (H.264 확장 기능이 활성화된 상태에서) 실행하세요:

```bash
sudo intel_gpu_top
```

영상이 재생되는 동안 **Video** 라인이 사용 중이라면, 디코딩이 GPU에서 이루어지고 있는 것입니다. CPU 사용량이 크게 줄어듭니다.

---

# 8. OpenGL 지원 확인

실행:

```bash
glxinfo | grep "OpenGL version"
```

예:

```
OpenGL version string: 2.1 Mesa 25.3.3
```

이것은 GMA 3100이 지원하는 최대치입니다.

---

# 9. 능동 가속도 확인

실행:

```bash
glxinfo | grep "direct rendering"
```

정확한 결과:

```
direct rendering: Yes
```

---

# 10. 로드된 드라이버 확인

실행:

```bash
lsmod | grep i915
```

결과:

```
i915
```

---

# 11. Xorg 상태 확인

실행:

```bash
grep -i driver /var/log/Xorg.0.log
```

정확한 결과:

```
modesetting
```

---

# 12. 너무 오래된 하드웨어에서는 xf86-video-intel을 사용하지 마십시오

설치되어 있다면 제거하십시오:

```bash
sudo xbps-remove xf86-video-intel
```

모드세팅 드라이버가 더 안정적입니다.

---

# 13. DRI 확인

실행:

```bash
glxinfo | grep DRI
```

구형 GPU의 경우, 다음을 사용하세요:

```
DRI2
```

---

# 14. 해상도 확인

실행:

```bash
xrandr
```

---

# 15. 일반적인 문제 및 해결

## XFCE에 들어갈 때 재부팅

합성기 비활성화:

```bash
xfconf-query -c xfwm4 -p /general/use_compositing -s false
```

---

## 파이어폭스가 멈췄어요

user.js를 통한 가속을 비활성화하세요(6단계). **GMA만** — Intel HD에서는 가속을 유지하는 것이 좋습니다.

---

## 많은 CPU를 사용하는 유튜브 동영상 (Intel HD)

- H.264를 강제하는 확장 기능이 활성화되어 있는지 확인하세요 (GPU는 VP9/AV1을 디코딩하지 않습니다).
- `vainfo`를 실행하고 `VAProfileH264...: VAEntrypointVLD`가 나타나는지 확인하세요.
- `LIBVA_DRIVER_NAME=i965`를 `echo $LIBVA_DRIVER_NAME`와 확인하세요.
- 섹션 6의 `user.js`가 적용되지 않았는지 확인하세요(이것은 VA-API를 비활성화합니다).

---

## 검은 화면

xf86-video-intel 제거:

```bash
sudo xbps-remove xf86-video-intel
```

---

# 16. 그래픽 시스템 전체 점검

실행:

```bash
inxi -G
glxinfo | grep OpenGL
lsmod | grep i915
xfconf-query -c xfwm4 -p /general/use_compositing
vainfo          # somente Intel HD (Grupo B)
```

---

# 17. 인텔 GPU 및 OpenGL 지원

| GPU | OpenGL |
|----|--------|
| GMA 950 | 2.1 |
| GMA 3100 | 2.1 |
| X3100 | 2.1 |
| X4500 | 2.1 |
| HD 2000 | 3.3 |
| HD 3000 | 3.3 |
| HD 4000 | 4.0 |
| UHD 620 | 4.6 |

---

# 18. 결론

**그룹 A – GMA (950, 3000, 3100, X3100, X4500):**

- 드라이버 커널: i915
- 드라이버 Xorg: 모드설정
- DRI2
- 비활성화된 컴포지터
- Firefox 가속 비활성화됨

**B 그룹 – 인텔 HD (아이언레이크, HD 2000/3000, HD 2500/4000, 하스웰):**

- 드라이버 커널: i915
- 드라이버 Xorg: 모드설정
- 드라이버 VA-API: i965 (`libva-intel-driver`) + `LIBVA_DRIVER_NAME=i965`
- Firefox에서 VA-API 활성화됨
- YouTube에서 H.264를 강제하는 확장 프로그램

시스템이 안정적이고 작동 가능할 것입니다.
