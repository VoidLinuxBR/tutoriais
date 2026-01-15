# Tailscale VPN sanglote Void Linux ;D

## 🎯 Objectif - Mettre en ligne une interface réseau connectée au VPN du service Tailscale.

## ✅ Téléchargez et décompressez le client Tailscale

```bash
cd /tmp
```

```bash
wget https://pkgs.tailscale.com/stable/tailscale_1.90.9_amd64.tgz
```

```bash
tar xvf tailscale_1.90.9_amd64.tgz
```

## ✅ Créer un répertoire de services runit

```bash
sudo mkdir -p /etc/sv/tailscaled
```

## ✅ Créez le fichier d'exécution

```bash
sudo tee /etc/sv/tailscaled/run << 'EOF'
#!/bin/sh
exec /usr/local/bin/tailscaled \
  --state=/var/lib/tailscale/tailscaled.state \
  2>&1
EOF
```

## ✅ Donnez l'autorisation d'exécution :

```bash
sudo chmod +x /etc/sv/tailscaled/run
```

## ✅ (Facultatif, mais recommandé) Créer un script de journal

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

## ✅ Activez le service au démarrage

```bash
sudo ln -s /etc/sv/tailscaled /var/service/
```

## ✅ Démarrez le service maintenant

```bash
sudo sv up tailscaled
```

```bash
sudo sv status tailscaled
```

## ✅ Connectez le client Tailscale

## Seulement une fois le service exécuté, exécutez :

```bash
sudo tailscale up
```

## 🎉 Désormais, Tailscale monte automatiquement au démarrage

## Si vous souhaitez confirmer après redémarrage :

```bash
sudo tailscale status
```
```bash
sudo sv status tailscaled
```

## ✅ SI vous avez à un moment donné exécuté le service et qu'il a mis en cache le DNS de Tailscale, vous pouvez recommencer à utiliser le DNS de votre propre réseau en exécutant la commande :

```bash
sudo tailscale up --accept-dns=false
```

## Après cela, cette nouvelle configuration sera sauvegardée dans l'état, et aux prochains démarrages le DNS ne sera plus modifié.

---

🎯 C'EST TOUS LES GENS !

👉Contact : zerolies@disroot.org
👉 https://t.me/z3r0l135


