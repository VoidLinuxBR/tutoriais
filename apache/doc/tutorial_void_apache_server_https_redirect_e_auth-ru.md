# 🌐 Загрузка сервера Apache в Void Linux (glibc)

## 🔐 HTTPS + перенаправление + базовая аутентификация HTTP

---

## 🧪 ОКРУЖАЮЩАЯ СРЕДА

- **Операционная система:** Void Linux Server (glibc)
- **Веб-сервер:** Apache 2.4
- **Система инициализации:** runit
- **Содержимое:** Статическое (MkDocs).
- **Аутентификация:** Базовый HTTP (htpasswd)

---

## 1️⃣ УСТАНОВКА ПАКЕТОВ

Войдите под пользователем root:

```bash
su -
```

Обновите репозитории, пакеты и операционную систему:

```bash
xbps-install -Syu
```

Установите Apache и утилиты

```bash
xbps-install apache openssl
```

---

## 2️⃣ АКТИВИРОВАТЬ СЕРВИС APACHE (RUNIT)

Включите службу Apache:

```bash
ln -s /etc/sv/apache /var/service/
```

Проверить статус:

```bash
sv status apache
```

Ожидаемый результат:

```bash
run: apache: (pid 1045) 16s; run: log: (pid 1044) 16s
```

В Void Linux основная конфигурация такова:

```bash
/etc/apache/httpd.conf
```

Вспомогательные настройки:

```bash
/etc/apache/extra/
```

Виртуальные хосты в:

```bash
/etc/apache/extra/httpd-vhosts.conf
```

И файл SSL по адресу:

```bash
/etc/apache/extra/httpd-ssl.conf
```

---

## 3️ РУТ В АПАЛИЧЕСКИЙ СЕРВИС

```bash
curl -I http://192.168.70.100
```

Ожидаемый результат:

```bash
HTTP/1.1 200 OK
Date: Mon, 02 Feb 2026 18:25:24 GMT
Server: Apache
Content-Type: text/html;charset=ISO-8859-1
```

---

## 4️⃣ ПОДТВЕРЖДИТЕ ВКЛЮЧЕНИЯ, КОТОРЫЕ АКТИВИРУЮТ МОДУЛИ

В Void **не существует**:

- /etc/apache2
- сайты доступны
- a2enmod, a2ensite

Модули загружаются через httpd.conf. через включения убедитесь, что эти строки существуют и не закомментированы.

```bash
vim /etc/apache/httpd.conf
```

Виртуальные хосты и SSL:

```bash
Include /etc/apache/extra/httpd-vhosts.conf
Include /etc/apache/extra/httpd-ssl.conf
```

Необходимые модули:

```bash
LoadModule ssl_module libexec/apache/mod_ssl.so
LoadModule socache_shmcb_module libexec/apache/mod_socache_shmcb.so
LoadModule rewrite_module libexec/apache/mod_rewrite.so
LoadModule auth_basic_module libexec/apache/mod_auth_basic.so
LoadModule authn_file_module libexec/apache/mod_authn_file.so
```

Без этого HTTPS и аутентификация не будут работать!

---

## 5️⃣ КОРЕНЬ ДОКУМЕНТА И РАЗРЕШЕНИЯ

Корень документа по умолчанию:

```bash
/srv/www/apache
```

Создать структуру:

```bash
mkdir /srv/www/apache/publico
mkdir /srv/www/apache/aluno
```

Настройте права пользователя и группы (_apache):

```bash
chown -R _apache:_apache /srv/www/apache/
chmod -R 755 /srv/www/apache/
```

Чтобы проверить

```bash
ls -la /srv/www/apache/
```

Ожидаемый результат

```bash
total 16K
drwxrwxr-x 4 _apache _apache 4,0K fev  2 13:15 ./
drwxr-xr-x 3 root    root    4,0K fev  2 11:24 ../
drwxrwxr-x 2 _apache _apache 4,0K fev  2 13:15 aluno/
drwxrwxr-x 2 _apache _apache 4,0K fev  2 13:15 publico/
```

---

## 6️⃣ СОЗДАНИЕ САМОЗНАПИСАЮЩЕГО SSL-СЕРТИФИКАТА

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/certs/conectux.key \
  -out /etc/ssl/certs/conectux.crt \
  -subj "/C=BR/ST=SP/L=LOCAL/O=Conectux/CN=192.168.122.100" \
  -addext "subjectAltName = IP:192.168.122.100, DNS:conectux.edu"
```

После генерации проверьте:

```bash
openssl x509 -in /etc/ssl/certs/conectux.crt -text -noout | grep -A2 "Subject Alternative Name"
```

---

## 7️⃣ НАСТРОЙКА HTTPS (ВИРТУАЛЬНЫЙ ХОСТ 443)

Создайте резервную копию и отредактируйте файл:

```bash
cp /etc/apache/extra/http-ssl.conf{,.bkp}
vim /etc/apache/extra/httpd-ssl.conf
```

Содержание:

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

ОБРАТИТЕ ВНИМАНИЕ, что путь к журналу по умолчанию указывает на несуществующий путь.

```bash
ErrorLog "/var/log/apache/error.log"
CustomLog "/var/log/apache/access.log" combined
```

Мы правильно изменили его на

```bash
ErrorLog "/var/log/httpd/error.log"
CustomLog "/var/log/httpd/access.log" combined
```

---

## 8️⃣ HTTP + ПЕРЕНАПРАВЛЕНИЕ HTTPS

Создайте резервную копию и отредактируйте файл:

```bash
cp /etc/apache/extra/httpd-vhosts.conf{,.bkp}
vim /etc/apache/extra/httpd-vhosts.conf
```

Содержание:

```bash
<VirtualHost *:80>
    ServerName 192.168.122.100
    DocumentRoot "/srv/www/apache"
    Redirect permanent / https://192.168.122.100/
</VirtualHost>
```

---

## 9️⃣ СОЗДАНИЕ ПОЛЬЗОВАТЕЛЯ (HTPASSWD)

Создать безопасный архив:

```bash
> /etc/apache/matriculados
```

Создайте файл, который будет содержать аутентифицированных пользователей за пределами публичного доступа.

```bash
chown root:_apache /etc/apache/matriculados
```

Установить права доступа к файлу

```bash
chmod 640 /etc/apache/matriculados
```

Создайте пользователей с разрешением на доступ к ограниченному содержимому страницы.
Первая вставка предполагает создание файла, поэтому опция -c

```bash
htpasswd -c /etc/apache/matriculados aluno01
```

Подтвердите создание имени пользователя и пароля в файле.

```bash
cat  /etc/apache/matriculados
```

Результат

```bash
aluno01:$apr1$jCnS/66r$HrJoIImLRKzt46JsDx7n70
```

Добавьте новых пользователей:
С этого момента всегда без опции -c

```bash
htpasswd /etc/apache/matriculados aluno02
```

---

## 📄 ПРОВЕРКА СИНТАКСИКА HTML

Проверьте синтаксис и проверьте конфигурацию Apache:

```bash
apachectl configtest
```

Ожидаемый результат:

```bash
Syntax OK
```

Перезапустите Апач:

```bash
sv restart apache
```

---

## 🧪 ТЕСТЫ И УСТРАНЕНИЕ НЕИСПРАВНОСТЕЙ

Проверьте открытые порты 80 и 443:

```bash
netstat -an | grep :80
```

Результат

```bash
tcp6       0      0 :::80                   :::*                    LISTEN
```

```bash
netstat -an | grep :443
```

Ожидаемый результат

```bash
tcp6       0      0 :::443                  :::*                    LISTEN
```

Версные журналы:

```bash
tail -f /var/log/apache/error.log
```

Тестируем SSL (на экране будет отображаться различная информация):

```bash
openssl s_client -connect 192.168.70.251:443
```

Тестовая аутентификация:

```bash
curl -I https://192.168.70.251/aluno/
```

Ожидаемый результат:

```bash
HTTP/1.1 401 Unauthorized
```

---

## СОЗДАНИЕ HTML-КОНТЕНТА, АУТЕНТИФИКАЦИЯ И ТЕСТИРОВАНИЕ ДОСТУПА

Поскольку мы уже создали пути и установили разрешения в /srv/www/apache/publico и /srv/www/apache/aluno

Мы создали HTML-страницу публичной зоны.

```bash
vim /srv/www/apache/publico/index.html
```

Содержимое файла

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

А также создание HTML-страницы для области студентов.

```bash
vim /var/www/apache/aluno/index.html
```

Содержание

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

Убедитесь, что Apache может читать файлы:

```bash
chown -R _apache:_apache /srv/www/apache
chmod -R 755 /srv/www/apache
```

Терминальные тесты можно выполнить с помощью Curl

```bash
curl -k https://192.168.122.100/publico/
```

Результат - html-контент

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

Результат — HTTP/1.1 401 Неавторизованный

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

Вы можете протестировать аутентификацию и посмотреть релиз.

```bash
curl -k -u aluno01 https://192.168.122.100/aluno/
```

Через браузер Публичная зона не должна запрашивать пароль для доступа к контенту.

https://192.168.122.100/publico/

В отличие от студенческой зоны, которая должна запрашивать учетные данные для доступа к контенту.

https://192.168.122.100/aluno/


---

## 🔐 ИМПОРТ СЕРТИФИКАТА В FIREFOX

Доступ:

о: предпочтения#конфиденциальность

- Сертификаты → Просмотр сертификатов → Центры → Импорт
- Выберите conectux.crt.
- Установите флажок: «Доверять этому центру сертификации идентификацию сайтов».
- Перезапустите браузер.

---

## 📚 УСТАНОВКА MKDOCS


Установка инструмента Void Server Mkdocs была включена в страницу github канала, доступ к которой вы можете получить здесь.

https://github.com/VoidLinuxBR/tutoriais/blob/main/misc/tutorial-void-server-mkdocs.md

Это будет полезно при создании html-контента для экспорта в Apache, если вы не являетесь фронтенд-разработчиком.

---

## ✅ ЗАКЛЮЧЕНИЕ

- Apache с работающим HTTPS
- Перенаправление HTTP → HTTPS
- Базовая HTTP-аутентификация
- Чистая структура

🎯 **ВОТ ВСЕ, ЛЮДИ**

