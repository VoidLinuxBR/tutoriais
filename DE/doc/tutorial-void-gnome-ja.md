# 🐧 Linux + GNOME を無効にする — チュートリアル

> ⚠️ **重要 — 始める前にお読みください**
>
> このチュートリアルは、**明示的に示されている場合を除き、**「root」として実行しないでください**。
>
> すべてのコマンドは、必要に応じて `sudo` を使用して、**一般ユーザー** によって実行されるように設計されています。
>
> 「root」としてログインしてチュートリアル全体を実行します。
> - 権限ロジックを壊す
> - `sudo` 設定などの手順を無効にします
> - サイレントエラーまたは予期しない動作が発生する可能性があります
>
> 👉 **推奨事項**
> システムをインストールしたばかりで、「root」としてログインしている場合:
>
> 1. 共通ユーザーを作成する
> 2. このユーザーでログインします
> 3. 通常通りチュートリアルに従います
>
> Unix/Linux システムの古典的なルール:
>
> **`root` は例外です。一般ユーザーがルールです。**

---

## 0. root パスワードの要求を避けるために sudo - Wheel グループ - を設定します。
```
sudo tee -a /etc/sudoers.d/g_wheel >/dev/null << EOF
%wヒール ALL=(ALL:ALL) NOPASSWD: ALL
終了後

#必要な権限
sudo chmod 440 /etc/sudoers.d/g_wheel
```

## 1. システムをアップデートする
```
sudo xbps-install -Syu
```

## 2. 完全な GNOME (メタパッケージ) をインストールする
```
sudo xbps-install -y gnome \
gnome-アイコン-テーマ \
論文 \
ネットワークマネージャーアプレット\
拡張機能マネージャー \
オウムガイ\
nautilus-papers-extension \
nautilus-gnome-console-extension \
nautilus-gnome-ターミナル拡張機能 \
gnome ターミナル \
アークテーマ \
Firefox \
Firefox-i18n-pt-BR \
xアーカイブ\
gnome-ディスク-ユーティリティ \
別れた\
gvfs\
p7zip \
解凍\
エオグ\
noto-fonts-emoji \
hトップ
```

## 3. GDM (公式ディスプレイマネージャー) をインストールします。
```
sudo xbps-install -y gdm
```

## 4. ディスプレイドライバー

### インテル
```
sudo xbps-install -y mesa-dri linux-firmware-intel
```

### 新しい AMD (amdgpu)
```
sudo xbps-install -y mesa-dri xf86-video-amdgpu
```

### 古い AMD
```
sudo xbps-install -y mesa-dri xf86-video-ati
```

### Nvidia (オープンドライバー)
```
sudo xbps-install -y mesa-nouveau-dri
```

## 5. PipeWire (Modern Void Sound) をインストールする
```
sudo xbps-install -y \
パイプワイヤー \
針金配管工 \
alsa-plugins-pulseaudio \
アルサパイプワイヤー \
libjack-pipewire \
パルスオーディオユーティリティ \
alsa-utils \
パブコントロール
```

## 6. ALSA → PipeWire の統合
```
sudo mkdir -p /etc/alsa/conf.d
sudo ln -sf /usr/share/alsa/alsa.conf.d/50-pipewire.conf /etc/alsa/conf.d
sudo ln -sf /usr/share/alsa/alsa.conf.d/99-pipewire-default.conf /etc/alsa/conf.d
```

## 7. Pipewire-pulse サーバーを有効にする (PulseAudio compat)
```
sudo mkdir -p /etc/pipewire/pipewire.conf.d
sudo ln -sf /usr/share/examples/pipewire/20-pipewire-pulse.conf /etc/pipewire/pipewire.conf.d/
```

## 8. セッションで PipeWire の自動起動を有効にする
```
mkdir -p ~/.config/autostart
ln -sf /usr/share/applications/pipewire.desktop ~/.config/autostart/
ln -sf /usr/share/applications/pipewire-pulse.desktop ~/.config/autostart/
ln -sf /usr/share/applications/wireplumber.desktop ~/.config/autostart/
```

## 9. (オプション) startx の .xinitrc を作成します。
```
cat <<EOF > ~/.xinitrc
#!/bin/sh
setxkbmap -layout br -variant abnt2 &
gnomeセッションの実行
終了後
```

## 10. タイムゾーンの構成 - タイムゾーンを定義します
```
sudo ln -sfv /usr/share/zoneinfo/America/Sao_Paulo /etc/localtime
```

## 11. ロケールを設定する
```
Sudo -i -i -e 's/s/#\(en_.us.utf-8 utf-8\)/' -e 's/^#\pt_br.br.br。
```

## 12. /etc/rc.conf をカスタマイズします。本体のデフォルトのタイムゾーン、キーボードレイアウト、フォントを設定します。必要に応じて変更します。
```
sudo tee -a /etc/rc.conf >/dev/null << EOF
TIMEZONE="アメリカ/サンパウロ"
KEYMAP="br-abnt2"
FONT=Lat2-Terminus16
終了後
```

## 13. /etc/locale.conf をカスタマイズします。言語を設定します。必要に応じて変更します。
```
sudo tee /etc/locale.conf >/dev/null << EOF
LANG=pt_BR.UTF-8
LANGUAGE=pt_BR.UTF-8
LC_COLLATE=pt_BR.UTF-8
終了後
```

## 14. 再構成
```
sudo xbps-reconfigure -fa
```

## 15. 必須サービス (runit) をアクティブ化します。
```
sudo ln -s /etc/sv/dbus /var/service/
sudo ln -s /etc/sv/elogind /var/service/
sudo ln -s /etc/sv/polkitd /var/service/
sudo ln -s /etc/sv/NetworkManager /var/service/
sudo ln -s /etc/sv/gdm /var/service/
```

## ファイナライズ
- GDM を使用する → システムは GNOME で直接起動します。
- GDM なし → `startx` を使用します (`.xinitrc` が存在する場合)。
