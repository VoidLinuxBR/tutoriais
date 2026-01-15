# Tailscale VPN sob Void Linux ;D

## 🎯 Objective - Upload a network interface connected to the Tailscale service VPN.

## ✅ Download and unzip the Tailscale client

```bash
cd /tmp
```

```bash
wget https://pkgs.tailscale.com/stable/tailscale_1.90.9_amd64.tgz
```

```bash
tar xvf tailscale_1.90.9_amd64.tgz
```

## ✅ Create runit service directory

```bash
sudo mkdir -p /etc/sv/tailscaled
```

## ✅ Create the run file

```bash
sudo tee /etc/sv/tailscaled/run << 'EOF'
#!/bin/sh
exec /usr/local/bin/tailscaled \
  --state=/var/lib/tailscale/tailscaled.state \
  2>&1
EOF
```

## ✅ Give execute permission:

```bash
sudo chmod +x /etc/sv/tailscaled/run
```

## ✅ (Optional, but recommended) Create log script

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

## ✅ Activate the service at boot

```bash
sudo ln -s /etc/sv/tailscaled /var/service/
```

## ✅ Start service now

```bash
sudo sv up tailscaled
```

```bash
sudo sv status tailscaled
```

## ✅ Connect Tailscale client

## Only after the service is running, run:

```bash
sudo tailscale up
```

## 🎉 Now Tailscale automatically goes up on boot

## If you want to confirm after restart:

```bash
sudo tailscale status
```
```bash
sudo sv status tailscaled
```

## ✅ IF you have at some point run the service and it has cached Tailscale's DNS, you can go back to using your own network DNS by running the command:

```bash
sudo tailscale up --accept-dns=false
```

## After that, this new configuration will be saved in the state, and in the next boots the DNS will no longer be modified.

---

🎯 THAT'S ALL FOLKS!

👉 Contact: zerolies@disroot.org
👉 https://t.me/z3r0l135


