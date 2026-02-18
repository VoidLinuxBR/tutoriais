# 完全なチュートリアル: Linux 上の Intel GPU (Void Linux、XFCE、および古いハードウェア)

このガイドでは、Linux 上で Intel GPU、特に GMA 3100、GMA X3100、HD 2000/3000 などの古いハードウェアを識別、構成、安定させる方法について説明します。

---

# 1. Intel GPU を特定する

実行する：

```bash
lspci -k | grep -A3 VGA
```

または：

```bash
inxi -G
```

出力例:

```
Device: Intel 82G33/G31 Express Integrated Graphics
Driver: i915
Display: X.Org modesetting
OpenGL: 2.1 Mesa i915
```

---

# 2. ドライバー Intel Linux なし

すべての Intel GPU は以下を使用します。

```
kernel driver: i915
```

独自のドライバーをインストールする必要はありません。

考えられる Xorg ドライバーは 2 つあります。

- モード設定→推奨
- xf86-video-intel → ほとんどの場合廃止

どちらが使用中であるかを確認します。

```bash
inxi -G
```

表示されている場合:

```
driver: modesetting
```

それは正しいです。

---

# 3. 古い Intel GPU (GMA 950、3000、3100、X3100、X4500) の場合



- OpenGL 2.1最大
- DRI3が不安定
- 作曲家がクラッシュを引き起こす
- 最新のアクセラレーションが正しく機能しない

---

# 4. 推奨される安定した構成

作成する：

```bash
sudo mkdir -p /etc/X11/xorg.conf.d
sudo nano /etc/X11/xorg.conf.d/20-intel.conf
```

コンテンツ：

```conf
Section "Device"
    Identifier "Intel Graphics"
    Driver "modesetting"
    Option "DRI" "2"
EndSection
```

---

# 5. XFCEコンポーザーを無効にする(古いGPUでは必須)

実行する：

```bash
xfconf-query -c xfwm4 -p /general/use_compositing -s false
```

チェック：

```bash
xfconf-query -c xfwm4 -p /general/use_compositing
```

期待される結果:

```
false
```

---

# 6. 古い Intel GPU 用の Firefox のセットアップ

作成する：

```bash
nano ~/.mozilla/firefox/*.default-release/user.js
```

コンテンツ：

```js
user_pref("layers.acceleration.disabled", true);
user_pref("gfx.webrender.force-disabled", true);
user_pref("gfx.xrender.enabled", true);
user_pref("media.ffmpeg.vaapi.enabled", false);
```

これによりクラッシュが防止されます。

---

# 7. OpenGL サポートを確認する

実行する：

```bash
glxinfo | grep "OpenGL version"
```

例：

```
OpenGL version string: 2.1 Mesa 25.3.3
```

これは、GMA 3100 でサポートされる最大値です。

---

# 8. アクティブな加速度を確認する

実行する：

```bash
glxinfo | grep "direct rendering"
```

正しい結果:

```
direct rendering: Yes
```

---

# 9. ロードされたドライバーを確認する

実行する：

```bash
lsmod | grep i915
```

結果：

```
i915
```

---

# 10. Xorg ステータスを確認する

実行する：

```bash
grep -i driver /var/log/Xorg.0.log
```

正しい結果:

```
modesetting
```

---

# 11. 非常に古いハードウェアでは xf86-video-intel を使用しないでください。

インストールされている場合は削除します。

```bash
sudo xbps-remove xf86-video-intel
```

モード設定ドライバーの方が安定しています。

---

# 12. DRI をチェックする

実行する：

```bash
glxinfo | grep DRI
```

古い GPU の場合は、以下を使用します。

```
DRI2
```

---

# 13. 解像度の確認

実行する：

```bash
xrandr
```

---

# 14. 一般的な問題と解決策

## XFCE に入るときに再起動する

コンポーザーを無効にする:

```bash
xfconf-query -c xfwm4 -p /general/use_compositing -s false
```

---

## Firefox がクラッシュする

user.js によるアクセラレーションを無効にする (ステップ 6)

---

## 黒い画面

xf86-video-intel を削除します。

```bash
sudo xbps-remove xf86-video-intel
```

---

# 15. 完全なグラフィックス システム チェック

実行する：

```bash
inxi -G
glxinfo | grep OpenGL
lsmod | grep i915
xfconf-query -c xfwm4 -p /general/use_compositing
```

---

# 16. Intel GPU と OpenGL のサポート

| GPU |オープンGL |
|----|--------|
| GMA950 | 2.1 |
| GMA3100 | 2.1 |
| X3100 | 2.1 |
| X4500 | 2.1 |
| HD2000 | 3.3 |
| HD3000 | 3.3 |
| HD4000 | 4.0 |
| UHD620 | 4.6 |

---

# 17. 結論

古い Intel GPU の場合は、次を使用します。

- ドライバーカーネル: i915
- ドライバ Xorg: モード設定
- DRI2
- 作曲家が無効になっています
- Firefox アクセラレーションが無効になっています

システムは安定して機能します。
