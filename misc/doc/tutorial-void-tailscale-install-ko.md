# Tailscale VPN sob Void Linux ;D

## 🎯 목표 - Tailscale 서비스 VPN에 연결된 네트워크 인터페이스를 업로드합니다.

## ✅ Tailscale 클라이언트 다운로드 및 압축 풀기

```bash
cd /tmp
```

```bash
wget https://pkgs.tailscale.com/stable/tailscale_1.90.9_amd64.tgz
```

```bash
tar xvf tailscale_1.90.9_amd64.tgz
```

## ✅ runit 서비스 디렉토리 생성

```bash
sudo mkdir -p /etc/sv/tailscaled
```

## ✅ 실행 파일 생성

```bash
sudo tee /etc/sv/tailscaled/run << 'EOF'
#!/bin/sh
exec /usr/local/bin/tailscaled \
  --state=/var/lib/tailscale/tailscaled.state \
  2>&1
EOF
```

## ✅ 실행 권한 부여:

```bash
sudo chmod +x /etc/sv/tailscaled/run
```

## ✅ (선택사항이지만 권장됨) 로그 스크립트 생성

```bash
sudo mkdir -p /etc/sv/tailscaled/log
```
```bash
sudo tee /etc/sv/tailscaled/log/run << 'EOF'
#!/bin/sh
exec svlogd -tt /var/log/tailscaled
EOF
```

```bash
sudo chmod +x /etc/sv/tailscaled/log/run
```

## ✅ 부팅 시 서비스 활성화

```bash
sudo ln -s /etc/sv/tailscaled /var/service/
```

## ✅ 지금 서비스 시작하기

```bash
sudo sv up tailscaled
```

```bash
sudo sv status tailscaled
```

## ✅ Tailscale 클라이언트 연결

## 서비스가 실행된 후에만 다음을 실행하십시오.

```bash
sudo tailscale up
```

## 🎉 이제 부팅 시 Tailscale이 자동으로 올라갑니다.

## 다시 시작한 후 확인하려면:

```bash
sudo tailscale status
```
```bash
sudo sv status tailscaled
```

## ✅ 어느 시점에서 서비스를 실행하고 Tailscale의 DNS를 캐시한 경우 다음 명령을 실행하여 자체 네트워크 DNS를 사용하도록 돌아갈 수 있습니다.

```bash
sudo tailscale up --accept-dns=false
```

## 그 후에는 이 새 구성이 상태에 저장되고 다음 부팅 시 DNS가 더 이상 수정되지 않습니다.

---

🎯 그게 전부입니다!

👉 문의: zerolies@disroot.org
👉 https://t.me/z3r0l135


