# 🌐 Void Linux (glibc) での Apache サーバーのアップロード

## 🔐 HTTPS + リダイレクト + HTTP 基本認証

---

## 🧪環境

- **運用システム:** ボイド Linux サーバー (glibc)
- **Web サーバー:** Apache 2.4
- **初期システム:** runit
- **コンテンツ:** 静的 (MkDocs)
- **認証:** HTTP 基本 (htpasswd)

---

## 1️⃣ パッケージのインストール

root ユーザーでログインします。

```bash
su -
```

リポジトリ、パッケージ、オペレーティング システムを更新します。

```bash
xbps-install -Syu
```

Apache とユーティリティをインストールする

```bash
xbps-install apache openssl
```

---

## 2️⃣ APache サービスをアクティブ化します (RUNIT)

Apache サービスを有効にします。

```bash
ln -s /etc/sv/apache /var/service/
```

ステータスを確認します:

```bash
sv status apache
```

期待される結果:

```bash
run: apache: (pid 1045) 16s; run: log: (pid 1044) 16s
```

Void Linux の主な構成は次のとおりです。

```bash
/etc/apache/httpd.conf
```

補助設定は次のとおりです。

```bash
/etc/apache/extra/
```

VirtualHost の場所:

```bash
/etc/apache/extra/httpd-vhosts.conf
```

SSL ファイルは次の場所にあります。

```bash
/etc/apache/extra/httpd-ssl.conf
```

---

## 3️ APAlicサービスへのマンネリ

```bash
curl -I http://192.168.70.100
```

期待される結果:

```bash
HTTP/1.1 200 OK
Date: Mon, 02 Feb 2026 18:25:24 GMT
Server: Apache
Content-Type: text/html;charset=ISO-8859-1
```

---

## 4️⃣ モジュールをアクティブ化するインクルードを確認する

Void には **存在しません**:

- /etc/apache2
- 利用可能なサイト
- a2enmod、a2ensite

モジュールは httpd.conf 経由でロードされます。 include を通じて、これらの行が存在し、コメント化されていないことを検証します。

```bash
vim /etc/apache/httpd.conf
```

仮想ホストとSSL:

```bash
Include /etc/apache/extra/httpd-vhosts.conf
Include /etc/apache/extra/httpd-ssl.conf
```

必要なモジュール:

```bash
LoadModule ssl_module libexec/apache/mod_ssl.so
LoadModule socache_shmcb_module libexec/apache/mod_socache_shmcb.so
LoadModule rewrite_module libexec/apache/mod_rewrite.so
LoadModule auth_basic_module libexec/apache/mod_auth_basic.so
LoadModule authn_file_module libexec/apache/mod_authn_file.so
```

これがないと、HTTPS と認証は機能しません。

---

## 5️⃣ ルートと権限を文書化する

デフォルトのドキュメントルート:

```bash
/srv/www/apache
```

構造を作成します。

```bash
mkdir /srv/www/apache/publico
mkdir /srv/www/apache/aluno
```

ユーザーとグループの権限を調整します (_apache):

```bash
chown -R _apache:_apache /srv/www/apache/
chmod -R 755 /srv/www/apache/
```

確認するには

```bash
ls -la /srv/www/apache/
```

期待される結果

```bash
total 16K
drwxrwxr-x 4 _apache _apache 4,0K fev  2 13:15 ./
drwxr-xr-x 3 root    root    4,0K fev  2 11:24 ../
drwxrwxr-x 2 _apache _apache 4,0K fev  2 13:15 aluno/
drwxrwxr-x 2 _apache _apache 4,0K fev  2 13:15 publico/
```

---

## 6️⃣ 自己署名SSL証明書の作成

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/certs/conectux.key \
  -out /etc/ssl/certs/conectux.crt \
  -subj "/C=BR/ST=SP/L=LOCAL/O=Conectux/CN=192.168.122.100" \
  -addext "subjectAltName = IP:192.168.122.100, DNS:conectux.edu"
```

生成後、以下を検証します。

```bash
openssl x509 -in /etc/ssl/certs/conectux.crt -text -noout | grep -A2 "Subject Alternative Name"
```

---

## 7️⃣ HTTPS セットアップ (仮想ホスト 443)

バックアップを作成してファイルを編集します。

```bash
cp /etc/apache/extra/http-ssl.conf{,.bkp}
vim /etc/apache/extra/httpd-ssl.conf
```

コンテンツ：

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

デフォルトでは、ログ パスは存在しないパスを指していることに注意してください。

```bash
ErrorLog "/var/log/apache/error.log"
CustomLog "/var/log/apache/access.log" combined
```

正しくは次のように変更しました

```bash
ErrorLog "/var/log/httpd/error.log"
CustomLog "/var/log/httpd/access.log" combined
```

---

## 8️⃣ HTTP + リダイレクト PARA HTTPS

バックアップを作成してファイルを編集します。

```bash
cp /etc/apache/extra/httpd-vhosts.conf{,.bkp}
vim /etc/apache/extra/httpd-vhosts.conf
```

コンテンツ：

```bash
<VirtualHost *:80>
    ServerName 192.168.122.100
    DocumentRoot "/srv/www/apache"
    Redirect permanent / https://192.168.122.100/
</VirtualHost>
```

---

## 9️⃣ ユーザー作成 (HTPASSWD)

安全なアーカイブを作成します。

```bash
> /etc/apache/matriculados
```

パブリックアクセスの外部で認証されたユーザーを含むファイルを作成します。

```bash
chown root:_apache /etc/apache/matriculados
```

ファイル権限を設定する

```bash
chmod 640 /etc/apache/matriculados
```

制限されたページのコンテンツにアクセスする権限を持つユーザーを作成する
最初の挿入にはファイルの作成が含まれるため、-c オプションが使用されます。

```bash
htpasswd -c /etc/apache/matriculados aluno01
```

ファイル内のユーザー名とパスワードの作成を検証します。

```bash
cat  /etc/apache/matriculados
```

結果

```bash
aluno01:$apr1$jCnS/66r$HrJoIImLRKzt46JsDx7n70
```

新しいユーザーを追加します。
今後は常に -c オプションを使用しないでください

```bash
htpasswd /etc/apache/matriculados aluno02
```

---

## 📄 HTML 構文を検証する

構文をテストし、Apache 構成を検証します。

```bash
apachectl configtest
```

期待される結果:

```bash
Syntax OK
```

Apache を再起動します。

```bash
sv restart apache
```

---

## 🧪 テストとトラブルシューティング

開いているポート 80 と 443 を検証します。

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

期待される結果

```bash
tcp6       0      0 :::443                  :::*                    LISTEN
```

Ver ログ:

```bash
tail -f /var/log/apache/error.log
```

SSL をテストします (画面にさまざまな情報が表示されます)。

```bash
openssl s_client -connect 192.168.70.251:443
```

テスト認証:

```bash
curl -I https://192.168.70.251/aluno/
```

期待される結果:

```bash
HTTP/1.1 401 Unauthorized
```

---

## HTML コンテンツの作成、認証およびアクセス テスト

すでにパスを作成し、/srv/www/apache/publico と /srv/www/apache/aluno に権限を設定しているため、

パブリックエリアHTMLページを作成しました

```bash
vim /srv/www/apache/publico/index.html
```

ファイルの内容

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

Student エリアの HTML ページを作成するだけでなく、

```bash
vim /var/www/apache/aluno/index.html
```

コンテンツ

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

Apache がファイルを読み取れることを確認します。

```bash
chown -R _apache:_apache /srv/www/apache
chmod -R 755 /srv/www/apache
```

ターミナルテストはcurlを使用して実行できます

```bash
curl -k https://192.168.122.100/publico/
```

結果 - HTML コンテンツ

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

結果 - HTTP/1.1 401 未承認

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

認証を使用してテストし、リリースを確認できます

```bash
curl -k -u aluno01 https://192.168.122.100/aluno/
```

ブラウザを介して、パブリックエリアはコンテンツにアクセスするためにパスワードを要求してはなりません

https://192.168.122.100/publico/

学生エリアとは異なり、コンテンツにアクセスするには資格情報を要求する必要があります。

https://192.168.122.100/aluno/


---

## 🔐 Firefox に証明書をインポート

アクセス：

概要:設定#プライバシー

- 証明書 → 証明書の表示 → 認証局 → インポート
- conectux.crtを選択します
- 「サイトを識別するためにこの CA を信頼する」にチェックを入れます。
- ブラウザを再起動します。

---

## 📚 MKDOCS のインストール


Void Server Mkdocs ツールのインストールは、チャンネルの github ページに含まれており、ここからアクセスできます。

https://github.com/VoidLinuxBR/tutoriais/blob/main/misc/tutorial-void-server-mkdocs.md

フロントエンド開発者でない場合は、Apache にエクスポートする HTML コンテンツを作成するのに役立ちます。

---

## ✅ 結論

- HTTPS が動作する Apache
- HTTP → HTTPS にリダイレクト
- HTTP基本認証
- すっきりとした構造

🎯 **以上です**

