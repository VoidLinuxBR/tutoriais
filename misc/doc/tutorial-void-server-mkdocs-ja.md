# Void Linux への Mkdocs サーバーのインストール

## 🎯 目的 - 高速かつシンプルでプロジェクトに焦点を当てた静的ドキュメント Web サイト ジェネレーターである MkDocs サーバーをアップロードします。単純な Markdown ファイルを、完全にナビゲート可能なプロフェッショナルなドキュメント Web サイトに変換します。構成は単一の YAML ファイル (mkdocs.yml) を通じて行われ、コンテンツは標準の Markdown で記述されます。技術文書、ユーザーマニュアル、ナレッジベースの作成に最適で、リアルタイム表示用の内蔵開発サーバーを提供します。

---

## XBPS 経由でシステムの依存関係 (Python および pipx) をインストールする

```bash
sudo xbps-install -S python3 python3-pipx
```

## 🏠 Python 仮想環境に mkdocs パッケージをインストールする

```bash
pipx install mkdocs
```

## オペレーティング システムへの新しいパスをローカルまたはグローバルに追加します。

## 地元

```bash
pipx ensurepath
```

## グローバル

```bash
sudo pipx ensurepath --global
```

## 場所はユーザーの .bashrc に表示されます

```bash
# Created by `pipx` on 2025-11-27 14:07:54
export PATH="$PATH:/home/suporte/.local/bin"
```

## ユーザーのオペレーティング システムへの新しいパスを検証します。

```bash
source ~/.bashrc
```

## パッケージのインストールを検証する

```bash
mkdocs --version
```

## Python 仮想環境へのマテリアル テーマのインストール

```bash
pipx inject mkdocs mkdocs-material
```

## インジェクションにより、テーマ パッケージがユーザーのホームの隠しパスにインストールされます。

```bash
/home/suporte/.local/bin/mkdocs
```

## ツールの使用手順:

## 1. 新しいプロジェクトを作成する

## 🔧 新しいドキュメント プロジェクトを開始するには、プロジェクトを作成するディレクトリに移動して、次のコマンドを実行します。

```bash
mkdocs new Void_Artigos
```

## これにより、基本的な MkDocs 構造を持つ Void_Artigos という新しいディレクトリが作成されます。

## 2. マテリアル テーマを使用する (オプション)

## 🧩 新しいプロジェクトを作成した場合は、プロジェクト ディレクトリ内の mkdocs.yml 構成ファイル (Void_Artigos/mkdocs.yml) を編集し、マテリアル テーマの構成を追加します。

```bash
site_name: Void Artigos
nav:
    - Home: index.md
    - Sobre: about.md

theme:
  name: material # Adicione esta linha para usar o tema Material
```

## 3. 開発サーバーを起動します

## 編集中にドキュメントをローカルで表示するには、プロジェクト ディレクトリに移動し、開発サーバーを起動します。

```bash
cd void-Artigos
```

```bash
mkdocs serve
```

## サーバーが起動し、ブラウザでドキュメント (通常は http://127.0.0.1:8000.) にアクセスできるようになります。 MkDocs はファイルへの変更を自動的に監視し、ページをリロードします。

## 内部ネットワークにサービスを提供するには、サーバーの IP とポートを指定します。

```bash
mkdocs serve 192.168.70.100:8000
```

## 内部ネットワーク上のどのブラウザからでもアクセス可能

```bash
http://192.168.70.100:8000
```

## 4. 静的ドキュメントの構築

## ドキュメントを公開する準備ができたら、静的ファイルを構築します。

```bash
mkdocs build
```

## これにより、Web サーバー上でドキュメントをホストするために必要なすべての HTML、CSS、および JavaScript ファイルを含む site/ というディレクトリが作成されます。つまり、アプリケーションを効果的に分離する pipx を使用するため、Void Linux を使用しても MkDocs ワークフローは変わりません。

---

🎯 以上です!

👉連絡先: zerolies@disroot.org
👉 チリ_REF_0_チリ
