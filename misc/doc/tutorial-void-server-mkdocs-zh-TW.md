# 在 Void Linux 上安裝 Mkdocs 服務器

## 🎯 目標 - 上傳 MkDocs Server，這是一個快速、簡單且以項目為中心的靜態文檔網站生成器。它將簡單的 Markdown 文件變成一個專業的、完全可導航的文檔網站。配置通過單個 YAML 文件 (mkdocs.yml) 完成，內容以標準 Markdown 編寫。它非常適合創建技術文檔、用戶手冊或知識庫，並提供內置開發服務器以供實時查看。

---

## 通過 XBPS 安裝系統依賴項（Python 和 pipx）

```bash
sudo xbps-install -S python3 python3-pipx
```

## 🏠 在Python虛擬環境中安裝mkdocs包

```bash
pipx install mkdocs
```

## 在本地或全局添加操作系統的新路徑

## 當地的

```bash
pipx ensurepath
```

## 全球的

```bash
sudo pipx ensurepath --global
```

## 位置將出現在用戶的 .bashrc 中

```bash
# Created by `pipx` on 2025-11-27 14:07:54
export PATH="$PATH:/home/suporte/.local/bin"
```

## 驗證用戶到操作系統的新路徑

```bash
source ~/.bashrc
```

## 驗證包安裝

```bash
mkdocs --version
```

## 在Python虛擬環境中安裝Material主題

```bash
pipx inject mkdocs mkdocs-material
```

## 注入會將主題包安裝在用戶家中的隱藏路徑中

```bash
/home/suporte/.local/bin/mkdocs
```

## 工具使用順序：

## 1. 創建一個新項目

## 🔧 要啟動新的文檔項目，請導航到要創建項目的目錄並運行：

```bash
mkdocs new Void_Artigos
```

## 這將創建一個名為 Void_Artigos 的新目錄，其具有基本的 MkDocs 結構。

## 2.使用Material主題（可選）

## 🧩 如果您創建了一個新項目，請編輯項目目錄中的 mkdocs.yml 配置文件 (Void_Artigos/mkdocs.yml) 並添加 Material 主題配置：

```bash
site_name: Void Artigos
nav:
    - Home: index.md
    - Sobre: about.md

theme:
  name: material # Adicione esta linha para usar o tema Material
```

## 3.啟動開發服務器

## 要在編輯時在本地查看文檔，請導航到項目目錄並啟動開發服務器：

```bash
cd void-Artigos
```

```bash
mkdocs serve
```

## 服務器將啟動，您將能夠在瀏覽器中訪問文檔，通常位於 http://127.0.0.1:8000.。 MkDocs 將自動監視文件的更改並重新加載頁面。

## 為了服務內部網絡，提供服務器的IP和端口

```bash
mkdocs serve 192.168.70.100:8000
```

## 可從內部網絡上的任何瀏覽器訪問

```bash
http://192.168.70.100:8000
```

## 4. 構建靜態文檔

## 當您的文檔準備好發佈時，構建靜態文件：

```bash
mkdocs build
```

## 這將創建一個名為 site/ 的目錄，其中包含在任何 Web 服務器上託管文檔所需的所有 HTML、CSS 和 JavaScript 文件。簡而言之，由於使用 pipx 有效隔離了應用程序，使用 Void Linux 並不會改變 MkDocs 工作流程。

---

🎯 這就是大家！

👉聯繫方式：zerolies@disroot.org
👉 https://t.me/z3r0l135
