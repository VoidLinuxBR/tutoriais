# Tailscale VPN sob Void Linux ;D

## 🎯 目标 - 上传连接到 Tailscale 服务 VPN 的网络接口。

## ✅ 下载并解压 Tailscale 客户端

```bash
cd /tmp
```

```bash
wget https://pkgs.tailscale.com/stable/tailscale_1.90.9_amd64.tgz
```

```bash
tar xvf tailscale_1.90.9_amd64.tgz
```

## ✅ 创建runit服务目录

```bash
sudo mkdir -p /etc/sv/tailscaled
```

## ✅ 创建运行文件

```bash
sudo tee /etc/sv/tailscaled/run << 'EOF'
#!/bin/sh
exec /usr/local/bin/tailscaled \
  --state=/var/lib/tailscale/tailscaled.state \
  2>&1
EOF
```

## ✅ 授予执行权限：

```bash
sudo chmod +x /etc/sv/tailscaled/run
```

## ✅ （可选，但推荐）创建日志脚本

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

## ✅ 开机时激活服务

```bash
sudo ln -s /etc/sv/tailscaled /var/service/
```

## ✅ 立即开始服务

```bash
sudo sv up tailscaled
```

```bash
sudo sv status tailscaled
```

## ✅ 连接 Tailscale 客户端

## 仅在服务运行后，运行：

```bash
sudo tailscale up
```

## 🎉 现在 Tailscale 在启动时自动上升

## 如果要重启后确认：

```bash
sudo tailscale status
```
```bash
sudo sv status tailscaled
```

## ✅ 如果您在某个时刻运行该服务并且它已缓存 Tailscale 的 DNS，您可以通过运行以下命令返回使用您自己的网络 DNS：

```bash
sudo tailscale up --accept-dns=false
```

## 之后，这个新的配置将被保存在状态中，下次启动时DNS将不再被修改。

---

🎯 这就是大家！

👉联系方式：zerolies@disroot.org
👉 https://t.me/z3r0l135


