# 教程 Distrobox no Void Linux

Distrobox 允許您在 Void Linux 中使用其他 Linux 發行版
不損害基本系統（主機）。
邏輯與往常一樣：**乾淨的系統，隔離的測試，零解決方法**。

---

## 這裡的建議是什麼

在 Void Linux 上安裝 distrobox，並使用容器安全地運行其他發行版。

這消除了以下風險：
- 打破系統依賴
- 使用其他發行版的軟件包污染主機
- 將虛空變成弗蘭肯斯坦

注意：本指南假設 Void Linux 已安裝。

---

## 首先：chililinux 存儲庫

官方 Void 存儲庫中不存在“distrobox”包。
但它是由 VoidLinuxBR Community 打包的，所以需要添加 chililinux 倉庫（巴西官方 Void 鏡像 - <https://xmirror.voidlinux.org/>).

**準確**執行以下命令：
```bash
sudo sh -c "{
  echo 'repository=https://repo-fastly.voidlinux.org/current'
  echo 'repository=https://void.chililinux.com/voidlinux/current'
} > /etc/xbps.d/00-repository-main.conf"
```

---

## 更新基礎系統

在安裝任何內容之前，請確保您的系統是最新的：

```bash
sudo xbps-install -Syu xbps
sudo xbps-install -Syu libssh2 xtools
sudo xbps-install -Suy
xcheckrestart
```
如果“xcheckrestart”指示重新啟動，則重新啟動。

---

## 安裝 Distrobox 和依賴項

現在，安裝必要的軟件包：

```bash
sudo xbps-install -Syf voidbr-distrobox podman docker crun
```

重要的：
安裝“crun”後，必須重新啟動系統：

```bash
sudo reboot
```

---

## 關於發行版兼容性

並非每個發行版都能在容器中正常運行。
選擇之前，先查閱官方列表：

辣椒_REF_0_辣椒

這樣可以避免浪費時間和頭痛。

---

## 創建第一個容器（Debian）

作為示例，將使用 Debian 測試。

```bash
distrobox create -Y --name debian --image docker.io/library/debian:testing
```

這裡發生了什麼：
- `distrobox create` 創建容器
- `-Y` 避免交互式問題
- `--name` 定義容器的名稱
- `--image` 定義基礎鏡像

要查看所有可用選項：

```bash
distrobox --help
```

---

## 進入容器

拉取鏡像後，進入容器：

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

您實際上是在另一個發行版中。

---

## 不進入容器執行命令

您還可以直接從主機運行命令。

示例：在 Debian 上安裝 Firefox，無需進入：

```bash
distrobox enter debian -- sudo apt install -y firefox-esr-l10n-pt-br
```

實用、快速、傳統。

---

## 將應用程序導出到主機系統

Distrobox 允許您從容器導出應用程序
到 VoidLinuxBR 圖形菜單。

示例：從 Debian 容器導出 Firefox：

```bash
distrobox enter debian -- distrobox-export --app firefox
```

該應用程序將出現在圖形環境菜單中
就好像他是本地人一樣。

---

## 更新所有容器

要一次更新所有容器，
不執行主機：

```bash
distrobox-upgrade --all -v
```

---

## 列出現有容器

查看所有創建的容器：

```bash
distrobox list
```

顯示名稱、狀態和使用的圖像。

---

## 停止容器

如果您只需要停止容器：

```bash
distrobox stop debian
```

---

## 移除容器

要刪除 Distrobox 容器：

```bash
distrobox rm debian
```

如果您還想刪除 Podman 映像：

```bash
podman rmi -f [IMAGE ID]
```

---

## 最終觀察結果

- 使用容器進行測試，而不是讓主機變得混亂
- 根據需要調整名稱和圖像
- 請始終查閱官方文檔：
辣椒_REF_0_辣椒
- 首先在虛擬機或實驗室機器上進行測試

Distrobox 是一款適合那些喜歡控制的人的工具，
絕緣且維護良好的系統。

---

## 📜 製作人員

創建者：Robson Nakane <theblizzard1983@hotmail.com>
社區：Void Linux 巴西 <https://github.com/voidlinuxbr>
發行版：Chili Linux <https://chililinux.com>
分佈：VoidBR <https://github.com/voidlinuxbr>

---

## ⚖️免責聲明（法律聲明）

本軟件/教程按“原樣”提供，絕對不提供任何保證
任何類型，無論是明示還是暗示，包括但不限於，
適銷性或特定用途適用性的保證。

使用本軟件的全部責任由用戶承擔。

作者或貢獻者在任何時候都不承擔任何責任
因使用而造成的任何損壞、數據丟失或系統故障
本計劃的。

---

版權所有 (C) 2026 羅布森·中根
