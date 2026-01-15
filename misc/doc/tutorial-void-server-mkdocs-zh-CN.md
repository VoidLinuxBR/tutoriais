# 在 Void Linux 上安装 Mkdocs 服务器

## 🎯 目标 - 上传 MkDocs Server，这是一个快速、简单且以项目为中心的静态文档网站生成器。它将简单的 Markdown 文件变成一个专业的、完全可导航的文档网站。配置通过单个 YAML 文件 (mkdocs.yml) 完成，内容以标准 Markdown 编写。它非常适合创建技术文档、用户手册或知识库，并提供内置开发服务器以供实时查看。

---

## 通过 XBPS 安装系统依赖项（Python 和 pipx）

```bash
sudo xbps-install -S python3 python3-pipx
```

## 🏠 在Python虚拟环境中安装mkdocs包

```bash
pipx install mkdocs
```

## 在本地或全局添加操作系统的新路径

## 当地的

```bash
pipx ensurepath
```

## 全球的

```bash
sudo pipx ensurepath --global
```

## 位置将出现在用户的 .bashrc 中

```bash
# Created by `pipx` on 2025-11-27 14:07:54
export PATH="$PATH:/home/suporte/.local/bin"
```

## 验证用户到操作系统的新路径

```bash
source ~/.bashrc
```

## 验证包安装

```bash
mkdocs --version
```

## 在Python虚拟环境中安装Material主题

```bash
pipx inject mkdocs mkdocs-material
```

## 注入会将主题包安装在用户家中的隐藏路径中

```bash
/home/suporte/.local/bin/mkdocs
```

## 工具使用顺序：

## 1. 创建一个新项目

## 🔧 要启动新的文档项目，请导航到要创建项目的目录并运行：

```bash
mkdocs new Void_Artigos
```

## 这将创建一个名为 Void_Artigos 的新目录，其具有基本的 MkDocs 结构。

## 2.使用Material主题（可选）

## 🧩 如果您创建了一个新项目，请编辑项目目录中的 mkdocs.yml 配置文件 (Void_Artigos/mkdocs.yml) 并添加 Material 主题配置：

```bash
site_name: Void Artigos
nav:
    - Home: index.md
    - Sobre: about.md

theme:
  name: material # Adicione esta linha para usar o tema Material
```

## 3.启动开发服务器

## 要在编辑时在本地查看文档，请导航到项目目录并启动开发服务器：

```bash
cd void-Artigos
```

```bash
mkdocs serve
```

## 服务器将启动，您将能够在浏览器中访问文档，通常位于 http://127.0.0.1:8000.。 MkDocs 将自动监视文件的更改并重新加载页面。

## 为了服务内部网络，提供服务器的IP和端口

```bash
mkdocs serve 192.168.70.100:8000
```

## 可从内部网络上的任何浏览器访问

```bash
http://192.168.70.100:8000
```

## 4. 构建静态文档

## 当您的文档准备好发布时，构建静态文件：

```bash
mkdocs build
```

## 这将创建一个名为 site/ 的目录，其中包含在任何 Web 服务器上托管文档所需的所有 HTML、CSS 和 JavaScript 文件。简而言之，由于使用 pipx 有效隔离了应用程序，使用 Void Linux 并不会改变 MkDocs 工作流程。

---

🎯 这就是大家！

👉联系方式：zerolies@disroot.org
👉 https://t.me/z3r0l135
