# Tailscale VPN sob Void Linux ;D

## 🎯 目標 - 上傳連接到 Tailscale 服務 VPN 的網絡接口。

## ✅ 下載並解壓 Tailscale 客戶端

```bash
cd /tmp
```

```bash
wget https://pkgs.tailscale.com/stable/tailscale_1.90.9_amd64.tgz
```

```bash
tar xvf tailscale_1.90.9_amd64.tgz
```

## ✅ 創建runit服務目錄

```bash
sudo mkdir -p /etc/sv/tailscaled
```

## ✅ 創建運行文件

```bash
sudo tee /etc/sv/tailscaled/run << 'EOF'
#!/bin/sh
exec /usr/local/bin/tailscaled \
  --state=/var/lib/tailscale/tailscaled.state \
  2>&1
EOF
```

## ✅ 授予執行權限：

```bash
sudo chmod +x /etc/sv/tailscaled/run
```

## ✅ （可選，但推薦）創建日誌腳本

```bash
sudo mkdir -p /etc/sv/tailscaled/log
```
```bash
sudo tee /etc/sv/tailscaled/log/run << 'EOF'
#!/bin/sh
exec svlogd -tt /var/log/tailscaled
EOF
```

```bash
sudo chmod +x /etc/sv/tailscaled/log/run
```

## ✅ 開機時激活服務

```bash
sudo ln -s /etc/sv/tailscaled /var/service/
```

## ✅ 立即開始服務

```bash
sudo sv up tailscaled
```

```bash
sudo sv status tailscaled
```

## ✅ 連接 Tailscale 客戶端

## 僅在服務運行後，運行：

```bash
sudo tailscale up
```

## 🎉 現在 Tailscale 在啟動時自動上升

## 如果要重啟後確認：

```bash
sudo tailscale status
```
```bash
sudo sv status tailscaled
```

## ✅ 如果您在某個時刻運行該服務並且它已緩存 Tailscale 的 DNS，您可以通過運行以下命令返回使用您自己的網絡 DNS：

```bash
sudo tailscale up --accept-dns=false
```

## 之後，這個新的配置將被保存在狀態中，下次啟動時DNS將不再被修改。

---

🎯 這就是大家！

👉聯繫方式：zerolies@disroot.org
👉 https://t.me/z3r0l135


