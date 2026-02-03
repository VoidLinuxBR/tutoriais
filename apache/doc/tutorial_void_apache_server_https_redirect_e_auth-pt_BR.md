# 🌐 Subindo um Apache Server no Void Linux (glibc)

## 🔐 HTTPS + Redirect + Autenticação HTTP Basic

---

## 🧪 AMBIENTE

- **Sistema Operacional:** Void Linux Server (glibc)
- **Servidor Web:** Apache 2.4
- **Sistema de inicialização:** runit
- **Conteúdo:** Estático (MkDocs)
- **Autenticação:** HTTP Basic (htpasswd)

---

## 1️⃣ INSTALAÇÃO DOS PACOTES

Logar com o usuário root:

```bash
su -
```

Atualizar repositórios, pacotes e sistema operacional:

```bash
xbps-install -Syu
```

Instalar Apache e utilitários

```bash
xbps-install apache openssl
```

---

## 2️⃣ ATIVAR O SERVIÇO APACHE (RUNIT)

Ativar o serviço do Apache:

```bash
ln -s /etc/sv/apache /var/service/
```

Verifique o status:

```bash
sv status apache
```

Resultado esperado:

```bash
run: apache: (pid 1045) 16s; run: log: (pid 1044) 16s
```

No Void Linux a configuração principal fica em:

```bash
/etc/apache/httpd.conf
```

As configurações auxiliares ficam em:

```bash
/etc/apache/extra/
```

Os VirtualHosts em:

```bash
/etc/apache/extra/httpd-vhosts.conf
```

E o arquivo de SSL em:

```bash
/etc/apache/extra/httpd-ssl.conf
```

---

## 3️ RUTA PARA SERVIÇO APAlic

```bash
curl -I http://192.168.70.100
```

Resultado esperado:

```bash
HTTP/1.1 200 OK
Date: Mon, 02 Feb 2026 18:25:24 GMT
Server: Apache
Content-Type: text/html;charset=ISO-8859-1
```

---

## 4️⃣ CONFIRMAR OS INCLUDES QUE ATIVAM OS MÓDULOS

No Void, **não existe**:

- /etc/apache2
- sites disponíveis
- a2enmod, a2ensite

Os módulos são carregados via httpd.conf. através dos includes, valide se estas linhas existem e estejam descomentadas

```bash
vim /etc/apache/httpd.conf
```

VirtualHosts e SSL:

```bash
Include /etc/apache/extra/httpd-vhosts.conf
Include /etc/apache/extra/httpd-ssl.conf
```

Módulos necessários:

```bash
LoadModule ssl_module libexec/apache/mod_ssl.so
LoadModule socache_shmcb_module libexec/apache/mod_socache_shmcb.so
LoadModule rewrite_module libexec/apache/mod_rewrite.so
LoadModule auth_basic_module libexec/apache/mod_auth_basic.so
LoadModule authn_file_module libexec/apache/mod_authn_file.so
```

Sem isso, HTTPS e autenticação não funcionam!

---

## 5️⃣ DOCUMENT ROOT E PERMISSÕES

DocumentRoot padrão:

```bash
/srv/www/apache
```

Criar estrutura:

```bash
mkdir /srv/www/apache/publico
mkdir /srv/www/apache/aluno
```

Ajustar permissões de usuário e grupo (_apache):

```bash
chown -R _apache:_apache /srv/www/apache/
chmod -R 755 /srv/www/apache/
```

Verificar

```bash
ls -la /srv/www/apache/
```

Resultado esperado

```bash
total 16K
drwxrwxr-x 4 _apache _apache 4,0K fev  2 13:15 ./
drwxr-xr-x 3 root    root    4,0K fev  2 11:24 ../
drwxrwxr-x 2 _apache _apache 4,0K fev  2 13:15 aluno/
drwxrwxr-x 2 _apache _apache 4,0K fev  2 13:15 publico/
```

---

## 6️⃣ CRIAÇÃO DO CERTIFICADO SSL AUTOASSINADO

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/certs/conectux.key \
  -out /etc/ssl/certs/conectux.crt \
  -subj "/C=BR/ST=SP/L=LOCAL/O=Conectux/CN=192.168.122.100" \
  -addext "subjectAltName = IP:192.168.122.100, DNS:conectux.edu"
```

Depois de gerar, valide:

```bash
openssl x509 -in /etc/ssl/certs/conectux.crt -text -noout | grep -A2 "Subject Alternative Name"
```

---

## 7️⃣ CONFIGURAÇÃO HTTPS (VIRTUALHOST 443)

Criar backup e editar o arquivo:

```bash
cp /etc/apache/extra/http-ssl.conf{,.bkp}
vim /etc/apache/extra/httpd-ssl.conf
```

Conteúdo:

```bash
Listen 443

<VirtualHost _default_:443>
    ServerName 192.168.122.100
    DocumentRoot "/srv/www/apache"

    ErrorLog "/var/log/httpd/error.log"
    CustomLog "/var/log/httpd/access.log" combined

    SSLEngine on
    SSLCertificateFile "/etc/ssl/certs/conectux.crt"
    SSLCertificateKeyFile "/etc/ssl/certs/conectux.key"

    <Directory "/srv/www/apache">
        Options -Indexes
        AllowOverride None
        Require all granted
    </Directory>

    <Directory "/srv/www/apache/aluno">
        Options -Indexes
        AllowOverride None

        AuthType Basic
        AuthName "Área do Aluno"
        AuthUserFile "/etc/apache/matriculados"
        Require valid-user
    </Directory>
</VirtualHost>
```

NOTE que o path de log, no default, está apontando pra um caminho inexistente

```bash
ErrorLog "/var/log/apache/error.log"
CustomLog "/var/log/apache/access.log" combined
```

Alteramos corretamente para

```bash
ErrorLog "/var/log/httpd/error.log"
CustomLog "/var/log/httpd/access.log" combined
```

---

## 8️⃣ HTTP + REDIRECIONAMENTO PARA HTTPS

Criar backup e editar o arquivo:

```bash
cp /etc/apache/extra/httpd-vhosts.conf{,.bkp}
vim /etc/apache/extra/httpd-vhosts.conf
```

Conteúdo:

```bash
<VirtualHost *:80>
    ServerName 192.168.122.100
    DocumentRoot "/srv/www/apache"
    Redirect permanent / https://192.168.122.100/
</VirtualHost>
```

---

## 9️⃣ CRIAÇÃO DE USUÁRIOS (HTPASSWD)

Criar arquivo seguro:

```bash
> /etc/apache/matriculados
```

Criar o arquivo que conterá os usuários autenticados fora do acesso público

```bash
chown root:_apache /etc/apache/matriculados
```

Setar permissão ao arquivo

```bash
chmod 640 /etc/apache/matriculados
```

Criar usuários com permissão de acesso ao conteúdo restrito da página
A primeira inserção, conta com a criação do arquivo, por isso a opção -c

```bash
htpasswd -c /etc/apache/matriculados aluno01
```

Valide a criação do usuário e senha, no arquivo

```bash
cat  /etc/apache/matriculados
```

Resultado

```bash
aluno01:$apr1$jCnS/66r$HrJoIImLRKzt46JsDx7n70
```

Adicionar novos usuários:
Sempre sem a opção -c, de agora em diante

```bash
htpasswd /etc/apache/matriculados aluno02
```

---

## 📄 VALIDAR SINTAXE HTML

Testar sintaxe e validar a configuração do Apache:

```bash
apachectl configtest
```

Resultado esperado:

```bash
Syntax OK
```

Reinicie o Apache:

```bash
sv restart apache
```

---

## 🧪 TESTES E RESOLUÇÃO DE PROBLEMAS

Validar as portas 80 e 443 abertas:

```bash
netstat -an | grep :80
```

Resultado

```bash
tcp6       0      0 :::80                   :::*                    LISTEN
```

```bash
netstat -an | grep :443
```

Resultado esperado

```bash
tcp6       0      0 :::443                  :::*                    LISTEN
```

Ver registros:

```bash
tail -f /var/log/apache/error.log
```

Testar SSL (vai lançar diversas informações na tela):

```bash
openssl s_client -connect 192.168.70.251:443
```

Testar autenticação:

```bash
curl -I https://192.168.70.251/aluno/
```

Resultado esperado:

```bash
HTTP/1.1 401 Unauthorized
```

---

## CRIAÇÃO DE CONTEÚDO HTML E TESTES DE AUTENTICAÇÃO E ACESSO

Sendo que já criamos os paths e setamos permissões em /srv/www/apache/publico e /srv/www/apache/aluno

Criamos a página em html da Área Pública

```bash
vim /srv/www/apache/publico/index.html
```

Conteúdo do arquivo

```bash
<!DOCTYPE html>
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <title>Área Pública</title>
</head>
<body>
    <h1>Área Pública</h1>
    <p>Este conteúdo é de livre acesso.</p>

    <hr>

    <p>
        <a href="/aluno/">Ir para a Área do Aluno (acesso restrito)</a>
    </p>
</body>
</html>
```

Bem como criamos a página html da área do Aluno

```bash
vim /var/www/apache/aluno/index.html
```

Conteúdo

```bash
<!DOCTYPE html>
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <title>Área do Aluno</title>
</head>
<body>
    <h1>Área do Aluno</h1>
    <p><strong>Acesso autorizado.</strong></p>

    <p>Se você está vendo esta página, a autenticação HTTP Basic funcionou corretamente.</p>

    <hr>

    <p>
        <a href="/publico/">Voltar para a Área Pública</a>
    </p>
</body>
</html>
```

Garanta que o Apache consiga ler os arquivos:

```bash
chown -R _apache:_apache /srv/www/apache
chmod -R 755 /srv/www/apache
```

Testes pelo terminal podem ser efetivados, usando o curl

```bash
curl -k https://192.168.122.100/publico/
```

Resultado - conteúdo html

```bash
<!DOCTYPE html>
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <title>Área Pública</title>
</head>
<body>
    <h1>Área Pública</h1>
    <p>Este conteúdo é de livre acesso.</p>

    <hr>

    <p>
        <a href="/aluno/">Ir para a Área do Aluno (acesso restrito)</a>
    </p>
</body>
</html>
```

```bash
curl -k -I https://192.168.122.100/aluno/
```

Resultado - HTTP/1.1 401 Não autorizado

```bash
HTTP/1.1 401 Unauthorized
Date: Mon, 02 Feb 2026 21:20:32 GMT
Server: Apache
WWW-Authenticate: Basic realm="Área do Aluno"
Vary: accept-language,accept-charset
Accept-Ranges: bytes
Content-Type: text/html; charset=utf-8
Content-Language: en
```

Pode testar com autenticação e ver a liberação

```bash
curl -k -u aluno01 https://192.168.122.100/aluno/
```

Pelo navegador, a Área Pública, não deve pedir senha para acesso ao conteúdo

https://192.168.122.100/publico/

Diferentemente da Área do Aluno, que deve solicitar as credenciais para acesso ao conteúdo

https://192.168.122.100/aluno/


---

## 🔐 IMPORTAR CERTIFICADO NO FIREFOX

Acessar:

sobre:preferências#privacidade

- Certificados → Ver certificados → Autoridades → Importar
- Selecionar conectux.crt
- Marcar: "Confiar nessa CA para identificar sites"
- Reinicie o navegador.

---

## 📚 INSTALAÇÃO DO MKDOCS


A instalação da ferramenta do Void Server Mkdocs, foi contemplada na página do github do canal, que você tem acesso aqui

https://github.com/VoidLinuxBR/tutoriais/blob/main/misc/tutorial-void-server-mkdocs.md

Será útil na criação de conteúdo html para exportar ao Apache, se você não for um desenvolvedor de frontend.

---

## ✅ CONCLUSÃO

- Apache com HTTPS funcional
- Redirecionar HTTP → HTTPS
- Autenticação HTTP Basic
- Estrutura limpa

🎯 **ISSO É TUDO GENTE**

