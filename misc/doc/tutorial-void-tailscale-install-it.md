# Tailscale VPN sob Void Linux ;D

## 🎯 Obiettivo - Carica un'interfaccia di rete connessa alla VPN del servizio Tailscale.

## ✅ Scarica e decomprimi il client Tailscale

```bash
cd /tmp
```

```bash
wget https://pkgs.tailscale.com/stable/tailscale_1.90.9_amd64.tgz
```

```bash
tar xvf tailscale_1.90.9_amd64.tgz
```

## ✅ Crea la directory del servizio runit

```bash
sudo mkdir -p /etc/sv/tailscaled
```

## ✅ Crea il file della corsa

```bash
sudo tee /etc/sv/tailscaled/run << 'EOF'
#!/bin/sh
exec /usr/local/bin/tailscaled \
  --state=/var/lib/tailscale/tailscaled.state \
  2>&1
EOF
```

## ✅ Dai il permesso di esecuzione:

```bash
sudo chmod +x /etc/sv/tailscaled/run
```

## ✅ (Facoltativo, ma consigliato) Crea script di registro

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

## ✅ Attiva il servizio al boot

```bash
sudo ln -s /etc/sv/tailscaled /var/service/
```

## ✅ Inizia subito il servizio

```bash
sudo sv up tailscaled
```

```bash
sudo sv status tailscaled
```

## ✅ Connetti il client Tailscale

## Solo dopo che il servizio è in esecuzione, esegui:

```bash
sudo tailscale up
```

## 🎉 Ora Tailscale si avvia automaticamente all'avvio

## Se vuoi confermare dopo il riavvio:

```bash
sudo tailscale status
```
```bash
sudo sv status tailscaled
```

## ✅ SE ad un certo punto hai eseguito il servizio e ha memorizzato nella cache il DNS di Tailscale, puoi tornare a utilizzare il DNS della tua rete eseguendo il comando:

```bash
sudo tailscale up --accept-dns=false
```

## Successivamente, questa nuova configurazione verrà salvata nello stato e ai successivi avvii i DNS non verranno più modificati.

---

🎯 E' TUTTO RAGAZZE!

👉 Contatto: zerolies@disroot.org
👉 https://t.me/z3r0l135


