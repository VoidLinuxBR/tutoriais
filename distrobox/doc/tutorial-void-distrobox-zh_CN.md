# 教程 Distrobox no Void Linux

Distrobox 允许您在 Void Linux 中使用其他 Linux 发行版
不损害基本系统（主机）。
逻辑与往常一样：**干净的系统，隔离的测试，零解决方法**。

---

## 这里的建议是什么

在 Void Linux 上安装 distrobox，并使用容器安全地运行其他发行版。

这消除了以下风险：
- 打破系统依赖
- 使用其他发行版的软件包污染主机
- 将虚空变成弗兰肯斯坦

注意：本指南假设 Void Linux 已安装。

---

## 首先：chililinux 存储库

官方 Void 存储库中不存在“distrobox”包。
但它是由 VoidLinuxBR Community 打包的，所以需要添加 chililinux 仓库（巴西官方 Void 镜像 - <https://xmirror.voidlinux.org/>).

**准确**执行以下命令：
```bash
sudo sh -c "{
  echo 'repository=https://repo-fastly.voidlinux.org/current'
  echo 'repository=https://void.chililinux.com/voidlinux/current'
} > /etc/xbps.d/00-repository-main.conf"
```

---

## 更新基础系统

在安装任何内容之前，请确保您的系统是最新的：

```bash
sudo xbps-install -Syu xbps
sudo xbps-install -Syu libssh2 xtools
sudo xbps-install -Suy
xcheckrestart
```
如果“xcheckrestart”指示重新启动，则重新启动。

---

## 安装 Distrobox 和依赖项

现在，安装必要的软件包：

```bash
sudo xbps-install -Syf voidbr-distrobox podman docker crun
```

重要的：
安装“crun”后，必须重新启动系统：

```bash
sudo reboot
```

---

## 关于发行版兼容性

并非每个发行版都能在容器中正常运行。
选择之前，先查阅官方列表：

辣椒_REF_0_辣椒

这样可以避免浪费时间和头痛。

---

## 创建第一个容器（Debian）

作为示例，将使用 Debian 测试。

```bash
distrobox create -Y --name debian --image docker.io/library/debian:testing
```

这里发生了什么：
- `distrobox create` 创建容器
- `-Y` 避免交互式问题
- `--name` 定义容器的名称
- `--image` 定义基础镜像

要查看所有可用选项：

```bash
distrobox --help
```

---

## 进入容器

拉取镜像后，进入容器：

```bash
distrobox enter debian
```

在 Debian 中，使用是正常的：

```bash
sudo apt update
sudo apt upgrade
sudo apt autoremove
sudo apt install firefox
```

您实际上是在另一个发行版中。

---

## 不进入容器执行命令

您还可以直接从主机运行命令。

示例：在 Debian 上安装 Firefox，无需进入：

```bash
distrobox enter debian -- sudo apt install -y firefox-esr-l10n-pt-br
```

实用、快速、传统。

---

## 将应用程序导出到主机系统

Distrobox 允许您从容器导出应用程序
到 VoidLinuxBR 图形菜单。

示例：从 Debian 容器导出 Firefox：

```bash
distrobox enter debian -- distrobox-export --app firefox
```

该应用程序将出现在图形环境菜单中
就好像他是本地人一样。

---

## 更新所有容器

要一次更新所有容器，
不执行主机：

```bash
distrobox-upgrade --all -v
```

---

## 列出现有容器

查看所有创建的容器：

```bash
distrobox list
```

显示名称、状态和使用的图像。

---

## 停止容器

如果您只需要停止容器：

```bash
distrobox stop debian
```

---

## 移除容器

要删除 Distrobox 容器：

```bash
distrobox rm debian
```

如果您还想删除 Podman 映像：

```bash
podman rmi -f [IMAGE ID]
```

---

## 最终观察结果

- 使用容器进行测试，而不是让主机变得混乱
- 根据需要调整名称和图像
- 请始终查阅官方文档：
辣椒_REF_0_辣椒
- 首先在虚拟机或实验室机器上进行测试

Distrobox 是一款适合那些喜欢控制的人的工具，
绝缘且维护良好的系统。

---

## 📜 制作人员

创建者：Robson Nakane <theblizzard1983@hotmail.com>
社区：Void Linux 巴西 <https://github.com/voidlinuxbr>
发行版：Chili Linux <https://chililinux.com>
分布：VoidBR <https://github.com/voidlinuxbr>

---

## ⚖️免责声明（法律声明）

本软件/教程按“原样”提供，绝对不提供任何保证
任何类型，无论是明示还是暗示，包括但不限于，
适销性或特定用途适用性的保证。

使用本软件的全部责任由用户承担。

作者或贡献者在任何时候都不承担任何责任
因使用而造成的任何损坏、数据丢失或系统故障
本计划的。

---

版权所有 (C) 2026 罗布森·中根
