# 🌐 Void Linux(glibc)에 Apache 서버 업로드

## 🔐 HTTPS + 리디렉션 + HTTP 기본 인증

---

## 🧪 환경

- **Sistema Operacional:** Void Linux Server(glibc)
- **웹 서버:** Apache 2.4
- **시스템 초기화:** runit
- **내용:** 정적(MkDocs)
- **인증:** HTTP 기본(htpasswd)

---

## 1️⃣ 패키지 설치

루트 사용자로 로그인합니다.

```bash
su -
```

저장소, 패키지 및 운영 체제 업데이트:

```bash
xbps-install -Syu
```

Apache 및 유틸리티 설치

```bash
xbps-install apache openssl
```

---

## 2️⃣ APACHE 서비스(RUNIT) 활성화

Apache 서비스를 활성화합니다:

```bash
ln -s /etc/sv/apache /var/service/
```

상태 확인:

```bash
sv status apache
```

예상 결과:

```bash
run: apache: (pid 1045) 16s; run: log: (pid 1044) 16s
```

Void Linux의 기본 구성은 다음과 같습니다.

```bash
/etc/apache/httpd.conf
```

보조 설정은 다음과 같습니다.

```bash
/etc/apache/extra/
```

VirtualHosts는 다음과 같습니다.

```bash
/etc/apache/extra/httpd-vhosts.conf
```

SSL 파일은 다음 위치에 있습니다.

```bash
/etc/apache/extra/httpd-ssl.conf
```

---

## 3️ APAlic 서비스에 대한 RUT

```bash
curl -I http://192.168.70.100
```

예상 결과:

```bash
HTTP/1.1 200 OK
Date: Mon, 02 Feb 2026 18:25:24 GMT
Server: Apache
Content-Type: text/html;charset=ISO-8859-1
```

---

## 4️⃣ 모듈을 활성화하는 항목이 포함되어 있는지 확인하세요.

Void에서는 **존재하지 않습니다**:

- /etc/아파치2
- 사이트 이용 가능
- a2enmod, a2ensite

모듈은 httpd.conf를 통해 로드됩니다. 포함을 통해 이러한 줄이 존재하고 주석 처리가 해제되었는지 확인합니다.

```bash
vim /etc/apache/httpd.conf
```

VirtualHost와 SSL:

```bash
Include /etc/apache/extra/httpd-vhosts.conf
Include /etc/apache/extra/httpd-ssl.conf
```

필수 모듈:

```bash
LoadModule ssl_module libexec/apache/mod_ssl.so
LoadModule socache_shmcb_module libexec/apache/mod_socache_shmcb.so
LoadModule rewrite_module libexec/apache/mod_rewrite.so
LoadModule auth_basic_module libexec/apache/mod_auth_basic.so
LoadModule authn_file_module libexec/apache/mod_authn_file.so
```

이것이 없으면 HTTPS와 인증이 작동하지 않습니다!

---

## 5️⃣ 문서 루트 및 권한

기본 문서 루트:

```bash
/srv/www/apache
```

구조 만들기:

```bash
mkdir /srv/www/apache/publico
mkdir /srv/www/apache/aluno
```

사용자 및 그룹 권한 조정(_apache):

```bash
chown -R _apache:_apache /srv/www/apache/
chmod -R 755 /srv/www/apache/
```

확인하려면

```bash
ls -la /srv/www/apache/
```

예상되는 결과

```bash
total 16K
drwxrwxr-x 4 _apache _apache 4,0K fev  2 13:15 ./
drwxr-xr-x 3 root    root    4,0K fev  2 11:24 ../
drwxrwxr-x 2 _apache _apache 4,0K fev  2 13:15 aluno/
drwxrwxr-x 2 _apache _apache 4,0K fev  2 13:15 publico/
```

---

## 6️⃣ 자체 서명 SSL 인증서 생성

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/certs/conectux.key \
  -out /etc/ssl/certs/conectux.crt \
  -subj "/C=BR/ST=SP/L=LOCAL/O=Conectux/CN=192.168.122.100" \
  -addext "subjectAltName = IP:192.168.122.100, DNS:conectux.edu"
```

생성 후 다음을 검증합니다.

```bash
openssl x509 -in /etc/ssl/certs/conectux.crt -text -noout | grep -A2 "Subject Alternative Name"
```

---

## 7️⃣ HTTPS 설정 (가상호스트 443)

백업 생성 및 파일 편집:

```bash
cp /etc/apache/extra/http-ssl.conf{,.bkp}
vim /etc/apache/extra/httpd-ssl.conf
```

콘텐츠:

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

기본적으로 로그 경로는 존재하지 않는 경로를 가리킵니다.

```bash
ErrorLog "/var/log/apache/error.log"
CustomLog "/var/log/apache/access.log" combined
```

우리는 그것을 다음과 같이 올바르게 변경했습니다.

```bash
ErrorLog "/var/log/httpd/error.log"
CustomLog "/var/log/httpd/access.log" combined
```

---

## 8️⃣ HTTP + 리디렉션 파라 HTTPS

백업 생성 및 파일 편집:

```bash
cp /etc/apache/extra/httpd-vhosts.conf{,.bkp}
vim /etc/apache/extra/httpd-vhosts.conf
```

콘텐츠:

```bash
<VirtualHost *:80>
    ServerName 192.168.122.100
    DocumentRoot "/srv/www/apache"
    Redirect permanent / https://192.168.122.100/
</VirtualHost>
```

---

## 9️⃣ 사용자 생성(HTPASSWD)

보안 아카이브 생성:

```bash
> /etc/apache/matriculados
```

공개 액세스 외부의 인증된 사용자를 포함할 파일을 만듭니다.

```bash
chown root:_apache /etc/apache/matriculados
```

파일 권한 설정

```bash
chmod 640 /etc/apache/matriculados
```

제한된 페이지 콘텐츠에 액세스할 수 있는 권한이 있는 사용자 만들기
첫 번째 삽입에는 파일 생성이 포함되므로 -c 옵션이

```bash
htpasswd -c /etc/apache/matriculados aluno01
```

파일에서 사용자 이름과 비밀번호 생성을 확인합니다.

```bash
cat  /etc/apache/matriculados
```

결과

```bash
aluno01:$apr1$jCnS/66r$HrJoIImLRKzt46JsDx7n70
```

새 사용자 추가:
이제부터 항상 -c 옵션 없이

```bash
htpasswd /etc/apache/matriculados aluno02
```

---

## 📄 HTML 구문 유효성 검사

구문을 테스트하고 Apache 구성을 검증합니다.

```bash
apachectl configtest
```

예상 결과:

```bash
Syntax OK
```

아파치를 다시 시작하세요:

```bash
sv restart apache
```

---

## 🧪 테스트 및 문제 해결

열려 있는 포트 80 및 443을 확인합니다.

```bash
netstat -an | grep :80
```

결과

```bash
tcp6       0      0 :::80                   :::*                    LISTEN
```

```bash
netstat -an | grep :443
```

예상되는 결과

```bash
tcp6       0      0 :::443                  :::*                    LISTEN
```

버전 로그:

```bash
tail -f /var/log/apache/error.log
```

SSL 테스트(화면에 다양한 정보가 표시됨):

```bash
openssl s_client -connect 192.168.70.251:443
```

테스트 인증:

```bash
curl -I https://192.168.70.251/aluno/
```

예상 결과:

```bash
HTTP/1.1 401 Unauthorized
```

---

## HTML 콘텐츠 생성, 인증, 액세스 테스트

이미 /srv/www/apache/publico 및 /srv/www/apache/aluno에 경로를 만들고 권한을 설정했으므로

공개 영역 HTML 페이지를 만들었습니다.

```bash
vim /srv/www/apache/publico/index.html
```

파일 내용

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

학생 영역에 대한 HTML 페이지 생성뿐만 아니라

```bash
vim /var/www/apache/aluno/index.html
```

콘텐츠

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

Apache가 파일을 읽을 수 있는지 확인하십시오.

```bash
chown -R _apache:_apache /srv/www/apache
chmod -R 755 /srv/www/apache
```

컬을 사용하여 터미널 테스트를 수행할 수 있습니다.

```bash
curl -k https://192.168.122.100/publico/
```

결과 - HTML 콘텐츠

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

Resultado - HTTP/1.1 401 승인되지 않음

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

인증으로 테스트하고 릴리스를 볼 수 있습니다

```bash
curl -k -u aluno01 https://192.168.122.100/aluno/
```

공개 영역에서는 브라우저를 통해 콘텐츠에 접근하기 위해 비밀번호를 요구해서는 안 됩니다.

https://192.168.122.100/publico/

콘텐츠에 액세스하려면 자격 증명을 요청해야 하는 학생 영역과 달리

https://192.168.122.100/aluno/


---

## 🔐 Firefox에서 가져오기 인증서

입장:

정보:기본 설정#개인 정보 보호

- 인증서 → 인증서 보기 → 기관 → 가져오기
- conectux.crt를 선택하세요.
- 확인: "사이트를 식별하려면 이 CA를 신뢰하십시오."
- 브라우저를 다시 시작하세요.

---

## 📚 MKDOCS 설치


Void Server Mkdocs 도구 설치는 채널의 github 페이지에 포함되어 있으며 여기에서 액세스할 수 있습니다.

https://github.com/VoidLinuxBR/tutoriais/blob/main/misc/tutorial-void-server-mkdocs.md

프론트엔드 개발자가 아닌 경우 Apache로 내보낼 HTML 콘텐츠를 생성하는 데 유용합니다.

---

## ✅ 결론

- HTTPS가 작동하는 Apache
- HTTP → HTTPS 리디렉션
- HTTP 기본 인증
- 깔끔한 구조

🎯 **그게 전부입니다**

