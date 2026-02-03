# 🌐 Carga de un servidor Apache en Void Linux (glibc)

## 🔐 HTTPS + Redireccionamiento + Autenticación básica HTTP

---

## 🧪 MEDIO AMBIENTE

- **Sistema Operacional:** Void Linux Server (glibc)
- **Servidor Web:** Apache 2.4
- **Sistema de inicio:** ejecutar
- **Contenido:** Estático (MkDocs)
- **Autenticación:** HTTP Básico (htpasswd)

---

## 1️⃣ INSTALACIÓN DE PAQUETES

Inicie sesión con el usuario root:

```bash
su -
```

Actualizar repositorios, paquetes y sistema operativo:

```bash
xbps-install -Syu
```

Instalar Apache y utilidades

```bash
xbps-install apache openssl
```

---

## 2️⃣ ACTIVAR EL SERVICIO APACHE (RUNIT)

Habilite el servicio Apache:

```bash
ln -s /etc/sv/apache /var/service/
```

Verificar estado:

```bash
sv status apache
```

Resultado esperado:

```bash
run: apache: (pid 1045) 16s; run: log: (pid 1044) 16s
```

En Void Linux la configuración principal es:

```bash
/etc/apache/httpd.conf
```

Las configuraciones auxiliares son:

```bash
/etc/apache/extra/
```

Los VirtualHosts en:

```bash
/etc/apache/extra/httpd-vhosts.conf
```

Y el archivo SSL en:

```bash
/etc/apache/extra/httpd-ssl.conf
```

---

## 3️ RUTA AL SERVICIO APÁlico

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

## 4️⃣ CONFIRMA LOS INCLUYE QUE ACTIVAN LOS MÓDULOS

En Void, **no existe**:

- /etc/apache2
- sitios disponibles
- a2enmod, a2ensite

Los módulos se cargan a través de httpd.conf. a través de include, validar que estas líneas existan y estén descomentadas

```bash
vim /etc/apache/httpd.conf
```

VirtualHosts y SSL:

```bash
Include /etc/apache/extra/httpd-vhosts.conf
Include /etc/apache/extra/httpd-ssl.conf
```

Módulos requeridos:

```bash
LoadModule ssl_module libexec/apache/mod_ssl.so
LoadModule socache_shmcb_module libexec/apache/mod_socache_shmcb.so
LoadModule rewrite_module libexec/apache/mod_rewrite.so
LoadModule auth_basic_module libexec/apache/mod_auth_basic.so
LoadModule authn_file_module libexec/apache/mod_authn_file.so
```

Sin esto, HTTPS y la autenticación no funcionan.

---

## 5️⃣ RAÍZ DEL DOCUMENTO Y PERMISOS

Raíz del documento predeterminada:

```bash
/srv/www/apache
```

Crear estructura:

```bash
mkdir /srv/www/apache/publico
mkdir /srv/www/apache/aluno
```

Ajustar permisos de usuarios y grupos (_apache):

```bash
chown -R _apache:_apache /srv/www/apache/
chmod -R 755 /srv/www/apache/
```

para comprobar

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

## 6️⃣ CREACIÓN DE CERTIFICADO SSL AUTOFIRMADO

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/certs/conectux.key \
  -out /etc/ssl/certs/conectux.crt \
  -subj "/C=BR/ST=SP/L=LOCAL/O=Conectux/CN=192.168.122.100" \
  -addext "subjectAltName = IP:192.168.122.100, DNS:conectux.edu"
```

Después de generar validar:

```bash
openssl x509 -in /etc/ssl/certs/conectux.crt -text -noout | grep -A2 "Subject Alternative Name"
```

---

## 7️⃣ CONFIGURACIÓN HTTPS (VIRTUALHOST 443)

Crear copia de seguridad y editar archivo:

```bash
cp /etc/apache/extra/http-ssl.conf{,.bkp}
vim /etc/apache/extra/httpd-ssl.conf
```

Contenido:

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

TENGA EN CUENTA que la ruta del registro, de forma predeterminada, apunta a una ruta inexistente

```bash
ErrorLog "/var/log/apache/error.log"
CustomLog "/var/log/apache/access.log" combined
```

Lo cambiamos correctamente a

```bash
ErrorLog "/var/log/httpd/error.log"
CustomLog "/var/log/httpd/access.log" combined
```

---

## 8️⃣ HTTP + REDIRECCIÓN PARA HTTPS

Crear copia de seguridad y editar archivo:

```bash
cp /etc/apache/extra/httpd-vhosts.conf{,.bkp}
vim /etc/apache/extra/httpd-vhosts.conf
```

Contenido:

```bash
<VirtualHost *:80>
    ServerName 192.168.122.100
    DocumentRoot "/srv/www/apache"
    Redirect permanent / https://192.168.122.100/
</VirtualHost>
```

---

## 9️⃣ CREACIÓN DE USUARIO (HTPASSWD)

Crear archivo seguro:

```bash
> /etc/apache/matriculados
```

Cree el archivo que contendrá los usuarios autenticados fuera del acceso público.

```bash
chown root:_apache /etc/apache/matriculados
```

Establecer permiso de archivo

```bash
chmod 640 /etc/apache/matriculados
```

Crear usuarios con permiso para acceder al contenido de la página restringida
La primera inserción implica la creación del archivo, por eso la opción -c

```bash
htpasswd -c /etc/apache/matriculados aluno01
```

Validar la creación del usuario y contraseña en el archivo

```bash
cat  /etc/apache/matriculados
```

Resultado

```bash
aluno01:$apr1$jCnS/66r$HrJoIImLRKzt46JsDx7n70
```

Agregar nuevos usuarios:
Siempre sin la opción -c, de ahora en adelante

```bash
htpasswd /etc/apache/matriculados aluno02
```

---

## 📄 VALIDAR SINTaxis HTML

Pruebe la sintaxis y valide la configuración de Apache:

```bash
apachectl configtest
```

Resultado esperado:

```bash
Syntax OK
```

Reiniciar Apache:

```bash
sv restart apache
```

---

## 🧪 TESTÍCULOS Y SOLUCIÓN DE PROBLEMAS

Validar los puertos abiertos 80 y 443:

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

Pruebe SSL (mostrará diversa información en la pantalla):

```bash
openssl s_client -connect 192.168.70.251:443
```

Autenticación de prueba:

```bash
curl -I https://192.168.70.251/aluno/
```

Resultado esperado:

```bash
HTTP/1.1 401 Unauthorized
```

---

## CREACIÓN DE CONTENIDO HTML Y AUTENTICACIÓN Y PRUEBAS DE ACCESO

Como ya hemos creado las rutas y establecido permisos en /srv/www/apache/publico y /srv/www/apache/aluno

Creamos la página HTML del Área Pública.

```bash
vim /srv/www/apache/publico/index.html
```

Contenido del archivo

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

Además de crear la página HTML para el área de Estudiantes.

```bash
vim /var/www/apache/aluno/index.html
```

Contenido

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

Asegúrese de que Apache pueda leer los archivos:

```bash
chown -R _apache:_apache /srv/www/apache
chmod -R 755 /srv/www/apache
```

Las pruebas terminales se pueden realizar usando curl

```bash
curl -k https://192.168.122.100/publico/
```

Resultado: contenido html

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

Resultado - HTTP/1.1 401 No autorizado

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

Puedes probar con autenticación y ver el lanzamiento.

```bash
curl -k -u aluno01 https://192.168.122.100/aluno/
```

A través del navegador, el Área Pública no debe solicitar contraseña para acceder al contenido

CHILE_REF_0_CHILI

A diferencia del Área de Estudiantes, que debe solicitar credenciales para acceder a los contenidos

CHILE_REF_0_CHILI


---

## 🔐 CERTIFICADO DE IMPORTACIÓN EN FIREFOX

Acceso:

acerca de:preferencias#privacidad

- Certificados → Ver certificados → Autoridades → Importar
- Seleccione conectarux.crt
- Marque: "Confíe en esta CA para identificar sitios"
- Reiniciar navegador.

---

## 📚 INSTALACIÓN DE MKDOCS


La instalación de la herramienta Void Server Mkdocs se incluyó en la página de github del canal, a la que puedes acceder aquí.

CHILE_REF_0_CHILI

Será útil para crear contenido html para exportar a Apache si no es un desarrollador frontend.

---

## ✅ CONCLUSIÓN

- Apache con HTTPS funcional
- Redirigir HTTP → HTTPS
- Autenticación básica HTTP
- Estructura limpia

🎯 **ESO ES TODO AMIGOS**

