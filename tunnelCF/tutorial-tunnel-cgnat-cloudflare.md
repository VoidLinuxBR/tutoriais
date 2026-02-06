# Cloudflare Tunnel + Nginx no Void Linux (CGNAT)

Este tutorial mostra como publicar um servidor **nginx local**, rodando em **Void Linux**, atrás de **CGNAT**, usando **Cloudflare Tunnel**, sem abrir portas, sem VPS e sem HTTPS local.

O cenário final é:
- URL pública: https://voidrepo.chililinux.com
- Backend: nginx HTTP local
- CGNAT: ignorado
- Proxy / SSL: Cloudflare
- Init: runit
- Distro: Void Linux

---

## 1. Pré-requisitos

- Domínio já gerenciado pelo Cloudflare
- Void Linux com acesso à internet
- nginx instalado
- cloudflared instalado
- Firewall liberando SAÍDA TCP 443
- Nenhuma porta aberta no modem

---

## 2. Instalar o cloudflared

Via repositório (se disponível):

```sh
xbps-install -S cloudflared
```

Ou binário direto:

```sh
curl -L https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64 \
  -o /usr/local/bin/cloudflared
chmod +x /usr/local/bin/cloudflared
```

Verificar:

```sh
cloudflared --version
```

---

## 3. Login no Cloudflare (gera cert.pem)

O comando abaixo **autentica a máquina no Cloudflare** e gera o arquivo `cert.pem`.

⚠️ IMPORTANTE:  
O `cert.pem` é salvo **na HOME do usuário que executa o comando**.

```sh
cloudflared tunnel login
```

- Abrirá o navegador
- Fazer login no Cloudflare
- Selecionar o domínio
- Autorizar

Resultado esperado:

```text
~/.cloudflared/cert.pem
```

Exemplo:
- Usuário comum: `/home/suporte/.cloudflared/cert.pem`
- Root: `/root/.cloudflared/cert.pem`

---

## 4. Onde colocar o cert.pem (REGRA DE OURO)

O `cloudflared` **só procura o `cert.pem` na HOME do usuário que executa o tunnel**.

O MESMO usuário deve:
- possuir o `cert.pem`
- criar o tunnel
- executar o cloudflared (manual ou serviço)

Misturar usuários = erro.

---

### Opção A — Usando usuário comum (ex: suporte)

Se o login foi feito como `suporte`, o arquivo estará em:

```text
/home/suporte/.cloudflared/cert.pem
```

Confirmação:

```sh
ls -l /home/suporte/.cloudflared/cert.pem
```

O tunnel e o `.json` serão criados nesse mesmo diretório.

---

### Opção B — Usando root (recomendado para serviço)

Se o objetivo é rodar o tunnel como **serviço permanente**, o ideal é usar **root**.

1. Criar diretório:
```sh
sudo mkdir -p /root/.cloudflared
```

2. Copiar o cert.pem:
```sh
sudo cp /home/suporte/.cloudflared/cert.pem /root/.cloudflared/
```

3. Ajustar permissões:
```sh
sudo chown root:root /root/.cloudflared/cert.pem
sudo chmod 600 /root/.cloudflared/cert.pem
```

4. Confirmar:
```sh
ls -l /root/.cloudflared/cert.pem
```

⚠️ A partir daqui, **todos os comandos devem ser executados como root**.

---

## 5. Criar o Tunnel

```sh
cloudflared tunnel create chililinux
```

Isso cria:
- Tunnel ID (UUID)
- Arquivo de credenciais:

```text
/root/.cloudflared/<UUID>.json
```

---

## 6. Criar DNS do subdomínio

```sh
cloudflared tunnel route dns chililinux voidrepo.chililinux.com
```

No Cloudflare será criado automaticamente:

```text
voidrepo.chililinux.com -> UUID.cfargotunnel.com
```

Não usar A record.  
Não usar IP.

---

## 7. Configurar o cloudflared

Criar diretório:

```sh
mkdir -p /etc/cloudflared
```

Criar arquivo:

```text
/etc/cloudflared/config.yml
```

Conteúdo:

```yaml
tunnel: chililinux
credentials-file: /etc/cloudflared/<UUID>.json

ingress:
  - hostname: voidrepo.chililinux.com
    service: http://127.0.0.1:80
  - service: http_status:404
```

Permissões:

```sh
chown root:root /etc/cloudflared/*
chmod 600 /etc/cloudflared/*
```

---

## 8. Testar o tunnel manualmente

```sh
cloudflared tunnel run chililinux
```

Abrir no navegador:

```text
https://voidrepo.chililinux.com
```

Se abrir, o tunnel está funcionando.

---

## 9. Ativar o serviço no Void (runit)

Criar link do serviço:

```sh
ln -s /etc/sv/cloudflared /var/service/
```

Verificar:

```sh
sv status cloudflared
```

---

## 10. Instalar e configurar o nginx

Instalar:

```sh
xbps-install -S nginx
ln -s /etc/sv/nginx /var/service/
```

Criar estrutura:

```sh
mkdir -p /voidrepo/void-mirror/voidlinux/docs
mkdir -p /voidrepo/void-mirror/voidlinux/current
```

---

## 11. Configuração do nginx (repositório)

Arquivo:

```text
/etc/nginx/conf.d/voidrepo.conf
```

Conteúdo:

```nginx
server {
  listen 80;
  server_name voidrepo.chililinux.com;

  location / {
    root /voidrepo/void-mirror/voidlinux/docs;
    index index.html;
    autoindex off;
  }

  location /voidlinux/current/ {
    alias /voidrepo/void-mirror/voidlinux/current/;
    autoindex on;
    autoindex_exact_size off;
    autoindex_localtime on;
  }
}
```

⚠️ IMPORTANTE:
- `root` concatena caminho
- `alias` substitui caminho
- `alias` SEMPRE com `/` no final

---

## 12. Logs do nginx no Void

Void não cria diretórios automaticamente.

```sh
mkdir -p /var/log/nginx
mkdir -p /run/nginx
touch /var/log/nginx/access.log /var/log/nginx/error.log
chown -R root:nginx /var/log/nginx /run/nginx
chmod 750 /var/log/nginx
chmod 755 /run/nginx
```

---

## 13. Recarregar nginx

```sh
nginx -t
sv restart nginx
```

Testes locais:

```sh
curl http://127.0.0.1/
curl http://127.0.0.1/voidlinux/current/
```

---

## 14. Resultado final

- URL pública:
  https://voidrepo.chililinux.com

- Backend:
  nginx HTTP local

- CGNAT:
  irrelevante

- Portas abertas:
  nenhuma

- SSL:
  Cloudflare

---

## Observações finais

- Cloudflare Tunnel SEMPRE entrega HTTPS externamente
- nginx não precisa de SSL
- Não abrir portas
- Não usar IP público
- Ideal para mirrors, docs e sites estáticos

Este método é simples, estável e funciona em ambiente residencial com CGNAT.
