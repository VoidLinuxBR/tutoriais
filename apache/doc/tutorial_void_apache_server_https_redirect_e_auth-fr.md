# 🌐 Télécharger un serveur Apache sur Void Linux (glibc)

## 🔐 HTTPS + Redirection + Authentification HTTP de base

---

## 🧪 ENVIRONNEMENT

- **Sistema Operacional :** Void Linux Server (glibc)
- **Serveur Web :** Apache 2.4
- **Système d'initialisation :** exécution
- **Contenu :** Statique (MkDocs)
- **Authentification :** HTTP Basic (htpasswd)

---

## 1️⃣ INSTALLATION DES FORFAITS

Connectez-vous avec l'utilisateur root :

```bash
su -
```

Mettre à jour les référentiels, les packages et le système d'exploitation :

```bash
xbps-install -Syu
```

Installer Apache et les utilitaires

```bash
xbps-install apache openssl
```

---

## 2️⃣ ACTIVER LE SERVICE APACHE (RUNIT)

Activez le service Apache :

```bash
ln -s /etc/sv/apache /var/service/
```

Vérifier l'état :

```bash
sv status apache
```

Résultat attendu :

```bash
run: apache: (pid 1045) 16s; run: log: (pid 1044) 16s
```

Dans Void Linux, la configuration principale est :

```bash
/etc/apache/httpd.conf
```

Les paramètres auxiliaires sont :

```bash
/etc/apache/extra/
```

Les hôtes virtuels dans :

```bash
/etc/apache/extra/httpd-vhosts.conf
```

Et le fichier SSL à l'adresse :

```bash
/etc/apache/extra/httpd-ssl.conf
```

---

## 3️ RUT AU SERVICE APALique

```bash
curl -I http://192.168.70.100
```

Résultat attendu :

```bash
HTTP/1.1 200 OK
Date: Mon, 02 Feb 2026 18:25:24 GMT
Server: Apache
Content-Type: text/html;charset=ISO-8859-1
```

---

## 4️⃣ CONFIRMER LES INCLUS QUI ACTIVENT LES MODULES

Dans Void, **n'existe pas** :

- /etc/apache2
- sites-disponibles
- a2enmod, a2ensite

Les modules sont chargés via httpd.conf. via des inclusions, valider que ces lignes existent et ne sont pas commentées

```bash
vim /etc/apache/httpd.conf
```

Hôtes virtuels et SSL :

```bash
Include /etc/apache/extra/httpd-vhosts.conf
Include /etc/apache/extra/httpd-ssl.conf
```

Modules requis :

```bash
LoadModule ssl_module libexec/apache/mod_ssl.so
LoadModule socache_shmcb_module libexec/apache/mod_socache_shmcb.so
LoadModule rewrite_module libexec/apache/mod_rewrite.so
LoadModule auth_basic_module libexec/apache/mod_auth_basic.so
LoadModule authn_file_module libexec/apache/mod_authn_file.so
```

Sans cela, HTTPS et l'authentification ne fonctionnent pas !

---

## 5️⃣ RACINE DU DOCUMENT ET AUTORISATIONS

Racine du document par défaut :

```bash
/srv/www/apache
```

Créer une structure :

```bash
mkdir /srv/www/apache/publico
mkdir /srv/www/apache/aluno
```

Ajustez les autorisations des utilisateurs et des groupes (_apache) :

```bash
chown -R _apache:_apache /srv/www/apache/
chmod -R 755 /srv/www/apache/
```

Pour vérifier

```bash
ls -la /srv/www/apache/
```

Résultat attendu

```bash
total 16K
drwxrwxr-x 4 _apache _apache 4,0K fev  2 13:15 ./
drwxr-xr-x 3 root    root    4,0K fev  2 11:24 ../
drwxrwxr-x 2 _apache _apache 4,0K fev  2 13:15 aluno/
drwxrwxr-x 2 _apache _apache 4,0K fev  2 13:15 publico/
```

---

## 6️⃣ CRÉATION DE CERTIFICAT SSL AUTO-SIGNÉ

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/certs/conectux.key \
  -out /etc/ssl/certs/conectux.crt \
  -subj "/C=BR/ST=SP/L=LOCAL/O=Conectux/CN=192.168.122.100" \
  -addext "subjectAltName = IP:192.168.122.100, DNS:conectux.edu"
```

Après génération, validez :

```bash
openssl x509 -in /etc/ssl/certs/conectux.crt -text -noout | grep -A2 "Subject Alternative Name"
```

---

## 7️⃣ CONFIGURATION HTTPS (VIRTUALHOST 443)

Créer une sauvegarde et modifier un fichier :

```bash
cp /etc/apache/extra/http-ssl.conf{,.bkp}
vim /etc/apache/extra/httpd-ssl.conf
```

Contenu:

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

REMARQUE que le chemin du journal, par défaut, pointe vers un chemin inexistant

```bash
ErrorLog "/var/log/apache/error.log"
CustomLog "/var/log/apache/access.log" combined
```

Nous l'avons correctement changé en

```bash
ErrorLog "/var/log/httpd/error.log"
CustomLog "/var/log/httpd/access.log" combined
```

---

## 8️⃣ HTTP + REDIRECTION PAR HTTPS

Créer une sauvegarde et modifier un fichier :

```bash
cp /etc/apache/extra/httpd-vhosts.conf{,.bkp}
vim /etc/apache/extra/httpd-vhosts.conf
```

Contenu:

```bash
<VirtualHost *:80>
    ServerName 192.168.122.100
    DocumentRoot "/srv/www/apache"
    Redirect permanent / https://192.168.122.100/
</VirtualHost>
```

---

## 9️⃣ CRÉATION D'UTILISATEUR (HTPASSWD)

Créez une archive sécurisée :

```bash
> /etc/apache/matriculados
```

Créez le fichier qui contiendra les utilisateurs authentifiés en dehors de l'accès public

```bash
chown root:_apache /etc/apache/matriculados
```

Définir l'autorisation du fichier

```bash
chmod 640 /etc/apache/matriculados
```

Créer des utilisateurs avec l'autorisation d'accéder au contenu de la page restreinte
La première insertion implique la création du fichier, c'est pourquoi l'option -c

```bash
htpasswd -c /etc/apache/matriculados aluno01
```

Valider la création du nom d'utilisateur et du mot de passe dans le fichier

```bash
cat  /etc/apache/matriculados
```

Résultat

```bash
aluno01:$apr1$jCnS/66r$HrJoIImLRKzt46JsDx7n70
```

Ajouter de nouveaux utilisateurs :
Toujours sans l'option -c, à partir de maintenant

```bash
htpasswd /etc/apache/matriculados aluno02
```

---

## 📄 VALIDER LA SYNTAXE HTML

Testez la syntaxe et validez la configuration d'Apache :

```bash
apachectl configtest
```

Résultat attendu :

```bash
Syntax OK
```

Redémarrez Apache :

```bash
sv restart apache
```

---

## 🧪 TESTS ET DÉPANNAGE

Validez les ports ouverts 80 et 443 :

```bash
netstat -an | grep :80
```

Résultat

```bash
tcp6       0      0 :::80                   :::*                    LISTEN
```

```bash
netstat -an | grep :443
```

Résultat attendu

```bash
tcp6       0      0 :::443                  :::*                    LISTEN
```

Journaux de version :

```bash
tail -f /var/log/apache/error.log
```

Testez SSL (il affichera diverses informations à l'écran) :

```bash
openssl s_client -connect 192.168.70.251:443
```

Tester l'authentification :

```bash
curl -I https://192.168.70.251/aluno/
```

Résultat attendu :

```bash
HTTP/1.1 401 Unauthorized
```

---

## CRÉATION DE CONTENU HTML, AUTHENTIFICATION ET TEST D'ACCÈS

Puisque nous avons déjà créé les chemins et défini les autorisations dans /srv/www/apache/publico et /srv/www/apache/aluno

Nous avons créé la page HTML de l'espace public

```bash
vim /srv/www/apache/publico/index.html
```

Contenu du fichier

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

En plus de créer la page HTML de l'espace étudiant

```bash
vim /var/www/apache/aluno/index.html
```

Contenu

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

Assurez-vous qu'Apache peut lire les fichiers :

```bash
chown -R _apache:_apache /srv/www/apache
chmod -R 755 /srv/www/apache
```

Les tests terminaux peuvent être effectués à l'aide de curl

```bash
curl -k https://192.168.122.100/publico/
```

Résultat - contenu html

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

Résultat - HTTP/1.1 401 non autorisé

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

Vous pouvez tester avec authentification et voir la version

```bash
curl -k -u aluno01 https://192.168.122.100/aluno/
```

Via le navigateur, l'espace public ne doit pas demander de mot de passe pour accéder au contenu

https://192.168.122.100/publico/

Contrairement à l'Espace Étudiant, qui doit demander des identifiants pour accéder au contenu

https://192.168.122.100/aluno/


---

## 🔐 CERTIFICAT D'IMPORTATION DANS FIREFOX

Accéder:

à propos de : préférences#confidentialité

- Certificats → Afficher les certificats → Autorités → Importer
- Sélectionnez connectux.crt
- Cochez : "Faites confiance à cette autorité de certification pour identifier les sites"
- Redémarrez le navigateur.

---

## 📚 INSTALLATION DE MKDOCS


L'installation de l'outil Void Server Mkdocs a été incluse sur la page github de la chaîne, à laquelle vous pouvez accéder ici

https://github.com/VoidLinuxBR/tutoriais/blob/main/misc/tutorial-void-server-mkdocs.md

Cela sera utile pour créer du contenu HTML à exporter vers Apache si vous n'êtes pas un développeur frontend.

---

## ✅CONCLUSION

- Apache avec HTTPS fonctionnel
- Redirection HTTP → HTTPS
- Authentification HTTP de base
- Structure propre

🎯 **C'EST TOUS LES GENS**

