# Tailscale VPN sob Void Linux ;D

## 🎯 Objetivo - Subir uma interface de rede conectada á VPN do serviço do Tailscale.

## ✅ Baixar e descompactar o client do Tailscale

```bash
cd /tmp
```

```bash
wget https://pkgs.tailscale.com/stable/tailscale_1.90.9_amd64.tgz
```

```bash
tar xvf tailscale_1.90.9_amd64.tgz
```

## ✅ Crie o diretório do serviço runit

```bash
sudo mkdir -p /etc/sv/tailscaled
```

## ✅ Crie o arquivo run

```bash
sudo tee /etc/sv/tailscaled/run << 'EOF'
#!/bin/sh
exec /usr/local/bin/tailscaled \
  --state=/var/lib/tailscale/tailscaled.state \
  2>&1
EOF
```

## ✅ Dê permissão de execução:

```bash
sudo chmod +x /etc/sv/tailscaled/run
```

## ✅ (Opcional, mas recomendado) Criar script de log

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

## ✅ Ativar o serviço no boot

```bash
sudo ln -s /etc/sv/tailscaled /var/service/
```

## ✅ Iniciar o serviço agora

```bash
sudo sv up tailscaled
```

```bash
sudo sv status tailscaled
```

## ✅ Conectar o cliente Tailscale

## Somente após o serviço estar rodando, execute:

```bash
sudo tailscale up
```

## 🎉 Agora o Tailscale sobe automaticamente no boot

## Se quiser confirmar após reiniciar:

```bash
sudo tailscale status
```
```bash
sudo sv status tailscaled
```

## ✅ CASO tenha em algum momento rodado o serviço e ele tenha cacheado o DNS do Tailscale, pode voltar a usar o seu próprio DNS da rede, rodando o comando:

```bash
sudo tailscale up --accept-dns=false
```

## Depois disso, essa nova configuração será salva no state, e nos próximos boots o DNS não será mais modificado.

---

🎯 THAT'S ALL FOLKS!

👉 Contato: zerolies@disroot.org
👉 https://t.me/z3r0l135


