# 튜토리얼 Distrobox no Void Linux

Distrobox를 사용하면 Void Linux 내에서 다른 Linux 배포판을 사용할 수 있습니다
기본 시스템(호스트)을 손상시키지 않고.
논리는 평소와 같습니다: **깨끗한 시스템, 격리된 테스트, 제로 해결 방법**.

---

## 여기서 제안은 무엇입니까

Void Linux에 distrobox를 설치하고 컨테이너를 사용하여 다른 배포판을 안전하게 실행하세요.

이는 다음과 같은 위험을 제거합니다.
- 시스템 의존성 깨기
- 다른 배포판의 패키지로 호스트를 오염시킵니다.
- 공허를 프랑켄슈타인으로 바꾸다

참고: 이 가이드에서는 Void Linux가 이미 설치되어 있다고 가정합니다.

---

## 우선: chillilinux 저장소

`distrobox` 패키지는 공식 Void 저장소에 존재하지 않습니다.
하지만 VoidLinuxBR 커뮤니티에서 패키징했기 때문에 chillilinux 저장소(브라질의 공식 Void 미러 - <https://xmirror.voidlinux.org/>).)를 추가해야 합니다.

아래 명령을 **정확히** 실행하세요.
```bash
sudo sh -c "{
  echo 'repository=https://repo-fastly.voidlinux.org/current'
  echo 'repository=https://void.chililinux.com/voidlinux/current'
} > /etc/xbps.d/00-repository-main.conf"
```

---

## 기본 시스템 업데이트

무엇이든 설치하기 전에 시스템이 최신 상태인지 확인하세요.

```bash
sudo xbps-install -Syu xbps
sudo xbps-install -Syu libssh2 xtools
sudo xbps-install -Suy
xcheckrestart
```
'xcheckrestart'가 다시 시작을 나타내면 다시 시작하세요.

---

## Distrobox 및 종속성 설치

이제 필요한 패키지를 설치하십시오.

```bash
sudo xbps-install -Syf voidbr-distrobox podman docker crun
```

중요한:
`crun`을 설치한 후에는 시스템을 다시 시작해야 합니다.

```bash
sudo reboot
```

---

## 배포 호환성 정보

모든 배포판이 컨테이너에서 제대로 작동하는 것은 아닙니다.
선택하기 전에 공식 목록을 참조하세요.

https://distrobox.it/compatibility/#containers-distros

이렇게 하면 시간 낭비와 두통을 피할 수 있습니다.

---

## 첫 번째 컨테이너 만들기(Debian)

예를 들어 Debian Testing이 사용됩니다.

```bash
distrobox create -Y --name debian --image docker.io/library/debian:testing
```

여기서 무슨 일이 일어나고 있나요?
- `distrobox create`는 컨테이너를 생성합니다.
- `-Y`는 대화형 질문을 피합니다.
- `--name`은 컨테이너의 이름을 정의합니다.
- `--image`는 기본 이미지를 정의합니다.

사용 가능한 모든 옵션을 보려면:

```bash
distrobox --help
```

---

## 컨테이너에 들어가기

이미지를 가져온 후 컨테이너에 들어갑니다.

```bash
distrobox enter debian
```

데비안 내에서는 사용법이 정상입니다:

```bash
sudo apt update
sudo apt upgrade
sudo apt autoremove
sudo apt install firefox
```

당신은 말 그대로 다른 배포판 안에 있습니다.

---

## 컨테이너에 들어가지 않고 명령 실행

호스트에서 직접 명령을 실행할 수도 있습니다.

예: Firefox를 Debian에 들어가지 않고 설치하는 방법:

```bash
distrobox enter debian -- sudo apt install -y firefox-esr-l10n-pt-br
```

실용적이고 빠르며 전통적입니다.

---

## 애플리케이션을 호스트 시스템으로 내보내기

Distrobox를 사용하면 컨테이너에서 애플리케이션을 내보낼 수 있습니다.
VoidLinuxBR 그래픽 메뉴로 이동합니다.

예: Debian 컨테이너에서 Firefox 내보내기:

```bash
distrobox enter debian -- distrobox-export --app firefox
```

응용 프로그램이 그래픽 환경 메뉴에 나타납니다.
마치 그가 원주민인 것처럼.

---

## 모든 컨테이너 업데이트 중

모든 컨테이너를 한 번에 업데이트하려면
호스트를 실행하지 않음:

```bash
distrobox-upgrade --all -v
```

---

## 기존 컨테이너 나열

생성된 모든 컨테이너를 보려면 다음 안내를 따르세요.

```bash
distrobox list
```

사용된 이름, 상태, 이미지가 표시됩니다.

---

## 컨테이너 중지

컨테이너를 중지해야 하는 경우:

```bash
distrobox stop debian
```

---

## 컨테이너 제거

Distrobox 컨테이너를 제거하려면:

```bash
distrobox rm debian
```

Podman 이미지도 제거하려면 다음을 수행하십시오.

```bash
podman rmi -f [IMAGE ID]
```

---

## 최종 관찰

- 호스트를 복잡하게 만드는 것이 아니라 테스트용으로 컨테이너를 사용하세요.
- 필요에 따라 이름과 이미지를 조정하세요.
- 항상 공식 문서를 참조하세요.
https://distrobox.it
- VM 또는 랩 컴퓨터에서 먼저 테스트

Distrobox는 제어를 좋아하는 사람들을 위한 도구입니다.
단열 및 잘 관리된 시스템.

---

## 📜 크레딧

작성자: Robson Nakane <theblizzard1983@hotmail.com>
커뮤니티: Void Linux 브라질 <https://github.com/voidlinuxbr>
배포: 칠리 리눅스 <https://chililinux.com>
배포: VoidBR <https://github.com/voidlinuxbr>

---

## ⚖️ 면책조항(법적 고지)

이 소프트웨어/튜토리얼은 어떠한 보증도 없이 "있는 그대로" 제공됩니다.
다음을 포함하되 이에 국한되지 않는 모든 종류의 명시적이든 묵시적이든,
특정 목적에 대한 상품성 또는 적합성에 대한 보증.

이 소프트웨어의 사용에 대한 모든 책임은 사용자에게 있습니다.

작성자나 기여자는 어떠한 경우에도 책임을 지지 않습니다.
사용으로 인해 발생하는 모든 손상, 데이터 손실 또는 시스템 오류
이 프로그램의.

---

Copyright (C) 2026 롭슨 나카네
