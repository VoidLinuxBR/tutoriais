# PipeWire + WirePlumber の Void Linux (2025)
実際の Void Linux パッケージのみに基づいた、機能的でわかりやすいガイド。

公式ドキュメント:
https://docs.voidlinux.org/config/media/pipewire.html

---

# 1. 正しいパッケージをインストールする

```
sudo xbps-install -S \
  pipewire \
  wireplumber \
  alsa-pipewire \
  alsa-lib \
  alsa-utils \
  alsa-plugins \
  alsa-firmware \
  pavucontrol \
  pulsemixer
```

これらは、Void の PipeWire に**唯一必要なパッケージ**です。

---

# 2. 構成

## 2.1 ALSA → PipeWire の統合
```
sudo mkdir -p /etc/alsa/conf.d
sudo ln -sf /usr/share/alsa/alsa.conf.d/50-pipewire.conf /etc/alsa/conf.d
sudo ln -sf /usr/share/alsa/alsa.conf.d/99-pipewire-default.conf /etc/alsa/conf.d
mkdir -p ~/.config/pipewire/pipewire.conf.d
mkdir -p ~/.config/pipewire/pipewire-pulse.conf.d
```

## 2.2 Pipewire-pulse サーバーを有効にする (PulseAudio compat)
```
sudo mkdir -p /etc/pipewire/pipewire.conf.d
sudo ln -sf /usr/share/examples/pipewire/20-pipewire-pulse.conf /etc/pipewire/pipewire.conf.d/
```

## 2.3 セッションで PipeWire 自動起動を有効にする
```
mkdir -p ~/.config/autostart
ln -sf /usr/share/applications/pipewire.desktop ~/.config/autostart/
ln -sf /usr/share/applications/pipewire-pulse.desktop ~/.config/autostart/
ln -sf /usr/share/applications/wireplumber.desktop ~/.config/autostart/
```
---

# 3. DE/WMによる自動起動

## 3.1 デスクトップ環境 (単独で実行)
これらの DE は PipeWire を自動的に開始します。
**何もすることはありません。**

- XFCE
- LXQt
- LXDE
- KDE プラズマ (X11 および Wayland)
- GNOME (X11 および Wayland)
- シナモン
- 死
- 啓発

PipeWire は介入なしで DBus 経由で起動します。

---

## 3.2 ウィンドウマネージャー (手動で起動する必要があります)

### i3 — `~/.xinitrc`

```
export XDG_RUNTIME_DIR="/run/user/$(id -u)"

dbus-run-session -- sh -c '
  pipewire &
  wireplumber &
  pipewire-pulse &
  exec i3
'
```

### bspwm — `~/.xinitrc`

```
export XDG_RUNTIME_DIR="/run/user/$(id -u)"

dbus-run-session -- sh -c '
  pipewire &
  wireplumber &
  pipewire-pulse &
  exec bspwm
'
```

### dwm — `~/.xinitrc`

```
export XDG_RUNTIME_DIR="/run/user/$(id -u)"

dbus-run-session -- sh -c '
  pipewire &
  wireplumber &
  pipewire-pulse &
  exec dwm
'
```

### オープンボックス — `~/.xinitrc`

```
export XDG_RUNTIME_DIR="/run/user/$(id -u)"

dbus-run-session -- sh -c '
  pipewire &
  wireplumber &
  pipewire-pulse &
  exec openbox-session
'
```

### Fluxbox / IceWM / JWM — `~/.xinitrc`

```
export XDG_RUNTIME_DIR="/run/user/$(id -u)"

dbus-run-session -- sh -c '
  pipewire &
  wireplumber &
  pipewire-pulse &
  exec fluxbox
'
```

---

## 3.3 Wayland WM

### Sway — `~/.config/sway/config`

```
exec pipewire
exec wireplumber
exec pipewire-pulse
```

### ウェイファイア

```
exec pipewire
exec wireplumber
exec pipewire-pulse
```

### ハイプラランド

```
exec-once = pipewire
exec-once = wireplumber
exec-once = pipewire-pulse
```

---

# 4. 必要なグループ (elogind がなければ無効)

```
sudo usermod -aG audio,video $USER
```

ログアウト/ログイン。

---

# 5. 一般テスト

パイプワイヤー:
```
ps aux | grep pipewire
```

ワイヤー配管工:
```
ps aux | grep wireplumber
```

PulseAudio 互換性:
```
pactl info | grep "Server Name"
```

期待される：
```
Server Name: PulseAudio (on PipeWire 0.3.x)
```

縫い目:
```
speaker-test -t wav -c 2
```

---

# 6. トラブルシューティング

古い PulseAudio 構成を削除します。
```
rm -rf ~/.config/pulse ~/.pulse
```

DBus を確認します。
```
ps aux | grep dbus
```

PipeWire を手動で開始します。
```
dbus-run-session -- pipewire
```

PulseAudio について問題を抱えているアプリケーション:
```
pipewire-pulse &
```

---

# 7. 簡単な概要

インストール：
```
sudo xbps-install pipewire wireplumber alsa-pipewire alsa-utils alsa-plugins alsa-lib alsa-firmware
```

自動的にサポートされる DE:
- XFCE、LXQt、LXDE、GNOME、KDE、シナモン、MATE、啓発

構成が必要な WM:
- i3、bspwm、dwm、Openbox、Fluxbox、sway、wayfire、hyprland

PipeWire はセッションごとに動作します (runit は使用しません)。

---

# 終わり

