# 完全チュートリアル: LinuxでのIntel GPU（Void Linux、XFCE、古いハードウェア）

このガイドでは、特に GMA 3100、GMA X3100、HD 2000/3000 などの古いハードウェアを含む、Linux で Intel GPU を識別、設定、および安定化する方法について説明します。

> **注意:** 古いIntelのGPUは2つのグループに分かれ、設定が変わります:
>
> - **グループA – GMA (950, 3000, 3100, X3100, X4500):** 安定性の手順に従います（DRI2、コンポジタ、およびFirefoxのハードウェアアクセラレーションは無効）。ハードウェアによる有用なビデオデコーディングはありません。
> - **グループB – Intel HD（第1世代Core/Ironlake、HD 2000/3000/Sandy Bridge、HD 2500/4000/Ivy Bridge、Haswell）：** 一般的に安定性の調整は必要なく、**`i965`ドライバーを使用してビデオアクセラレーション（VA-API）を利用することができ、かつ推奨されます**（セクション7参照）。
>
> セクション1を使って、自分のGPUがどのグループに属しているかを確認してください。

---

# クイック方式: `voidbr-gpu-config`

VoidBRでは、ユーティリティ `voidbr-gpu-config`（パッケージ `voidbr-utils`）が、このチュートリアルで説明されていることを自動的に行います。これは **Intel および AMD** のGPUで動作します。

- GPUと世代（GMA、Intel HD、AMD）を検出する；
- 正しいパッケージ（`libva-intel-driver`、`intel-media-driver`、または `mesa-vaapi`）をインストールします；
- `vainfo`でどのVA-APIドライバーが実際に動作するかテストして、`LIBVA_DRIVER_NAME`を設定します。
- ユーザーを `video` および `render` グループに追加します。
- Firefox、Google Chrome、Brave を設定する（VA-API は Intel HD/AMD で有効、または GMA で安定性プロファイル）。
- すべてをバックアップし、元に戻すことができるようにします。

まず、診断だけ（何も変わらない）：

```bash
voidbr-gpu-config --check
```

推奨設定を適用する:

```bash
sudo voidbr-gpu-config --apply
```

元に戻す:

```bash
sudo voidbr-gpu-config --revert
```

アプリメニューにも **VoidBR GPU Config** として表示されます。適用した後、セッションから一度ログアウトして再度ログインしてください。

もし何かがうまくいかない場合は、`voidbr-gpu-config --check` を実行し、出力を Telegram のグループに投稿してください: **https://t.me/voidlinux_br**

もしすべて手動で行いたい場合は、以下のセクションに従ってください。

---

# 1. 自分のIntel GPUを識別する

実行:

```bash
lspci -k | grep -A3 VGA
```

または:

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

# 2. Linux の Intel ドライバー

すべてのIntel GPUは以下を使用します：

```
kernel driver: i915
```

あなたは専用ドライバーをインストールする必要はありません。Intelでは、**すべてのドライバーがオープンです**。

## 2.1 i915、i965、iHD：混同しないでください

それぞれ異なる層であり、互いに代替可能だと考えることはよくあります。

| 名前 | 何ですか | どこから来ますか |
|------|---------|----------|
| `i915` | **カーネル**ドライバー（ビデオ/モード設定） | カーネルにすでに含まれており、何もインストールしません |
| `i965` | GPU向け**VA-API**ドライバー（ビデオデコード、〜Coffee Lakeまで） | パッケージ `libva-intel-driver` |
| `iHD` | Broadwell（第5世代）以降用のドライバー **VA-API** | パッケージ `intel-media-driver` |
| `crocus` / `iris` | Mesa の **OpenGL** ドライバー (crocus: Gen4 a Haswell; iris: Broadwell+) | `mesa-dri` パッケージ |

つまり、HD 2000/3000を持っている人は、`i915`（カーネル）**と**`i965`（VA-API）を同時に使用します。

## 2.2 ドライバー Xorg

使用可能なXorgドライバーは2つあります:

- モード設定 → 推奨
- xf86-video-intel → ほとんどの場合、廃止予定

どれが使用中か確認してください:

```bash
inxi -G
```

表示する場合:

```
driver: modesetting
```

正しいです。

---

# 3. 古いIntel GPU向け（GMA 950、3000、3100、X3100、X4500）

これらのGPUには制限があります:

- OpenGL 2.1 最大
- 不安定なDRI3
- コンポジターがクラッシュを引き起こす
- 現代の加速は正しく機能しません

---

# 4. 推奨される安定した設定

作成:

```bash
sudo mkdir -p /etc/X11/xorg.conf.d
sudo nano /etc/X11/xorg.conf.d/20-intel.conf
```

内容:

```conf
Section "Device"
    Identifier "Intel Graphics"
    Driver "modesetting"
    Option "DRI" "2"
EndSection
```

---

# 5. XFCE コンポジタを無効化する（古い GPU では必須）

実行:

```bash
xfconf-query -c xfwm4 -p /general/use_compositing -s false
```

確認してください:

```bash
xfconf-query -c xfwm4 -p /general/use_compositing
```

期待される結果:

```
false
```

---

# 6. 古いIntel GPU向けのFirefoxの設定（グループAのみ – GMA）

> **GMA 950/3000/3100/X3100/X4500専用。** お使いのGPUがIntel HD（HD 2000/3000以降）の場合、**この手順は実行しないでください**：これはビデオアクセラレーションを無効にします。直接セクション7に進んでください。

作成:

```bash
nano ~/.mozilla/firefox/*.default-release/user.js
```

内容:

```js
user_pref("layers.acceleration.disabled", true);
user_pref("gfx.webrender.force-disabled", true);
user_pref("gfx.xrender.enabled", true);
user_pref("media.ffmpeg.vaapi.enabled", false);
```

これによりフリーズを防ぎます。

---

# 7. ビデオアクセラレーション (VA-API) インテル HD 用 (グループ B)

VA-APIを使うと、GPUがビデオをデコードし、CPUは空きます。これにより、第2世代/第3世代のCore i3/i5搭載ノートパソコンでYouTubeを見る際に大きな違いが生まれます。

## 7.1 各GPUがハードウェアでデコードするもの

| GPU | ハードウェアアクセラレーション対応コーデック | VA-API ドライバー |
|-----|----------------------|---------------|
| GMA 950/3000/3100/X3100 | なし | — |
| GMA X4500 | MPEG-2 | `i965` |
| HDグラフィックス（アイアンレイク、第1世代Core）| MPEG-2、H.264、VC-1 | `i965` |
| HD 2000/3000（サンディブリッジ） | MPEG-2、H.264、VC-1 | `i965` |
| HD 2500/4000（アイビーブリッジ） | MPEG-2、H.264、VC-1、JPEG | `i965` |
| HD 4200–5200（ハスウェル）| MPEG-2、H.264、VC-1、JPEG | `i965` |
| Broadwell以降 | 世代に応じて+ VP8/HEVC/VP9/AV1 | `iHD` |

**Haswell以前のGPUではVP9やAV1をデコードできません。** YouTubeはデフォルトでVP9/AV1を使用するため、その場合はブラウザにH.264を強制する拡張機能（例：*h264ify* または *enhanced-h264ify*）をインストールしてください。それがないと、動画は再びCPUでデコードされます。

## 7.2 パッケージをインストールする

```bash
sudo xbps-install -S mesa-dri libva-intel-driver libva-utils intel-gpu-tools
```

ユーザーが`video`グループにいることを確認してください（その後、セッションから一度ログアウトして再度ログインしてください）：

```bash
sudo usermod -aG video $USER
```

## 7.3 正しい VA-API ドライバーを強制する

`~/.profile`（または `/etc/environment`、`export` を除く）に追加してください:

```bash
export LIBVA_DRIVER_NAME=i965
```

## 7.4 テストする

```bash
vainfo
```

期待される結果（要約）:

```
vainfo: Driver version: Intel i965 driver for Intel(R) Sandybridge Mobile
      VAProfileH264Main               : VAEntrypointVLD
      VAProfileH264High               : VAEntrypointVLD
```

`VAEntrypointVLD` = ハードウェアによるデコードが利用可能です。

## 7.5 ファイアフォックス

`about:config`で:

```
media.ffmpeg.vaapi.enabled = true
```

Firefox の最新バージョンでは、デコードが自動で開始しない場合も、次のようにします:

```
media.hardware-video-decoding.force-enabled = true
```

もしセクション6の`user.js`を適用した場合は、それを削除してください。Firefoxを再起動し、`about:support`の**メディア**セクションでH264がハードウェアデコードで表示されるか確認してください。

## 7.6 GPUが動作していることを確認する

H.264拡張機能を有効にした状態でYouTubeの動画を開き、次の操作を行います:

```bash
sudo intel_gpu_top
```

動画再生中に**Video**ラインが使用中の場合、デコードはGPUで行われています。CPUの使用量はかなり下がります。

---

# 8. OpenGLサポートの確認

実行:

```bash
glxinfo | grep "OpenGL version"
```

例:

```
OpenGL version string: 2.1 Mesa 25.3.3
```

これは GMA 3100 がサポートする最大です。

---

# 9. 能動的加速の確認

実行:

```bash
glxinfo | grep "direct rendering"
```

正しい結果：

```
direct rendering: Yes
```

---

# 10. ロードされたドライバーを確認する

実行:

```bash
lsmod | grep i915
```

結果：

```
i915
```

---

# 11. Xorgのステータスを確認する

実行:

```bash
grep -i driver /var/log/Xorg.0.log
```

正しい結果：

```
modesetting
```

---

# 12. 古いハードウェアでは xf86-video-intel を使用しないでください

インストールされている場合は削除してください：

```bash
sudo xbps-remove xf86-video-intel
```

モードセッティングドライバーの方がより安定しています。

---

# 13. DRIを確認する

実行:

```bash
glxinfo | grep DRI
```

古いGPUの場合は、次を使用してください:

```
DRI2
```

---

# 14. 解像度を確認する

実行:

```bash
xrandr
```

---

# 15. よくある問題と解決策

## XFCEに入ると再起動

コンポジタを無効にする:

```bash
xfconf-query -c xfwm4 -p /general/use_compositing -s false
```

---

## Firefoxがフリーズする

user.jsでのアクセラレーションを無効にする（ステップ6）。**GMAのみ** — Intel HDの場合は、アクセラレーションを維持することを推奨。

---

## 多くのCPUを使用するYouTube動画（Intel HD）

- H.264を強制する拡張機能が有効になっているか確認してください（GPUはVP9/AV1をデコードしません）。
- `vainfo` を実行して、`VAProfileH264...: VAEntrypointVLD` が表示されるか確認してください。
- `LIBVA_DRIVER_NAME=i965` を `echo $LIBVA_DRIVER_NAME` と確認してください。
- セクション6の`user.js`が適用されていないことを確認してください（これによりVA-APIがオフになります）。

---

## 黒い画面

xf86-video-intel を削除してください:

```bash
sudo xbps-remove xf86-video-intel
```

---

# 16. グラフィックシステムの完全な確認

実行:

```bash
inxi -G
glxinfo | grep OpenGL
lsmod | grep i915
xfconf-query -c xfwm4 -p /general/use_compositing
vainfo          # somente Intel HD (Grupo B)
```

---

# 17. インテルGPUとOpenGLサポート

| GPU | OpenGL |
|----|--------|
| GMA 950 | 2.1 |
| GMA 3100 | 2.1 |
| X3100 | 2.1 |
| X4500 | 2.1 |
| HD 2000 | 3.3 |
| HD 3000 | 3.3 |
| HD 4000 | 4.0 |
| UHD 620 | 4.6 |

---

# 18. 結論

**グループA – GMA (950、3000、3100、X3100、X4500):**

- ドライバー カーネル: i915
- ドライバー Xorg: モードセッティング
- DRI2
- 作曲家無効
- Firefox のハードウェアアクセラレーションが無効になっている

**グループB – Intel HD（Ironlake、HD 2000/3000、HD 2500/4000、Haswell）：**

- ドライバー カーネル: i915
- ドライバー Xorg: モードセッティング
- ドライバー VA-API: i965 (`libva-intel-driver`) + `LIBVA_DRIVER_NAME=i965`
- FirefoxでVA-APIが有効になっています
- YouTubeでH.264を強制する拡張機能

システムは安定して機能します。
