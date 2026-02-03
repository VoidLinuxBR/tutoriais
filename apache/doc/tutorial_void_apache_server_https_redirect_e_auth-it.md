# 🌐 Caricamento di un server Apache su Void Linux (glibc)

## 🔐 HTTPS + Reindirizzamento + Autenticazione di base HTTP

---

## 🧪AMBIENTE

- **Sistema operativo:** Void Linux Server (glibc)
- **Server Web:** Apache 2.4
- **Sistema di inizializzazione:** runit
- **Contenuto:** Statico (MkDocs)
- **Autenticazione:** HTTP Basic (htpasswd)

---

## 1️⃣ INSTALLAZIONE DEI PACCHETTI

Accedi con l'utente root:

```bash
su -
```

Aggiorna repository, pacchetti e sistema operativo:

```bash
xbps-install -Syu
```

Installa Apache e le utilità

```bash
xbps-install apache openssl
```

---

## 2️⃣ ATTIVA IL SERVIZIO APACHE (RUNIT)

Abilita il servizio Apache:

```bash
ln -s /etc/sv/apache /var/service/
```

Controlla lo stato:

```bash
sv status apache
```

Risultato atteso:

```bash
run: apache: (pid 1045) 16s; run: log: (pid 1044) 16s
```

In Void Linux la configurazione principale è:

```bash
/etc/apache/httpd.conf
```

Le impostazioni ausiliarie sono:

```bash
/etc/apache/extra/
```

Gli host virtuali in:

```bash
/etc/apache/extra/httpd-vhosts.conf
```

E il file SSL in:

```bash
/etc/apache/extra/httpd-ssl.conf
```

---

## 3️ RUT AL SERVIZIO APAlico

```bash
curl -I http://192.168.70.100
```

Risultato atteso:

```bash
HTTP/1.1 200 OK
Date: Mon, 02 Feb 2026 18:25:24 GMT
Server: Apache
Content-Type: text/html;charset=ISO-8859-1
```

---

## 4️⃣ CONFERMA GLI INCLUSI CHE ATTIVANO I MODULI

Nel Vuoto, **non esiste**:

- /etc/apache2
- siti disponibili
- a2enmod, a2ensite

I moduli vengono caricati tramite httpd.conf. tramite include, verifica che queste righe esistano e non siano commentate

```bash
vim /etc/apache/httpd.conf
```

Host virtuali e SSL:

```bash
Include /etc/apache/extra/httpd-vhosts.conf
Include /etc/apache/extra/httpd-ssl.conf
```

Moduli richiesti:

```bash
LoadModule ssl_module libexec/apache/mod_ssl.so
LoadModule socache_shmcb_module libexec/apache/mod_socache_shmcb.so
LoadModule rewrite_module libexec/apache/mod_rewrite.so
LoadModule auth_basic_module libexec/apache/mod_auth_basic.so
LoadModule authn_file_module libexec/apache/mod_authn_file.so
```

Senza questo, HTTPS e l'autenticazione non funzionano!

---

## 5️⃣ ROOT DEL DOCUMENTO E PERMESSI

Radice documento predefinita:

```bash
/srv/www/apache
```

Crea struttura:

```bash
mkdir /srv/www/apache/publico
mkdir /srv/www/apache/aluno
```

Modifica le autorizzazioni dell'utente e del gruppo (_apache):

```bash
chown -R _apache:_apache /srv/www/apache/
chmod -R 755 /srv/www/apache/
```

Per controllare

```bash
ls -la /srv/www/apache/
```

Risultato atteso

```bash
total 16K
drwxrwxr-x 4 _apache _apache 4,0K fev  2 13:15 ./
drwxr-xr-x 3 root    root    4,0K fev  2 11:24 ../
drwxrwxr-x 2 _apache _apache 4,0K fev  2 13:15 aluno/
drwxrwxr-x 2 _apache _apache 4,0K fev  2 13:15 publico/
```

---

## 6️⃣ CREAZIONE DI CERTIFICATO SSL AUTOFIRMATO

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/certs/conectux.key \
  -out /etc/ssl/certs/conectux.crt \
  -subj "/C=BR/ST=SP/L=LOCAL/O=Conectux/CN=192.168.122.100" \
  -addext "subjectAltName = IP:192.168.122.100, DNS:conectux.edu"
```

Dopo la generazione, convalida:

```bash
openssl x509 -in /etc/ssl/certs/conectux.crt -text -noout | grep -A2 "Subject Alternative Name"
```

---

## 7️⃣ CONFIGURAZIONE HTTPS (VIRTUALHOST 443)

Crea backup e modifica file:

```bash
cp /etc/apache/extra/http-ssl.conf{,.bkp}
vim /etc/apache/extra/httpd-ssl.conf
```

Contenuto:

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

NOTA che il percorso del registro, per impostazione predefinita, punta a un percorso inesistente

```bash
ErrorLog "/var/log/apache/error.log"
CustomLog "/var/log/apache/access.log" combined
```

L'abbiamo modificato correttamente in

```bash
ErrorLog "/var/log/httpd/error.log"
CustomLog "/var/log/httpd/access.log" combined
```

---

## 8️⃣ HTTP + REDIRECT PARA HTTPS

Crea backup e modifica file:

```bash
cp /etc/apache/extra/httpd-vhosts.conf{,.bkp}
vim /etc/apache/extra/httpd-vhosts.conf
```

Contenuto:

```bash
<VirtualHost *:80>
    ServerName 192.168.122.100
    DocumentRoot "/srv/www/apache"
    Redirect permanent / https://192.168.122.100/
</VirtualHost>
```

---

## 9️⃣ CREAZIONE UTENTE (HTPASSWD)

Crea archivio sicuro:

```bash
> /etc/apache/matriculados
```

Creare il file che conterrà gli utenti autenticati al di fuori dell'accesso pubblico

```bash
chown root:_apache /etc/apache/matriculados
```

Imposta i permessi del file

```bash
chmod 640 /etc/apache/matriculados
```

Crea utenti con autorizzazione per accedere ai contenuti della pagina con restrizioni
Il primo inserimento prevede la creazione del file, ecco perché l'opzione -c

```bash
htpasswd -c /etc/apache/matriculados aluno01
```

Convalidare la creazione del nome utente e della password nel file

```bash
cat  /etc/apache/matriculados
```

Risultato

```bash
aluno01:$apr1$jCnS/66r$HrJoIImLRKzt46JsDx7n70
```

Aggiungi nuovi utenti:
D'ora in poi sempre senza l'opzione -c

```bash
htpasswd /etc/apache/matriculados aluno02
```

---

## 📄 CONVALIDA SINTASSI HTML

Testare la sintassi e convalidare la configurazione di Apache:

```bash
apachectl configtest
```

Risultato atteso:

```bash
Syntax OK
```

Riavvia Apache:

```bash
sv restart apache
```

---

## 🧪 TEST E RISOLUZIONE DEI PROBLEMI

Convalida le porte aperte 80 e 443:

```bash
netstat -an | grep :80
```

Risultato

```bash
tcp6       0      0 :::80                   :::*                    LISTEN
```

```bash
netstat -an | grep :443
```

Risultato atteso

```bash
tcp6       0      0 :::443                  :::*                    LISTEN
```

Registri di versione:

```bash
tail -f /var/log/apache/error.log
```

Test SSL (visualizzerà varie informazioni sullo schermo):

```bash
openssl s_client -connect 192.168.70.251:443
```

Prova l'autenticazione:

```bash
curl -I https://192.168.70.251/aluno/
```

Risultato atteso:

```bash
HTTP/1.1 401 Unauthorized
```

---

## CREAZIONE E AUTENTICAZIONE DI CONTENUTI HTML E TEST D'ACCESSO

Poiché abbiamo già creato i percorsi e impostato le autorizzazioni in /srv/www/apache/publico e /srv/www/apache/aluno

Abbiamo creato la pagina HTML dell'Area Pubblica

```bash
vim /srv/www/apache/publico/index.html
```

Contenuto del file

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

Oltre a creare la pagina HTML per l'area Studenti

```bash
vim /var/www/apache/aluno/index.html
```

Contenuto

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

Assicurati che Apache possa leggere i file:

```bash
chown -R _apache:_apache /srv/www/apache
chmod -R 755 /srv/www/apache
```

I test terminali possono essere eseguiti utilizzando curl

```bash
curl -k https://192.168.122.100/publico/
```

Risultato: contenuto HTML

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

Risultato: HTTP/1.1 401 Non autorizzato

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

Puoi testare con l'autenticazione e vedere la versione

```bash
curl -k -u aluno01 https://192.168.122.100/aluno/
```

Attraverso il browser, l'Area Pubblica non deve richiedere la password per accedere ai contenuti

https://192.168.122.100/publico/

A differenza dell'Area Studenti, che deve richiedere le credenziali per accedere ai contenuti

https://192.168.122.100/aluno/


---

## 🔐 CERTIFICATO DI IMPORTAZIONE IN FIREFOX

Accesso:

informazioni su:preferenze#privacy

- Certificati → Visualizza certificati → Autorità → Importa
- Selezionare conectux.crt
- Seleziona: "Considera attendibile questa CA per identificare i siti"
- Riavvia il browser.

---

## 📚 INSTALLAZIONE MKDOCS


L'installazione dello strumento Void Server Mkdocs è stata inclusa nella pagina github del canale, a cui puoi accedere qui

https://github.com/VoidLinuxBR/tutoriais/blob/main/misc/tutorial-void-server-mkdocs.md

Sarà utile per creare contenuti html da esportare su Apache se non sei uno sviluppatore frontend.

---

## ✅CONCLUSIONE

- Apache con HTTPS funzionante
- Reindirizza HTTP → HTTPS
- Autenticazione di base HTTP
- Struttura pulita

🎯 **QUESTO È TUTTO RAGAZZE**

