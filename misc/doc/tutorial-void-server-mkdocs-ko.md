# Void Linux에 Mkdocs 서버 설치

## 🎯 목표 - 빠르고 간단하며 프로젝트 중심의 정적 문서 웹 사이트 생성기인 MkDocs 서버를 업로드합니다. 간단한 Markdown 파일을 전문적이고 탐색이 가능한 문서 웹사이트로 바꿔줍니다. 구성은 단일 YAML 파일(mkdocs.yml)을 통해 수행되며 콘텐츠는 표준 Markdown으로 작성됩니다. 실시간 보기를 위한 내장 개발 서버를 제공하여 기술 문서, 사용자 매뉴얼 또는 지식 기반을 만드는 데 이상적입니다.

---

## XBPS를 통해 시스템 종속성(Python 및 pipx) 설치

```bash
sudo xbps-install -S python3 python3-pipx
```

## 🏠 Python 가상 환경에 mkdocs 패키지 설치

```bash
pipx install mkdocs
```

## 로컬 또는 전역적으로 운영 체제에 새 경로를 추가합니다.

## 현지의

```bash
pipx ensurepath
```

## 글로벌

```bash
sudo pipx ensurepath --global
```

## 위치는 사용자의 .bashrc에 표시됩니다.

```bash
# Created by `pipx` on 2025-11-27 14:07:54
export PATH="$PATH:/home/suporte/.local/bin"
```

## 운영 체제에 대한 사용자의 새 경로 유효성을 검사합니다.

```bash
source ~/.bashrc
```

## 패키지 설치 유효성 검사

```bash
mkdocs --version
```

## Python 가상 환경에 Material 테마 설치

```bash
pipx inject mkdocs mkdocs-material
```

## 주입은 사용자의 집에 있는 숨겨진 경로에 테마 패키지를 설치합니다.

```bash
/home/suporte/.local/bin/mkdocs
```

## 도구 사용 순서:

## 1. 새 프로젝트 만들기

## 🔧 새 문서 프로젝트를 시작하려면 프로젝트를 생성하려는 디렉터리로 이동하여 다음을 실행하세요.

```bash
mkdocs new Void_Artigos
```

## 그러면 기본 MkDocs 구조를 사용하여 Void_Artigos라는 새 디렉터리가 생성됩니다.

## 2. 머티리얼 테마 사용(선택 사항)

## 🧩 새 프로젝트를 생성한 경우 프로젝트 디렉터리(Void_Artigos/mkdocs.yml) 내에서 mkdocs.yml 구성 파일을 편집하고 Material 테마 구성을 추가하세요.

```bash
site_name: Void Artigos
nav:
    - Home: index.md
    - Sobre: about.md

theme:
  name: material # Adicione esta linha para usar o tema Material
```

## 3. 개발 서버 시작

## 문서를 편집하는 동안 로컬에서 보려면 프로젝트 디렉터리로 이동하여 개발 서버를 시작하세요.

```bash
cd void-Artigos
```

```bash
mkdocs serve
```

## 서버가 시작되고 브라우저의 일반적으로 http://127.0.0.1:8000.에서 문서에 액세스할 수 있습니다. MkDocs는 파일 변경 사항을 자동으로 모니터링하고 페이지를 다시 로드합니다.

## 내부 네트워크를 서비스하기 위해 서버의 IP와 포트를 제공합니다.

```bash
mkdocs serve 192.168.70.100:8000
```

## 내부 네트워크의 모든 브라우저에서 접근 가능

```bash
http://192.168.70.100:8000
```

## 4. 정적 문서 작성

## 문서를 게시할 준비가 되면 정적 파일을 빌드하세요.

```bash
mkdocs build
```

## 그러면 모든 웹 서버에서 문서를 호스팅하는 데 필요한 모든 HTML, CSS 및 JavaScript 파일을 포함하는 site/라는 디렉토리가 생성됩니다. 간단히 말해서, Void Linux를 사용하더라도 애플리케이션을 효과적으로 격리하는 pipx를 사용하므로 MkDocs 작업 흐름이 변경되지 않습니다.

---

🎯 그게 전부입니다!

👉 문의: zerolies@disroot.org
👉 https://t.me/z3r0l135
