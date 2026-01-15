# Tailscale VPN は Void Linux を泣きます ;D

## 🎯 目的 - Tailscale サービス VPN に接続されたネットワーク インターフェイスをアップロードします。

## ✅ Tailscale クライアントをダウンロードして解凍します。

```bash
cd /tmp
```

```bash
wget https://pkgs.tailscale.com/stable/tailscale_1.90.9_amd64.tgz
```

```bash
tar xvf tailscale_1.90.9_amd64.tgz
```

## ✅ runitサービスディレクトリを作成する

```bash
sudo mkdir -p /etc/sv/tailscaled
```

## ✅ 実行ファイルを作成する

```bash
sudo tee /etc/sv/tailscaled/run << 'EOF'
#!/bin/sh
exec /usr/local/bin/tailscaled \
  --state=/var/lib/tailscale/tailscaled.state \
  2>&1
EOF
```

## ✅ 実行権限を与えます:

```bash
sudo chmod +x /etc/sv/tailscaled/run
```

## ✅ (オプションですが推奨) ログスクリプトを作成します

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

## ✅ 起動時にサービスをアクティブ化する

```bash
sudo ln -s /etc/sv/tailscaled /var/service/
```

## ✅ 今すぐサービスを開始

```bash
sudo sv up tailscaled
```

```bash
sudo sv status tailscaled
```

## ✅ Tailscaleクライアントを接続する

## サービスの実行後にのみ、以下を実行します。

```bash
sudo tailscale up
```

## 🎉 起動時に Tailscale が自動的に起動するようになりました

## 再起動後に確認したい場合：

```bash
sudo tailscale status
```
```bash
sudo sv status tailscaled
```

## ✅ ある時点でサービスを実行し、Tailscale の DNS がキャッシュされている場合は、次のコマンドを実行して独自のネットワーク DNS の使用に戻すことができます。

```bash
sudo tailscale up --accept-dns=false
```

## その後、この新しい構成は状態に保存され、次回の起動では DNS は変更されなくなります。

---

🎯 以上です!

👉連絡先: zerolies@disroot.org
👉 チリ_REF_0_チリ


