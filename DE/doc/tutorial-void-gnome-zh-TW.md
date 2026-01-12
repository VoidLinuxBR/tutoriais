# 🐧 Void Linux + GNOME — 教程


>
> 本教程**不應以“root”身份運行**，除非**明確指出**。
>
> 所有命令均設計為由 **普通用戶** 執行，必要時使用“sudo”。
>
> 以“root”身份登錄運行整個教程：
> - 破壞權限邏輯
> - 使“sudo”配置等步驟無效
> - pode gerar erros silenciosos ou comportamentos inesperados
>
> 👉 **推薦**
> 如果您剛剛安裝系統並以“root”身份登錄：
>
> 1.創建普通用戶
> 2. 使用該用戶登錄
> 3.正常按照教程進行操作
>
> Unix/Linux 系統的經典規則：
>
> **`root` 是一個例外。普通用戶是規則。 **

---

## 0. 配置 sudo - 輪組 - 以避免詢問 root 密碼
```
sudo tee -a /etc/sudoers.d/g_wheel >/dev/null << EOF
%wheel ALL=(全部:全部) NOPASSWD: 全部
EOF

#所需權限
須藤 chmod 440 /etc/sudoers.d/g_wheel
```

## 1.更新系統
```
sudo xbps-install -Syu
```

## 2. 安裝完整的 GNOME（元包）
```
sudo xbps-install -y gnome \
侏儒圖標主題 \
論文\
網絡管理器小程序 \
擴展管理器 \
鸚鵡螺\
鸚鵡螺論文擴展 \

nautilus-gnome-terminal-擴展 \
gnome 終端 \
弧形主題 \
火狐瀏覽器\
firefox-i18n-pt-BR \
xarchiver \
gnome 磁盤實用程序 \
gparted \
gvfs\
p7zip \
解壓\
腦電圖\
noto-字體-表情符號 \
頂部
```

## 3.安裝GDM（官方顯示管理器）
```
sudo xbps-install -y gdm
```

## 4. 顯示驅動程序

### 英特爾
```
sudo xbps-install -y mesa-dri linux-firmware-intel
```

### 新 AMD (amdgpu)
```
sudo xbps-install -y mesa-dri xf86-video-amdgpu
```

###老AMD
```
sudo xbps-install -y mesa-dri xf86-video-ati
```

### Nvidia（開放驅動程序）
```
sudo xbps-install -y mesa-nouveau-dri
```

## 5.安裝PipeWire（現代虛空聲音）
```
sudo xbps-install -y \
管材\
電線工\
alsa-插件-pulseaudio \
alsa-pipewire \
libjack-pipewire \
脈衝音頻實用程序\
alsa-utils \
帕武控制
```

## 6. 集成 ALSA → PipeWire
```
sudo mkdir -p /etc/alsa/conf.d
sudo ln -sf /usr/share/alsa/alsa.conf.d/50-pipewire.conf /etc/alsa/conf.d
sudo ln -sf /usr/share/alsa/alsa.conf.d/99-pipewire-default.conf /etc/alsa/conf.d
```

## 7. 啟用 pipeline-pulse 服務器（PulseAudio 兼容）
```
sudo mkdir -p /etc/pipewire/pipewire.conf.d
sudo ln -sf /usr/share/examples/pipewire/20-pipewire-pulse.conf /etc/pipewire/pipewire.conf.d/
```

## 8. 在會話中啟用 PipeWire 自動啟動
```
mkdir -p ~/.config/自動啟動
ln -sf /usr/share/applications/pipewire.desktop ~/.config/autostart/
ln -sf /usr/share/applications/pipewire-pulse.desktop ~/.config/autostart/
ln -sf /usr/share/applications/wireplumber.desktop ~/.config/autostart/
```

## 9.（可選）為startx創建.xinitrc
```
貓 <<EOF > ~/.xinitrc
#!/bin/sh
setxkbmap -layout br -variant abnt2 &
執行 gnome 會話
EOF
```

## 10.configure timezone - 定義時區
```
sudo ln -sfv /usr/share/zoneinfo/America/Sao_Paulo /etc/localtime
```

## 11. 配置區域設置
```
須藤 -i -i -e 's/s/#\(en_.us.utf-8 utf-8\)/' -e 's/^#\pt_br.br.br.
```

## 12. Customize /etc/rc.conf.設置控制台的默認時區、鍵盤佈局和字體。根據需要進行更改。
```
sudo tee -a /etc/rc.conf >/dev/null << EOF
TIMEZONE="美國/聖保羅"
KEYMAP="br-abnt2"
FONT=Lat2-Terminus16
EOF
```

## 13. 自定義/etc/locale.conf。設置語言。根據需要進行更改。
```
sudo tee /etc/locale.conf >/dev/null << EOF
LANG=pt_BR.UTF-8
語言=pt_BR.UTF-8
LC_COLLATE=pt_BR.UTF-8
EOF
```

## 14. 重新配置
```
sudo xbps-重新配置-fa
```

## 15.激活強制服務（runit）
```
sudo ln -s /etc/sv/dbus /var/service/
sudo ln -s /etc/sv/elogind /var/service/
sudo ln -s /etc/sv/polkitd /var/service/
sudo ln -s /etc/sv/NetworkManager /var/service/
sudo ln -s /etc/sv/gdm /var/service/
```

## 最終確定
- 使用GDM → 系統直接在GNOME 中啟動。
- 沒有 GDM → 使用 `startx` （如果 `.xinitrc` 存在）。
