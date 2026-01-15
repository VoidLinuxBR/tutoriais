# VPN de escala trasera llora por Void Linux; D

## 🎯 Objetivo: cargar una interfaz de red conectada a la VPN del servicio Tailscale.

## ✅ Descargue y descomprima el cliente Tailscale

```bash
cd /tmp
```

```bash
wget https://pkgs.tailscale.com/stable/tailscale_1.90.9_amd64.tgz
```

```bash
tar xvf tailscale_1.90.9_amd64.tgz
```

## ✅ Crear directorio de servicios runit

```bash
sudo mkdir -p /etc/sv/tailscaled
```

## ✅ Crea el archivo de ejecución

```bash
sudo tee /etc/sv/tailscaled/run << 'EOF'
#!/bin/sh
exec /usr/local/bin/tailscaled \
  --state=/var/lib/tailscale/tailscaled.state \
  2>&1
EOF
```

## ✅ Dar permiso de ejecución:

```bash
sudo chmod +x /etc/sv/tailscaled/run
```

## ✅ (Opcional, pero recomendado) Crear script de registro

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

## ✅ Activar el servicio al arrancar

```bash
sudo ln -s /etc/sv/tailscaled /var/service/
```

## ✅ Iniciar servicio ahora

```bash
sudo sv up tailscaled
```

```bash
sudo sv status tailscaled
```

## ✅ Conecte el cliente Tailscale

## Sólo después de que el servicio se esté ejecutando, ejecute:

```bash
sudo tailscale up
```

## 🎉 Ahora Tailscale se activa automáticamente al arrancar

## Si desea confirmar después de reiniciar:

```bash
sudo tailscale status
```
```bash
sudo sv status tailscaled
```

## ✅ SI en algún momento ejecutó el servicio y almacenó en caché el DNS de Tailscale, puede volver a usar su propio DNS de red ejecutando el comando:

```bash
sudo tailscale up --accept-dns=false
```

## Después de eso, esta nueva configuración se guardará en el estado, y en los próximos arranques ya no se modificarán los DNS.

---

🎯 ¡ESO ES TODO AMIGOS!

👉 Contacto: zerolies@disroot.org
👉https://t.me/z3r0l135


