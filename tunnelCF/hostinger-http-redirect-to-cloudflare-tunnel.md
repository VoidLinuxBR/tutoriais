# Redirecionamento HTTP → Cloudflare Tunnel mantendo HTTPS no Hostinger

Este tutorial documenta um cenário real onde:
- o domínio principal está hospedado na **Hostinger**
- um subdomínio usa **Cloudflare Tunnel** apontando para um servidor **nginx local**
- **HTTP** deve redirecionar para o túnel
- **HTTPS** deve continuar sendo servido pela Hostinger

O objetivo é separar o comportamento **por protocolo**, sem quebrar SSL, sem gambiarra e sem mover tudo para o Cloudflare.

---

## 1. Cenário inicial

Domínio:
- chililinux.com

Subdomínios envolvidos:
- void.chililinux.com → Hostinger
- voidrepo.chililinux.com → Cloudflare Tunnel → nginx local

Situação inicial:
- HTTP e HTTPS de `void.chililinux.com` apontavam para a Hostinger
- `voidrepo.chililinux.com` já funcionava via Cloudflare Tunnel

---

## 2. Dúvida inicial

A dúvida era:

> É possível fazer com que:
>
> - http://void.chililinux.com redirecione para o Cloudflare Tunnel
> - https://void.chililinux.com continue funcionando normalmente na Hostinger?

A resposta curta é:
> **Sim, mas NÃO pelo Cloudflare.**

---

## 3. Limitação técnica importante

Cloudflare **não consegue**:
- separar HTTP e HTTPS se o hostname estiver em **DNS only**
- aplicar regras de redirect se o tráfego **não passa pelo proxy**

E se o hostname for **proxied (nuvem laranja)**:
- TODO o tráfego (HTTP e HTTPS) sai da Hostinger
- o HTTPS deixa de ser terminado pelo Hostinger

Portanto:
- **Cloudflare não é o lugar certo** para resolver esse caso

---

## 4. Regra de ouro

> **Quem termina o HTTPS é quem manda no redirect.**

No cenário:
- HTTPS termina na **Hostinger**
- logo, o redirect precisa ser feito **na Hostinger**

---

## 5. Solução correta: redirect no Hostinger (.htaccess)

A Hostinger (Apache/LiteSpeed) consegue diferenciar:
- conexões HTTP
- conexões HTTPS

Portanto, o redirect deve ser feito via `.htaccess`.

---

## 6. Situação do .htaccess antes

O arquivo `.htaccess` continha apenas:

```apache
Options +Indexes
```

Isso não faz redirect nenhum.

---

## 7. Configuração correta do .htaccess

O conteúdo final correto do `.htaccess` é:

```apache
Options +Indexes

RewriteEngine On

# Se NÃO for HTTPS (ou seja, HTTP)
RewriteCond %{HTTPS} off
RewriteRule ^(.*)$ https://voidrepo.chililinux.com/$1 [R=301,L]
```

---

## 8. O que essa regra faz

- Mantém `Options +Indexes`
- Ativa o `mod_rewrite`
- Aplica a regra **somente para HTTP**
- Redireciona tudo para o Cloudflare Tunnel
- Preserva o caminho da URL
- Usa redirect permanente (301)

---

## 9. Comportamento final esperado

| URL acessada | Resultado |
|-------------|-----------|
| http://void.chililinux.com | Redireciona para https://voidrepo.chililinux.com |
| https://void.chililinux.com | Continua na Hostinger |
| https://voidrepo.chililinux.com | Cloudflare Tunnel → nginx local |

---

## 10. Teste com curl

### HTTP

```sh
curl -I http://void.chililinux.com
```

Resultado esperado:

```text
HTTP/1.1 301 Moved Permanently
Location: https://voidrepo.chililinux.com/
Server: LiteSpeed
```

---

### HTTPS

```sh
curl -I https://void.chililinux.com
```

Resultado esperado:

```text
HTTP/2 200
Server: LiteSpeed
```

---

## 11. Resultado final comprovado

- HTTP redireciona corretamente para o túnel
- HTTPS permanece funcionando no Hostinger
- Nenhuma regra no Cloudflare foi necessária
- Nenhum ajuste no túnel foi necessário
- Nenhum DNS foi alterado
- Nenhum loop foi criado

---

## 12. Conclusão

Este é o **único método correto** para esse cenário.

Tentativas de resolver isso com:
- Cloudflare Redirect Rules
- Page Rules
- CNAMEs
- múltiplos túneis

não funcionam ou quebram o HTTPS.

A solução correta é:
> **resolver no servidor que termina o TLS**.

Esse método é simples, estável e funciona de forma previsível.

---

## Observação final

Separar comportamento por protocolo **só é possível no servidor** que recebe a conexão.
DNS e proxies não fazem esse tipo de distinção quando o hostname não está totalmente sob controle deles.
