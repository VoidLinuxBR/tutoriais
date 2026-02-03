# 🌐 Hochladen eines Apache-Servers auf Void Linux (glibc)

## 🔐 HTTPS + Redirect + HTTP-Basisauthentifizierung

---

## 🧪 UMWELT

- **Betriebssystem:** Void Linux Server (glibc)
- **Webserver:** Apache 2.4
- **Init-System:** runit
- **Inhalt:** Statisch (MkDocs)
- **Authentifizierung:** HTTP Basic (htpasswd)

---

## 1️⃣ INSTALLATION VON PAKETEN

Melden Sie sich mit dem Root-Benutzer an:

```bash
su -
```

Repositorys, Pakete und Betriebssystem aktualisieren:

```bash
xbps-install -Syu
```

Installieren Sie Apache und Dienstprogramme

```bash
xbps-install apache openssl
```

---

## 2️⃣ APACHE-DIENST (RUNIT) AKTIVIEREN

Aktivieren Sie den Apache-Dienst:

```bash
ln -s /etc/sv/apache /var/service/
```

Status prüfen:

```bash
sv status apache
```

Erwartetes Ergebnis:

```bash
run: apache: (pid 1045) 16s; run: log: (pid 1044) 16s
```

In Void Linux ist die Hauptkonfiguration:

```bash
/etc/apache/httpd.conf
```

Die Hilfseinstellungen sind:

```bash
/etc/apache/extra/
```

Die VirtualHosts in:

```bash
/etc/apache/extra/httpd-vhosts.conf
```

Und die SSL-Datei unter:

```bash
/etc/apache/extra/httpd-ssl.conf
```

---

## 3️ RUT TO APAlic SERVICE

```bash
curl -I http://192.168.70.100
```

Erwartetes Ergebnis:

```bash
HTTP/1.1 200 OK
Date: Mon, 02 Feb 2026 18:25:24 GMT
Server: Apache
Content-Type: text/html;charset=ISO-8859-1
```

---

## 4️⃣ BESTÄTIGEN SIE DIE INCLUDES, DIE DIE MODULE AKTIVIEREN

In Void, **existiert nicht**:

- /etc/apache2
- Websites verfügbar
- a2enmod, a2ensite

Module werden über httpd.conf geladen. Überprüfen Sie mithilfe von Includes, ob diese Zeilen vorhanden und unkommentiert sind

```bash
vim /etc/apache/httpd.conf
```

VirtualHosts und SSL:

```bash
Include /etc/apache/extra/httpd-vhosts.conf
Include /etc/apache/extra/httpd-ssl.conf
```

Erforderliche Module:

```bash
LoadModule ssl_module libexec/apache/mod_ssl.so
LoadModule socache_shmcb_module libexec/apache/mod_socache_shmcb.so
LoadModule rewrite_module libexec/apache/mod_rewrite.so
LoadModule auth_basic_module libexec/apache/mod_auth_basic.so
LoadModule authn_file_module libexec/apache/mod_authn_file.so
```

Ohne dies funktionieren HTTPS und Authentifizierung nicht!

---

## 5️⃣ DOKUMENTENWURZEL UND BERECHTIGUNGEN

Standard-Dokumentenstamm:

```bash
/srv/www/apache
```

Struktur erstellen:

```bash
mkdir /srv/www/apache/publico
mkdir /srv/www/apache/aluno
```

Benutzer- und Gruppenberechtigungen anpassen (_apache):

```bash
chown -R _apache:_apache /srv/www/apache/
chmod -R 755 /srv/www/apache/
```

Zur Überprüfung

```bash
ls -la /srv/www/apache/
```

Erwartetes Ergebnis

```bash
total 16K
drwxrwxr-x 4 _apache _apache 4,0K fev  2 13:15 ./
drwxr-xr-x 3 root    root    4,0K fev  2 11:24 ../
drwxrwxr-x 2 _apache _apache 4,0K fev  2 13:15 aluno/
drwxrwxr-x 2 _apache _apache 4,0K fev  2 13:15 publico/
```

---

## 6️⃣ ERSTELLUNG EINES SELBSTSIGNIERTEN SSL-ZERTIFIKATS

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/certs/conectux.key \
  -out /etc/ssl/certs/conectux.crt \
  -subj "/C=BR/ST=SP/L=LOCAL/O=Conectux/CN=192.168.122.100" \
  -addext "subjectAltName = IP:192.168.122.100, DNS:conectux.edu"
```

Validieren Sie nach dem Generieren Folgendes:

```bash
openssl x509 -in /etc/ssl/certs/conectux.crt -text -noout | grep -A2 "Subject Alternative Name"
```

---

## 7️⃣ HTTPS-SETUP (VIRTUALHOST 443)

Backup erstellen und Datei bearbeiten:

```bash
cp /etc/apache/extra/http-ssl.conf{,.bkp}
vim /etc/apache/extra/httpd-ssl.conf
```

Inhalt:

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

Beachten Sie, dass der Protokollpfad standardmäßig auf einen nicht vorhandenen Pfad verweist

```bash
ErrorLog "/var/log/apache/error.log"
CustomLog "/var/log/apache/access.log" combined
```

Wir haben es richtigerweise in geändert

```bash
ErrorLog "/var/log/httpd/error.log"
CustomLog "/var/log/httpd/access.log" combined
```

---

## 8️⃣ HTTP + REDIRECT PARA HTTPS

Backup erstellen und Datei bearbeiten:

```bash
cp /etc/apache/extra/httpd-vhosts.conf{,.bkp}
vim /etc/apache/extra/httpd-vhosts.conf
```

Inhalt:

```bash
<VirtualHost *:80>
    ServerName 192.168.122.100
    DocumentRoot "/srv/www/apache"
    Redirect permanent / https://192.168.122.100/
</VirtualHost>
```

---

## 9️⃣ BENUTZERERSTELLUNG (HTPASSWD)

Sicheres Archiv erstellen:

```bash
> /etc/apache/matriculados
```

Erstellen Sie die Datei, die authentifizierte Benutzer außerhalb des öffentlichen Zugriffs enthält

```bash
chown root:_apache /etc/apache/matriculados
```

Dateiberechtigung festlegen

```bash
chmod 640 /etc/apache/matriculados
```

Erstellen Sie Benutzer mit der Berechtigung, auf eingeschränkte Seiteninhalte zuzugreifen
Beim ersten Einfügen wird die Datei erstellt, weshalb die Option -c verwendet wird

```bash
htpasswd -c /etc/apache/matriculados aluno01
```

Überprüfen Sie die Erstellung des Benutzernamens und Passworts in der Datei

```bash
cat  /etc/apache/matriculados
```

Ergebnis

```bash
aluno01:$apr1$jCnS/66r$HrJoIImLRKzt46JsDx7n70
```

Neue Benutzer hinzufügen:
Von nun an immer ohne die Option -c

```bash
htpasswd /etc/apache/matriculados aluno02
```

---

## 📄 VALIDIEREN SIE DIE HTML-SYNTAX

Testen Sie die Syntax und validieren Sie die Apache-Konfiguration:

```bash
apachectl configtest
```

Erwartetes Ergebnis:

```bash
Syntax OK
```

Apache neu starten:

```bash
sv restart apache
```

---

## 🧪 TESTS UND FEHLERBEHEBUNG

Validieren Sie die offenen Ports 80 und 443:

```bash
netstat -an | grep :80
```

Ergebnis

```bash
tcp6       0      0 :::80                   :::*                    LISTEN
```

```bash
netstat -an | grep :443
```

Erwartetes Ergebnis

```bash
tcp6       0      0 :::443                  :::*                    LISTEN
```

Ver-Protokolle:

```bash
tail -f /var/log/apache/error.log
```

Testen Sie SSL (es werden verschiedene Informationen auf dem Bildschirm angezeigt):

```bash
openssl s_client -connect 192.168.70.251:443
```

Testauthentifizierung:

```bash
curl -I https://192.168.70.251/aluno/
```

Erwartetes Ergebnis:

```bash
HTTP/1.1 401 Unauthorized
```

---

## Erstellung und Authentifizierung von HTML-Inhalten sowie Zugriffstests

Da wir bereits die Pfade erstellt und Berechtigungen in /srv/www/apache/publico und /srv/www/apache/aluno festgelegt haben

Wir haben die HTML-Seite für den öffentlichen Bereich erstellt

```bash
vim /srv/www/apache/publico/index.html
```

Dateiinhalte

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

Sowie die Erstellung der HTML-Seite für den Studentenbereich

```bash
vim /var/www/apache/aluno/index.html
```

Inhalt

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

Stellen Sie sicher, dass Apache die Dateien lesen kann:

```bash
chown -R _apache:_apache /srv/www/apache
chmod -R 755 /srv/www/apache
```

Terminaltests können mit Curl durchgeführt werden

```bash
curl -k https://192.168.122.100/publico/
```

Ergebnis – HTML-Inhalt

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

Ergebnis – HTTP/1.1 401 Nicht autorisiert

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

Sie können mit Authentifizierung testen und die Version sehen

```bash
curl -k -u aluno01 https://192.168.122.100/aluno/
```

Über den Browser sollte der öffentliche Bereich nicht nach einem Passwort fragen, um auf den Inhalt zuzugreifen

https://192.168.122.100/publico/

Im Gegensatz zum Studentenbereich, der Anmeldeinformationen anfordern muss, um auf Inhalte zugreifen zu können

https://192.168.122.100/aluno/


---

## 🔐 ZERTIFIKAT IN FIREFOX IMPORTIEREN

Zugang:

about:preferences#privacy

- Zertifikate → Zertifikate anzeigen → Behörden → Importieren
- Wählen Sie conectux.crt
- Überprüfen Sie: „Dieser Zertifizierungsstelle vertrauen, um Websites zu identifizieren“
- Starten Sie den Browser neu.

---

## 📚 MKDOCS-INSTALLATION


Die Installation des Void Server Mkdocs-Tools war auf der Github-Seite des Kanals enthalten, auf die Sie hier zugreifen können

https://github.com/VoidLinuxBR/tutoriais/blob/main/misc/tutorial-void-server-mkdocs.md

Wenn Sie kein Frontend-Entwickler sind, ist dies bei der Erstellung von HTML-Inhalten für den Export nach Apache hilfreich.

---

## ✅ FAZIT

- Apache mit funktionierendem HTTPS
- Leiten Sie HTTP → HTTPS um
- HTTP-Basisauthentifizierung
- Saubere Struktur

🎯 **DAS IST ALLES, LEUTE**

