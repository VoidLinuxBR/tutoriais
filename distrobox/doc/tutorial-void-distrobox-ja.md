# チュートリアル Distrobox の Void Linux なし

Distrobox を使用すると、Void Linux 内で他の Linux ディストリビューションを使用できます
基本システム (ホスト) を損なうことなく。
ロジックはいつもの通りです: **クリーンなシステム、分離されたテスト、ゼロの回避策**。

---

## ここでの提案は何ですか

Void Linux に distrobox をインストールし、コンテナーを使用して他のディストリビューションを安全に実行します。

これにより、次のようなリスクが排除されます。
- システムの依存関係を解消する
- 他のディストリビューションからのパッケージでホストを汚染する
- ヴォイドをフランケンシュタインに変える

注: このガイドは、Void Linux がすでにインストールされていることを前提としています。

---

## まず最初に: chililinux リポジトリ

「distrobox」パッケージは、公式の Void リポジトリには存在しません。
ただし、VoidLinuxBR コミュニティによってパッケージ化されているため、chililinux リポジトリ (ブラジルの公式 Void ミラー - <https://xmirror.voidlinux.org/>).) を追加する必要があります。

以下のコマンドを **正確に** 実行します。
```bash
sudo sh -c "{
  echo 'repository=https://repo-fastly.voidlinux.org/current'
  echo 'repository=https://void.chililinux.com/voidlinux/current'
} > /etc/xbps.d/00-repository-main.conf"
```

---

## 基本システムのアップデート

何かをインストールする前に、システムが最新であることを確認してください。

```bash
sudo xbps-install -Syu xbps
sudo xbps-install -Syu libssh2 xtools
sudo xbps-install -Suy
xcheckrestart
```
xcheckrestart が再起動を示している場合は再起動します。

---

## Distrobox と依存関係のインストール

次に、必要なパッケージをインストールします。

```bash
sudo xbps-install -Syf voidbr-distrobox podman docker crun
```

重要：
「crun」をインストールした後は、システムを再起動する必要があります。

```bash
sudo reboot
```

---

## ディストリビューションの互換性について

すべてのディストリビューションがコンテナーで適切に動作するわけではありません。
選択する前に、公式リストを参照してください。

https://distrobox.it/compatibility/#containers-distros

これにより、時間の無駄や頭痛の種を避けることができます。

---

## 最初のコンテナの作成 (Debian)

例として、Debian テストが使用されます。

```bash
distrobox create -Y --name debian --image docker.io/library/debian:testing
```

ここで何が起こっているのか:
- `distrobox create` はコンテナを作成します
- `-Y` は対話型の質問を回避します
- `--name` はコンテナの名前を定義します
- `--image` はベースイメージを定義します

利用可能なすべてのオプションを表示するには:

```bash
distrobox --help
```

---

## コンテナに入る

イメージを取得した後、コンテナーに入ります。

```bash
distrobox enter debian
```

Debian 内では通常の使用法です。

```bash
sudo apt update
sudo apt upgrade
sudo apt autoremove
sudo apt install firefox
```

文字通り、別のディストリビューション内にいます。

---

## コンテナに入らずにコマンドを実行する

ホストから直接コマンドを実行することもできます。

例: Firefox を Debian にインストールせずにインストールする:

```bash
distrobox enter debian -- sudo apt install -y firefox-esr-l10n-pt-br
```

実用的、高速、そして伝統的。

---

## アプリケーションをホスト システムにエクスポートする

Distrobox を使用すると、コンテナからアプリケーションをエクスポートできます
VoidLinuxBR グラフィカル メニューに移動します。

例: Debian コンテナから Firefox をエクスポートします。

```bash
distrobox enter debian -- distrobox-export --app firefox
```

アプリケーションがグラフィカル環境メニューに表示されます
まるで彼がネイティブであるかのように。

---

## すべてのコンテナを更新する

すべてのコンテナを一度に更新するには、
ホストを実行しません:

```bash
distrobox-upgrade --all -v
```

---

## 既存のコンテナのリスト表示

作成されたすべてのコンテナを表示するには:

```bash
distrobox list
```

名前、ステータス、使用画像が表示されます。

---

## コンテナの停止

コンテナを停止するだけの場合は、次のようにします。

```bash
distrobox stop debian
```

---

## コンテナの削除

Distrobox コンテナを削除するには:

```bash
distrobox rm debian
```

Podman イメージも削除したい場合:

```bash
podman rmi -f [IMAGE ID]
```

---

## 最終観察

- ホストを混乱させるためではなく、テストのためにコンテナを使用する
- 必要に応じて名前と画像を調整します
- 必ず公式ドキュメントを参照してください。
https://distrobox.it
- 最初に VM またはラボ マシンでテストします

Distrobox はコントロールが好きな人のためのツールです。
断熱性とメンテナンスの行き届いたシステム。

---

## 📜 クレジット

作成者: ロブソン中根 <theblizzard1983@hotmail.com>
コミュニティ: Void Linux ブラジル <https://github.com/voidlinuxbr>
ディストリビューション: Chili Linux <https://chililinux.com>
配布: VoidBR <https://github.com/voidlinuxbr>

---

## ⚖️免責事項（法的通知）

このソフトウェア/チュートリアルは「現状のまま」提供され、一切の保証はありません
明示的か黙示的かにかかわらず、以下を含みますがこれらに限定されない、あらゆる種類の
商品性または特定の目的への適合性の保証。

このソフトウェアの使用はユーザーの全責任となります。

著者または寄稿者はいかなる場合も責任を負わないものとします。
使用に起因するあらゆる損害、データの損失、またはシステム障害
このプログラムの。

---

Copyright (C) 2026 ロブソン中根
