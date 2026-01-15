# Tailscale VPN schluchzt Void Linux ;D

## 🎯 Ziel – Hochladen einer Netzwerkschnittstelle, die mit dem Tailscale-Dienst-VPN verbunden ist.

## ✅ Laden Sie den Tailscale-Client herunter und entpacken Sie ihn

```bash
cd /tmp
```

```bash
wget https://pkgs.tailscale.com/stable/tailscale_1.90.9_amd64.tgz
```

```bash
tar xvf tailscale_1.90.9_amd64.tgz
```

## ✅ Erstellen Sie ein Runit-Dienstverzeichnis

```bash
sudo mkdir -p /etc/sv/tailscaled
```

## ✅ Erstellen Sie die Laufdatei

```bash
sudo tee /etc/sv/tailscaled/run << 'EOF'
#!/bin/sh
exec /usr/local/bin/tailscaled \
  --state=/var/lib/tailscale/tailscaled.state \
  2>&1
EOF
```

## ✅ Ausführungsberechtigung erteilen:

```bash
sudo chmod +x /etc/sv/tailscaled/run
```

## ✅ (Optional, aber empfohlen) Erstellen Sie ein Protokollskript

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

## ✅ Aktivieren Sie den Dienst beim Booten

```bash
sudo ln -s /etc/sv/tailscaled /var/service/
```

## ✅ Jetzt Service starten

```bash
sudo sv up tailscaled
```

```bash
sudo sv status tailscaled
```

## ✅ Tailscale-Client verbinden

## Führen Sie Folgendes aus, nachdem der Dienst ausgeführt wurde:

```bash
sudo tailscale up
```

## 🎉 Jetzt wird Tailscale beim Booten automatisch hochgefahren

## Wenn Sie nach dem Neustart bestätigen möchten:

```bash
sudo tailscale status
```
```bash
sudo sv status tailscaled
```

## ✅ WENN Sie den Dienst irgendwann ausgeführt haben und er das DNS von Tailscale zwischengespeichert hat, können Sie wieder Ihr eigenes Netzwerk-DNS verwenden, indem Sie den folgenden Befehl ausführen:

```bash
sudo tailscale up --accept-dns=false
```

## Danach wird diese neue Konfiguration im Status gespeichert und bei den nächsten Starts wird der DNS nicht mehr geändert.

---

🎯 DAS IST ALLES, LEUTE!

👉 Kontakt: zerolies@disroot.org
👉 https://t.me/z3r0l135


