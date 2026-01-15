# Tailscale VPN рыдает Void Linux ;D

## 🎯 Цель — загрузить сетевой интерфейс, подключенный к VPN службы Tailscale.

## ✅ Загрузите и разархивируйте клиент Tailscale.

```bash
cd /tmp
```

```bash
wget https://pkgs.tailscale.com/stable/tailscale_1.90.9_amd64.tgz
```

```bash
tar xvf tailscale_1.90.9_amd64.tgz
```

## ✅ Создайте каталог службы runit.

```bash
sudo mkdir -p /etc/sv/tailscaled
```

## ✅ Создайте файл запуска

```bash
sudo tee /etc/sv/tailscaled/run << 'EOF'
#!/bin/sh
exec /usr/local/bin/tailscaled \
  --state=/var/lib/tailscale/tailscaled.state \
  2>&1
EOF
```

## ✅ Дайте разрешение на выполнение:

```bash
sudo chmod +x /etc/sv/tailscaled/run
```

## ✅ (Необязательно, но рекомендуется) Создать скрипт журнала

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

## ✅ Активируйте услугу при загрузке

```bash
sudo ln -s /etc/sv/tailscaled /var/service/
```

## ✅ Запустите услугу прямо сейчас

```bash
sudo sv up tailscaled
```

```bash
sudo sv status tailscaled
```

## ✅ Подключить клиент Tailscale

## Только после запуска службы запустите:

```bash
sudo tailscale up
```

## 🎉 Теперь Tailscale автоматически запускается при загрузке.

## Если вы хотите подтвердить после перезапуска:

```bash
sudo tailscale status
```
```bash
sudo sv status tailscaled
```

## ✅ ЕСЛИ в какой-то момент вы запустили службу и она кэшировала DNS Tailscale, вы можете вернуться к использованию собственного сетевого DNS, выполнив команду:

```bash
sudo tailscale up --accept-dns=false
```

## После этого эта новая конфигурация сохранится в состоянии, и при следующих загрузках DNS уже не будет модифицироваться.

---

🎯ВОТ ВСЕ, ЛЮДИ!

👉 Контакт: Zerolies@disroot.org
👉 https://t.me/z3r0l135


