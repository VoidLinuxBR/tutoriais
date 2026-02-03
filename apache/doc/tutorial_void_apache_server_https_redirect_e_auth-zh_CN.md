# 🌐 在 Void Linux (glibc) 上上传 Apache 服务器

## 🔐 HTTPS + 重定向 + HTTP 基本身份验证

---

## 🧪 环境

- **操作系统：** Void Linux 服务器 (glibc)
- **网络服务器：** Apache 2.4
- **初始化系统：** runit
- **内容：** 静态 (MkDocs)
- **身份验证：** HTTP 基本 (htpasswd)

---

## 1️⃣ 软件包安装

使用root用户登录：

```bash
su -
```

更新存储库、软件包和操作系统：

```bash
xbps-install -Syu
```

安装 Apache 和实用程序

```bash
xbps-install apache openssl
```

---

## 2️⃣激活APACHE服务（RUNIT）

启用 Apache 服务：

```bash
ln -s /etc/sv/apache /var/service/
```

检查状态：

```bash
sv status apache
```

预期结果：

```bash
run: apache: (pid 1045) 16s; run: log: (pid 1044) 16s
```

在Void Linux中主要配置是：

```bash
/etc/apache/httpd.conf
```

辅助设置有：

```bash
/etc/apache/extra/
```

虚拟主机位于：

```bash
/etc/apache/extra/httpd-vhosts.conf
```

SSL 文件位于：

```bash
/etc/apache/extra/httpd-ssl.conf
```

---

## 3️ RUT 至亚太服务

```bash
curl -I http://192.168.70.100
```

预期结果：

```bash
HTTP/1.1 200 OK
Date: Mon, 02 Feb 2026 18:25:24 GMT
Server: Apache
Content-Type: text/html;charset=ISO-8859-1
```

---

## 4️⃣ 确认激活模块的内容

在Void中，**不存在**：

- /etc/apache2
- 可用站点
- a2enmod, a2ensite

模块通过 httpd.conf 加载。通过包含，验证这些行是否存在并且未注释

```bash
vim /etc/apache/httpd.conf
```

虚拟主机和 SSL：

```bash
Include /etc/apache/extra/httpd-vhosts.conf
Include /etc/apache/extra/httpd-ssl.conf
```

所需模块：

```bash
LoadModule ssl_module libexec/apache/mod_ssl.so
LoadModule socache_shmcb_module libexec/apache/mod_socache_shmcb.so
LoadModule rewrite_module libexec/apache/mod_rewrite.so
LoadModule auth_basic_module libexec/apache/mod_auth_basic.so
LoadModule authn_file_module libexec/apache/mod_authn_file.so
```

如果没有这个，HTTPS 和身份验证将无法工作！

---

## 5️⃣ 文档根目录和权限

默认文档根目录：

```bash
/srv/www/apache
```

创建结构：

```bash
mkdir /srv/www/apache/publico
mkdir /srv/www/apache/aluno
```

调整用户和组权限（_apache）：

```bash
chown -R _apache:_apache /srv/www/apache/
chmod -R 755 /srv/www/apache/
```

检查

```bash
ls -la /srv/www/apache/
```

预期结果

```bash
total 16K
drwxrwxr-x 4 _apache _apache 4,0K fev  2 13:15 ./
drwxr-xr-x 3 root    root    4,0K fev  2 11:24 ../
drwxrwxr-x 2 _apache _apache 4,0K fev  2 13:15 aluno/
drwxrwxr-x 2 _apache _apache 4,0K fev  2 13:15 publico/
```

---

## 6️⃣ 创建自签名 SSL 证书

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/certs/conectux.key \
  -out /etc/ssl/certs/conectux.crt \
  -subj "/C=BR/ST=SP/L=LOCAL/O=Conectux/CN=192.168.122.100" \
  -addext "subjectAltName = IP:192.168.122.100, DNS:conectux.edu"
```

生成后，验证：

```bash
openssl x509 -in /etc/ssl/certs/conectux.crt -text -noout | grep -A2 "Subject Alternative Name"
```

---

## 7️⃣ HTTPS 设置（虚拟主机 443）

创建备份并编辑文件：

```bash
cp /etc/apache/extra/http-ssl.conf{,.bkp}
vim /etc/apache/extra/httpd-ssl.conf
```

内容：

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

注意，默认情况下，日志路径指向不存在的路径

```bash
ErrorLog "/var/log/apache/error.log"
CustomLog "/var/log/apache/access.log" combined
```

我们正确地将其更改为

```bash
ErrorLog "/var/log/httpd/error.log"
CustomLog "/var/log/httpd/access.log" combined
```

---

## 8️⃣ HTTP + 重定向 PARA HTTPS

创建备份并编辑文件：

```bash
cp /etc/apache/extra/httpd-vhosts.conf{,.bkp}
vim /etc/apache/extra/httpd-vhosts.conf
```

内容：

```bash
<VirtualHost *:80>
    ServerName 192.168.122.100
    DocumentRoot "/srv/www/apache"
    Redirect permanent / https://192.168.122.100/
</VirtualHost>
```

---

## 9️⃣ 用户创建（HTPASSWD）

创建安全存档：

```bash
> /etc/apache/matriculados
```

创建将包含公共访问之外的经过身份验证的用户的文件

```bash
chown root:_apache /etc/apache/matriculados
```

设置文件权限

```bash
chmod 640 /etc/apache/matriculados
```

创建有权访问受限页面内容的用户
第一次插入涉及文件的创建，这就是为什么使用 -c 选项

```bash
htpasswd -c /etc/apache/matriculados aluno01
```

验证文件中用户名和密码的创建

```bash
cat  /etc/apache/matriculados
```

结果

```bash
aluno01:$apr1$jCnS/66r$HrJoIImLRKzt46JsDx7n70
```

添加新用户：
从现在开始，始终不使用 -c 选项

```bash
htpasswd /etc/apache/matriculados aluno02
```

---

## 📄 验证 HTML 语法

测试语法并验证 Apache 配置：

```bash
apachectl configtest
```

预期结果：

```bash
Syntax OK
```

重新启动阿帕奇：

```bash
sv restart apache
```

---

## 🧪 测试和故障排除

验证开放端口 80 和 443：

```bash
netstat -an | grep :80
```

结果

```bash
tcp6       0      0 :::80                   :::*                    LISTEN
```

```bash
netstat -an | grep :443
```

预期结果

```bash
tcp6       0      0 :::443                  :::*                    LISTEN
```

版本日志：

```bash
tail -f /var/log/apache/error.log
```

测试SSL（屏幕上会显示各种信息）：

```bash
openssl s_client -connect 192.168.70.251:443
```

测试认证：

```bash
curl -I https://192.168.70.251/aluno/
```

预期结果：

```bash
HTTP/1.1 401 Unauthorized
```

---

## HTML 内容创建、身份验证和访问测试

由于我们已经在 /srv/www/apache/publico 和 /srv/www/apache/aluno 中创建了路径并设置了权限

我们创建了公共区域 HTML 页面

```bash
vim /srv/www/apache/publico/index.html
```

文件内容

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

以及为学生区域创建 HTML 页面

```bash
vim /var/www/apache/aluno/index.html
```

内容

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

确保 Apache 可以读取这些文件：

```bash
chown -R _apache:_apache /srv/www/apache
chmod -R 755 /srv/www/apache
```

可以使用curl进行终端测试

```bash
curl -k https://192.168.122.100/publico/
```

结果 - html 内容

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

结果 - HTTP/1.1 401 未经授权

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

您可以通过身份验证进行测试并查看版本

```bash
curl -k -u aluno01 https://192.168.122.100/aluno/
```

通过浏览器，公共区域不应要求输入密码来访问内容

辣椒_REF_0_辣椒

与学生区不同，学生区必须请求凭据才能访问内容

辣椒_REF_0_辣椒


---

## 🔐 在 Firefox 中导入证书

使用权：

关于：首选项#隐私

- 证书 → 查看证书 → 权威机构 → 导入
- 选择 conectux.crt
- 检查：“信任此 CA 来识别站点”
- 重新启动浏览器。

---

## 📚MKDOCS 安装


Void Server Mkdocs 工具的安装包含在该频道的 github 页面上，您可以在此处访问该页面

辣椒_REF_0_辣椒

如果您不是前端开发人员，那么它对于创建要导出到 Apache 的 html 内容非常有用。

---

## ✅ 结论

- Apache 具有可用的 HTTPS
- 重定向 HTTP → HTTPS
- HTTP 基本身份验证
- 结构简洁

🎯 **这就是大家**

