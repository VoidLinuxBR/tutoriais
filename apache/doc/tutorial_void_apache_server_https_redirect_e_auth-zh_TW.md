# 🌐 在 Void Linux (glibc) 上上傳 Apache 服務器

## 🔐 HTTPS + 重定向 + HTTP 基本身份驗證

---

## 🧪 環境

- **操作系統：** Void Linux 服務器 (glibc)
- **網絡服務器：** Apache 2.4
- **初始化系統：** runit
- **內容：** 靜態 (MkDocs)
- **身份驗證：** HTTP 基本 (htpasswd)

---

## 1️⃣ 軟件包安裝

使用root用戶登錄：

```bash
su -
```

更新存儲庫、軟件包和操作系統：

```bash
xbps-install -Syu
```

安裝 Apache 和實用程序

```bash
xbps-install apache openssl
```

---

## 2️⃣激活APACHE服務（RUNIT）

啟用 Apache 服務：

```bash
ln -s /etc/sv/apache /var/service/
```

檢查狀態：

```bash
sv status apache
```

預期結果：

```bash
run: apache: (pid 1045) 16s; run: log: (pid 1044) 16s
```

在Void Linux中主要配置是：

```bash
/etc/apache/httpd.conf
```

輔助設置有：

```bash
/etc/apache/extra/
```

虛擬主機位於：

```bash
/etc/apache/extra/httpd-vhosts.conf
```

SSL 文件位於：

```bash
/etc/apache/extra/httpd-ssl.conf
```

---

## 3️ RUT 至亞太服務

```bash
curl -I http://192.168.70.100
```

預期結果：

```bash
HTTP/1.1 200 OK
Date: Mon, 02 Feb 2026 18:25:24 GMT
Server: Apache
Content-Type: text/html;charset=ISO-8859-1
```

---

## 4️⃣ 確認激活模塊的內容

在Void中，**不存在**：

- /etc/apache2
- 可用站點
- a2enmod, a2ensite

模塊通過 httpd.conf 加載。通過包含，驗證這些行是否存在並且未註釋

```bash
vim /etc/apache/httpd.conf
```

虛擬主機和 SSL：

```bash
Include /etc/apache/extra/httpd-vhosts.conf
Include /etc/apache/extra/httpd-ssl.conf
```

所需模塊：

```bash
LoadModule ssl_module libexec/apache/mod_ssl.so
LoadModule socache_shmcb_module libexec/apache/mod_socache_shmcb.so
LoadModule rewrite_module libexec/apache/mod_rewrite.so
LoadModule auth_basic_module libexec/apache/mod_auth_basic.so
LoadModule authn_file_module libexec/apache/mod_authn_file.so
```

如果沒有這個，HTTPS 和身份驗證將無法工作！

---

## 5️⃣ 文檔根目錄和權限

默認文檔根目錄：

```bash
/srv/www/apache
```

創建結構：

```bash
mkdir /srv/www/apache/publico
mkdir /srv/www/apache/aluno
```

調整用戶和組權限（_apache）：

```bash
chown -R _apache:_apache /srv/www/apache/
chmod -R 755 /srv/www/apache/
```

檢查

```bash
ls -la /srv/www/apache/
```

預期結果

```bash
total 16K
drwxrwxr-x 4 _apache _apache 4,0K fev  2 13:15 ./
drwxr-xr-x 3 root    root    4,0K fev  2 11:24 ../
drwxrwxr-x 2 _apache _apache 4,0K fev  2 13:15 aluno/
drwxrwxr-x 2 _apache _apache 4,0K fev  2 13:15 publico/
```

---

## 6️⃣ 創建自簽名 SSL 證書

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/certs/conectux.key \
  -out /etc/ssl/certs/conectux.crt \
  -subj "/C=BR/ST=SP/L=LOCAL/O=Conectux/CN=192.168.122.100" \
  -addext "subjectAltName = IP:192.168.122.100, DNS:conectux.edu"
```

生成後，驗證：

```bash
openssl x509 -in /etc/ssl/certs/conectux.crt -text -noout | grep -A2 "Subject Alternative Name"
```

---

## 7️⃣ HTTPS 設置（虛擬主機 443）

創建備份並編輯文件：

```bash
cp /etc/apache/extra/http-ssl.conf{,.bkp}
vim /etc/apache/extra/httpd-ssl.conf
```

內容：

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

注意，默認情況下，日誌路徑指向不存在的路徑

```bash
ErrorLog "/var/log/apache/error.log"
CustomLog "/var/log/apache/access.log" combined
```

我們正確地將其更改為

```bash
ErrorLog "/var/log/httpd/error.log"
CustomLog "/var/log/httpd/access.log" combined
```

---

## 8️⃣ HTTP + 重定向 PARA HTTPS

創建備份並編輯文件：

```bash
cp /etc/apache/extra/httpd-vhosts.conf{,.bkp}
vim /etc/apache/extra/httpd-vhosts.conf
```

內容：

```bash
<VirtualHost *:80>
    ServerName 192.168.122.100
    DocumentRoot "/srv/www/apache"
    Redirect permanent / https://192.168.122.100/
</VirtualHost>
```

---

## 9️⃣ 用戶創建（HTPASSWD）

創建安全存檔：

```bash
> /etc/apache/matriculados
```

創建將包含公共訪問之外的經過身份驗證的用戶的文件

```bash
chown root:_apache /etc/apache/matriculados
```

設置文件權限

```bash
chmod 640 /etc/apache/matriculados
```

創建有權訪問受限頁面內容的用戶
第一次插入涉及文件的創建，這就是為什麼使用 -c 選項

```bash
htpasswd -c /etc/apache/matriculados aluno01
```

驗證文件中用戶名和密碼的創建

```bash
cat  /etc/apache/matriculados
```

結果

```bash
aluno01:$apr1$jCnS/66r$HrJoIImLRKzt46JsDx7n70
```

添加新用戶：
從現在開始，始終不使用 -c 選項

```bash
htpasswd /etc/apache/matriculados aluno02
```

---

## 📄 驗證 HTML 語法

測試語法並驗證 Apache 配置：

```bash
apachectl configtest
```

預期結果：

```bash
Syntax OK
```

重新啟動阿帕奇：

```bash
sv restart apache
```

---

## 🧪 測試和故障排除

驗證開放端口 80 和 443：

```bash
netstat -an | grep :80
```

結果

```bash
tcp6       0      0 :::80                   :::*                    LISTEN
```

```bash
netstat -an | grep :443
```

預期結果

```bash
tcp6       0      0 :::443                  :::*                    LISTEN
```

版本日誌：

```bash
tail -f /var/log/apache/error.log
```

測試SSL（屏幕上會顯示各種信息）：

```bash
openssl s_client -connect 192.168.70.251:443
```

測試認證：

```bash
curl -I https://192.168.70.251/aluno/
```

預期結果：

```bash
HTTP/1.1 401 Unauthorized
```

---

## HTML 內容創建、身份驗證和訪問測試

由於我們已經在 /srv/www/apache/publico 和 /srv/www/apache/aluno 中創建了路徑並設置了權限

我們創建了公共區域 HTML 頁面

```bash
vim /srv/www/apache/publico/index.html
```

文件內容

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

以及為學生區域創建 HTML 頁面

```bash
vim /var/www/apache/aluno/index.html
```

內容

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

確保 Apache 可以讀取這些文件：

```bash
chown -R _apache:_apache /srv/www/apache
chmod -R 755 /srv/www/apache
```

可以使用curl進行終端測試

```bash
curl -k https://192.168.122.100/publico/
```

結果 - html 內容

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

結果 - HTTP/1.1 401 未經授權

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

您可以通過身份驗證進行測試並查看版本

```bash
curl -k -u aluno01 https://192.168.122.100/aluno/
```

通過瀏覽器，公共區域不應要求輸入密碼來訪問內容

辣椒_REF_0_辣椒

與學生區不同，學生區必須請求憑據才能訪問內容

辣椒_REF_0_辣椒


---

## 🔐 在 Firefox 中導入證書

使用權：

關於：首選項#隱私

- 證書 → 查看證書 → 權威機構 → 導入
- 選擇 conectux.crt
- 檢查：“信任此 CA 來識別站點”
- 重新啟動瀏覽器。

---

## 📚MKDOCS 安裝


Void Server Mkdocs 工具的安裝包含在該頻道的 github 頁面上，您可以在此處訪問該頁面

辣椒_REF_0_辣椒

如果您不是前端開發人員，那麼它對於創建要導出到 Apache 的 html 內容非常有用。

---

## ✅ 結論

- Apache 具有可用的 HTTPS
- 重定向 HTTP → HTTPS
- HTTP 基本身份驗證
- 結構簡潔

🎯 **這就是大家**

