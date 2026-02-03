# 🌐 Uploading an Apache Server on Void Linux (glibc)

## 🔐 HTTPS + Redirect + HTTP Basic Authentication

---

## 🧪 ENVIRONMENT

- **Sistema Operacional:** Void Linux Server (glibc)
- **Web Server:** Apache 2.4
- **Init System:** runit
- **Content:** Static (MkDocs)
- **Authentication:** HTTP Basic (htpasswd)

---

## 1️⃣ INSTALLATION OF PACKAGES

Log in with the root user:

```bash
su -
```

Update repositories, packages and operating system:

```bash
xbps-install -Syu
```

Install Apache and utilities

```bash
xbps-install apache openssl
```

---

## 2️⃣ ACTIVATE THE APACHE SERVICE (RUNIT)

Enable the Apache service:

```bash
ln -s /etc/sv/apache /var/service/
```

Check status:

```bash
sv status apache
```

Expected result:

```bash
run: apache: (pid 1045) 16s; run: log: (pid 1044) 16s
```

In Void Linux the main configuration is:

```bash
/etc/apache/httpd.conf
```

The auxiliary settings are:

```bash
/etc/apache/extra/
```

The VirtualHosts in:

```bash
/etc/apache/extra/httpd-vhosts.conf
```

And the SSL file at:

```bash
/etc/apache/extra/httpd-ssl.conf
```

---

## 3️ RUT TO APAlic SERVICE

```bash
curl -I http://192.168.70.100
```

Expected result:

```bash
HTTP/1.1 200 OK
Date: Mon, 02 Feb 2026 18:25:24 GMT
Server: Apache
Content-Type: text/html;charset=ISO-8859-1
```

---

## 4️⃣ CONFIRM THE INCLUDES THAT ACTIVATE THE MODULES

In Void, **does not exist**:

- /etc/apache2
- sites-available
- a2enmod, a2ensite

Modules are loaded via httpd.conf. through includes, validate that these lines exist and are uncommented

```bash
vim /etc/apache/httpd.conf
```

VirtualHosts e SSL:

```bash
Include /etc/apache/extra/httpd-vhosts.conf
Include /etc/apache/extra/httpd-ssl.conf
```

Required modules:

```bash
LoadModule ssl_module libexec/apache/mod_ssl.so
LoadModule socache_shmcb_module libexec/apache/mod_socache_shmcb.so
LoadModule rewrite_module libexec/apache/mod_rewrite.so
LoadModule auth_basic_module libexec/apache/mod_auth_basic.so
LoadModule authn_file_module libexec/apache/mod_authn_file.so
```

Without this, HTTPS and authentication don't work!

---

## 5️⃣ DOCUMENT ROOT AND PERMISSIONS

Default DocumentRoot:

```bash
/srv/www/apache
```

Create structure:

```bash
mkdir /srv/www/apache/publico
mkdir /srv/www/apache/aluno
```

Adjust user and group permissions (_apache):

```bash
chown -R _apache:_apache /srv/www/apache/
chmod -R 755 /srv/www/apache/
```

To check

```bash
ls -la /srv/www/apache/
```

Expected result

```bash
total 16K
drwxrwxr-x 4 _apache _apache 4,0K fev  2 13:15 ./
drwxr-xr-x 3 root    root    4,0K fev  2 11:24 ../
drwxrwxr-x 2 _apache _apache 4,0K fev  2 13:15 aluno/
drwxrwxr-x 2 _apache _apache 4,0K fev  2 13:15 publico/
```

---

## 6️⃣ CREATION OF SELF-SIGNED SSL CERTIFICATE

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/certs/conectux.key \
  -out /etc/ssl/certs/conectux.crt \
  -subj "/C=BR/ST=SP/L=LOCAL/O=Conectux/CN=192.168.122.100" \
  -addext "subjectAltName = IP:192.168.122.100, DNS:conectux.edu"
```

After generating, validate:

```bash
openssl x509 -in /etc/ssl/certs/conectux.crt -text -noout | grep -A2 "Subject Alternative Name"
```

---

## 7️⃣ HTTPS SETUP (VIRTUALHOST 443)

Create backup and edit file:

```bash
cp /etc/apache/extra/http-ssl.conf{,.bkp}
vim /etc/apache/extra/httpd-ssl.conf
```

Content:

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

NOTE that the log path, by default, is pointing to a non-existent path

```bash
ErrorLog "/var/log/apache/error.log"
CustomLog "/var/log/apache/access.log" combined
```

We correctly changed it to

```bash
ErrorLog "/var/log/httpd/error.log"
CustomLog "/var/log/httpd/access.log" combined
```

---

## 8️⃣ HTTP + REDIRECT PARA HTTPS

Create backup and edit file:

```bash
cp /etc/apache/extra/httpd-vhosts.conf{,.bkp}
vim /etc/apache/extra/httpd-vhosts.conf
```

Content:

```bash
<VirtualHost *:80>
    ServerName 192.168.122.100
    DocumentRoot "/srv/www/apache"
    Redirect permanent / https://192.168.122.100/
</VirtualHost>
```

---

## 9️⃣ USER CREATION (HTPASSWD)

Create secure archive:

```bash
> /etc/apache/matriculados
```

Create the file that will contain authenticated users outside of public access

```bash
chown root:_apache /etc/apache/matriculados
```

Set file permission

```bash
chmod 640 /etc/apache/matriculados
```

Create users with permission to access restricted page content
The first insertion involves the creation of the file, which is why the -c option

```bash
htpasswd -c /etc/apache/matriculados aluno01
```

Validate the creation of the username and password in the file

```bash
cat  /etc/apache/matriculados
```

Result

```bash
aluno01:$apr1$jCnS/66r$HrJoIImLRKzt46JsDx7n70
```

Add new users:
Always without the -c option, from now on

```bash
htpasswd /etc/apache/matriculados aluno02
```

---

## 📄 VALIDATE HTML SYNTAX

Test syntax and validate Apache configuration:

```bash
apachectl configtest
```

Expected result:

```bash
Syntax OK
```

Restart Apache:

```bash
sv restart apache
```

---

## 🧪 TESTES E TROUBLESHOOTING

Validate open ports 80 and 443:

```bash
netstat -an | grep :80
```

Result

```bash
tcp6       0      0 :::80                   :::*                    LISTEN
```

```bash
netstat -an | grep :443
```

Expected result

```bash
tcp6       0      0 :::443                  :::*                    LISTEN
```

Ver logs:

```bash
tail -f /var/log/apache/error.log
```

Test SSL (it will display various information on the screen):

```bash
openssl s_client -connect 192.168.70.251:443
```

Test authentication:

```bash
curl -I https://192.168.70.251/aluno/
```

Expected result:

```bash
HTTP/1.1 401 Unauthorized
```

---

## HTML CONTENT CREATION AND AUTHENTICATION AND ACCESS TESTING

Since we have already created the paths and set permissions in /srv/www/apache/publico and /srv/www/apache/aluno

We created the Public Area HTML page

```bash
vim /srv/www/apache/publico/index.html
```

File contents

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

As well as creating the HTML page for the Student area

```bash
vim /var/www/apache/aluno/index.html
```

Content

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

Ensure that Apache can read the files:

```bash
chown -R _apache:_apache /srv/www/apache
chmod -R 755 /srv/www/apache
```

Terminal tests can be carried out using curl

```bash
curl -k https://192.168.122.100/publico/
```

Result - html content

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

Resultado - HTTP/1.1 401 Unauthorized

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

You can test with authentication and see the release

```bash
curl -k -u aluno01 https://192.168.122.100/aluno/
```

Through the browser, the Public Area should not ask for a password to access the content

https://192.168.122.100/publico/

Unlike the Student Area, which must request credentials to access content

https://192.168.122.100/aluno/


---

## 🔐 IMPORT CERTIFICATE IN FIREFOX

Access:

about:preferences#privacy

- Certificates → View certificates → Authorities → Import
- Select conectux.crt
- Check: "Trust this CA to identify sites"
- Restart browser.

---

## 📚 MKDOCS INSTALLATION


The installation of the Void Server Mkdocs tool was included on the channel's github page, which you can access here

https://github.com/VoidLinuxBR/tutoriais/blob/main/misc/tutorial-void-server-mkdocs.md

It will be useful in creating html content to export to Apache if you are not a frontend developer.

---

## ✅ CONCLUSION

- Apache with working HTTPS
- Redirect HTTP → HTTPS
- HTTP Basic Authentication
- Clean structure

🎯 **THAT'S ALL FOLKS**

