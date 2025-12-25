# Jitsi Meet + Docker + Tailscale no Void Linux  
## Guia Final, Revisado e Correto (root)

Este tutorial cobre **tudo** o que foi feito, incluindo:
- instalação dos pacotes certos  
- ativação de serviços no Void (runit)  
- clone da stack docker-jitsi-meet  
- correção do mapeamento de portas (127.0.0.1)  
- ajuste do .env  
- ajuste do docker-compose.yml  
- problemas de porta 80 com nginx nativo  
- limitações de Tailnet (sem Funnel)  
- solução via Tailscale Serve (interno)  
- acesso final via `https://jitsi.tailf0138e.ts.net`  
- sem IP fixo, sem abrir portas, sem DNS externo

---

## 1. Instalar pacotes necessários

```bash
xbps-install -Sy docker docker-compose tailscale git
```

Habilitar serviços do Void:

```bash
ln -s /etc/sv/docker /var/service/
ln -s /etc/sv/tailscaled /var/service/
sv status docker tailscaled
```

---

## 2. Ativar e autenticar Tailscale

```bash
tailscale up
```

Abrir link no navegador, logar, autorizar.

Renomear o dispositivo dentro da Tailnet:

```bash
tailscale set --hostname=jitsi
```

Confirmar o DNSName:

```bash
tailscale status --json | grep DNSName
```

Esperado:

```
"DNSName": "jitsi.tailf0138e.ts.net.",
```

⚠ Importante:
- Se você NÃO é admin da Tailnet
- Portanto **Funnel está bloqueado**
- Mas Serve interno funciona perfeitamente

---

## 3. Baixar e preparar o stack do Jitsi

```bash
mkdir -p /opt/jitsi
cd /opt/jitsi
git clone https://github.com/jitsi/docker-jitsi-meet.git
cd docker-jitsi-meet
cp env.example .env
./gen-passwords.sh
```

---

## 4. Ajustar `.env` para uso com Tailscale  
(antes precisamos confirmar o IP e o DNS interno da Tailnet)

Antes de editar o `.env`, confirme:

**1 — IP interno Tailscale**

```bash
tailscale ip -4
```

Exemplo real usado:
```
100.75.137.60
```

**2 — Nome DNS interno da máquina na Tailnet**

```bash
tailscale status --json | grep DNSName
```

Resultado real:

```
"DNSName": "jitsi.tailf0138e.ts.net.",
```

Esse é o domínio **interno verdadeiro** gerado pelo Tailscale, com base no hostname configurado e no ID da Tailnet.

Somente depois disso edite o `.env`:

```bash
nano /opt/jitsi/docker-jitsi-meet/.env
```

Configure assim:

```ini
PUBLIC_URL=https://jitsi.tailf0138e.ts.net
ENABLE_LETSENCRYPT=0
DISABLE_HTTPS=1
ENABLE_AUTH=1
ENABLE_GUESTS=1
AUTH_TYPE=internal

XMPP_DOMAIN=meet.jitsi
#XMPP_AUTH_DOMAIN=auth.meet.jitsi
#XMPP_AUTH_DOMAIN_PREFIX=auth
XMPP_MUC_DOMAIN=muc.meet.jitsi
XMPP_INTERNAL_MUC_DOMAIN=internal-muc.meet.jitsi
XMPP_GUEST_DOMAIN=guest.meet.jitsi




```

### Justificativas:
- **PUBLIC_URL aponta para o nome interno do Tailscale**  
  pois é a URL real usada para acessar o servidor dentro da Tailnet.  

- **HTTPS dentro do container é desativado porque o TLS vem do Tailscale**  
  (o Tailscale Serve fornece o HTTPS interno e não precisamos do TLS do nginx do Jitsi).  

- **Não usamos Let’s Encrypt porque não há domínio público nem Funnel liberado**  
  e o Admin da TailNet ainda não habilitou o recurso, portanto TLS público não existe — somente interno.  

---

## 5. Ajustar docker-compose.yml  
(muito importante — foi aqui que corrigimos a maior dor de cabeça)

Esse passo é absolutamente ESSENCIAL porque foi onde corrigimos:

- o nginx do Void aparecendo no lugar do Jitsi  
- conflito de portas 80/8000  
- Tailscale Serve reclamando que “somente localhost é suportado”  
- o Jitsi sendo servido externamente sem querer  
- o backend não funcionando no Serve  
- a necessidade de expose local-only  
- inicialização automática dos containers  
- preparação para FUNNEL futuro sem alterar nada depois  

O serviço `web` deve expor APENAS no localhost, pois:

- **Tailscale Serve EXIGE backend em 127.0.0.1**  
  (versão atual do Serve só aceita localhost, senão dá erro de proxy)
- **Evita conflito com o nginx do Void**, que roda na porta 80 do sistema  
  (foi por isso que aparecia “Welcome to nginx!”)
- **Garante que o Serve encaminhe para o Jitsi**, não para o nginx do host
- **Impede exposição acidental na internet**, já que localhost não aceita conexões externas
- **Garante compatibilidade futura com FUNNEL**, caso o Admin da TailNet libere
- **Com `restart: always`, os containers sobem automaticamente após reboot**, sem runit adicional

Edite o compose:

```bash
nano /opt/jitsi/docker-jitsi-meet/docker-compose.yml
```

E deixe EXATAMENTE assim:

```yaml
services:

  web:
    image: jitsi/web:unstable
    restart: always
    ports:
      - "127.0.0.1:8000:80"
      - "127.0.0.1:8443:443"

  prosody:
    image: jitsi/prosody:unstable
    restart: always

  jicofo:
    image: jitsi/jicofo:unstable
    restart: always

  jvb:
    image: jitsi/jvb:unstable
    restart: always
```

Explicação resumida:

- **127.0.0.1:8000 → 80**  
  → A porta 80 do container só existe internamente, e quem recebe é 127.0.0.1  
  → Por isso o Tailscale Serve consegue redirecionar corretamente

- **restart: always**  
  → Se o Void reiniciar, o Jitsi volta sozinho  
  → Se o Docker reiniciar, o Jitsi volta sozinho  
  → Se houver queda de energia, volta sozinho  

- **Isso elimina 100% o problema do nginx do Void**  
- **Isso deixa o Jitsi invisível na internet pública** (o que é desejado dentro da Tailnet)
- **Isso prepara tudo para ativar Funnel no futuro com apenas um comando**

Salvar e sair.

---

## 6. Subir o stack Docker

```bash
docker-compose up -d
docker-compose ps
```

Confirme que o web está **127.0.0.1:8000 → 80**.

Testar o frontend dentro do servidor:

```bash
curl -I http://127.0.0.1:8000
```

Esperado:

```
HTTP/1.1 200 OK
Server: nginx
```

⚠ Se você visse "Welcome to nginx!", era o nginx do Void.  
Esse teste garantiu que o backend do Jitsi está correto.

---

## 7. Expor via Tailscale Serve (interno)

Resetar qualquer regra anterior:

```bash
tailscale serve reset
```

Criar o proxy interno:

```bash
tailscale serve --bg http://127.0.0.1:8000
```

Saída esperada:

```
Available within your tailnet:

https://jitsi.tailf0138e.ts.net/
|-- proxy http://127.0.0.1:8000
```

Verificar status:

```bash
tailscale serve status
```

Agora o Tailscale Serve está corretamente servindo o Jitsi.

---

## 8. Acessar via Tailnet (funciona em QUALQUER rede)

No notebook, celular, PC — desde que logado no Tailscale:

```
https://jitsi.tailf0138e.ts.net/
```

Sim:

- HTTPS funciona  
- Certificado do Tailscale é válido  
- Nada de aviso  
- Nada de nginx do Void  
- Nada de :8000  
- Tudo direto no domínio bonito  

⚠ Acesso **somente** para membros da Tailnet (por enquanto).

---

## 9. Comandos úteis

Ver containers:

```bash
docker-compose ps
```

Logs:

```bash
docker-compose logs -f web
```

Parar:

```bash
docker-compose down
```

Serve status:

```bash
tailscale serve status
```

Reset serve:

```bash
tailscale serve reset
```

---

## 10. Adicionando usuarios

```bash
docker compose exec prosody prosodyctl --config /config/prosody.cfg.lua register admin meet.jitsi Jitsi1234
```

Saída esperada:

```
usermanager         info	User account created: admin@meet.jitsi
```

---

## 11. Quando o Admin da TailNet liberar o FUNNEL (opcional, acesso público)

Se o **Admin da TailNet** habilitar o Funnel, você poderá expor
o Jitsi para a INTERNET inteira, com HTTPS válido, sem depender de firewall, modem ou IP fixo.

Com Funnel ativado, você executa:

```bash
tailscale funnel --https=443 http://127.0.0.1:8000
```

E o acesso passa a ser:

```
https://jitsi.tailf0138e.ts.net/
```

---

### 🔶 NOTA IMPORTANTE: COMO LIBERAR O FUNNEL

Somente o **administrador da TailNet** pode habilitar o Funnel.

O Admin precisa fazer:

1. Entrar em:
   https://login.tailscale.com/admin/acls

2. No menu lateral, clicar em:
   **Settings → Funnel**

3. Ativar a opção:
   ✔ **Allow Funnel for this tailnet**

4. E ativar também:
   ✔ selecionar o dispositivo **jitsi**  
     (ou o nome que você configurou com `tailscale set --hostname`)

5. Salvar.

Depois disso, você testa:

```bash
tailscale funnel status
```

Se estiver liberado, o comando deixa de dar erro e você pode ativar Funnel normalmente.

---

### ✔ O que muda quando o Funnel está ativo

- O Jitsi fica acessível PUBLICAMENTE (sem TailNet)
- HTTPS válido automaticamente (via Let's Encrypt do Tailscale)
- A URL permanece:
  ```
  https://jitsi.tailf0138e.ts.net/
  ```
- Pode ser compartilhada com QUALQUER pessoa

---

### ✔ O que NÃO muda

- Nada do tutorial anterior quebra  
- Serve interno continua funcionando  
- Docker não precisa ser modificado  
- Jitsi não precisa reiniciar  

---

## FIM  
Configuração revisada, limpa, sem buracos.  
Tudo em ordem certa e funcionando.
