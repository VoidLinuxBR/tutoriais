# 完整教程：Linux 上的 Intel GPU（Void Linux、XFCE 和舊硬體）

本指南說明了如何在 Linux 上識別、配置和穩定 Intel GPU，特別是像 GMA 3100、GMA X3100、HD 2000/3000 等舊款硬體。

> **注意：**舊款 Intel GPU 分為兩組，設定會有所不同：
>
> - **A 組 – GMA (950, 3000, 3100, X3100, X4500):** 遵循穩定性步驟（禁用 DRI2、組合器和 Firefox 加速）。不具備有用的硬體影片解碼功能。
> - **B組 – Intel HD（第一代 Core/Ironlake、HD 2000/3000/Sandy Bridge、HD 2500/4000/Ivy Bridge、Haswell）：** 一般不需要穩定性調整，**可以且應該使用視頻加速（VA-API）**，搭配 `i965` 驅動程式（參見第7節）。
>
> 使用第1部分來查找您的GPU屬於哪個組別。

---

# 1. 識別您的 Intel 顯示卡

執行：

```bash
lspci -k | grep -A3 VGA
```

ou:

```bash
inxi -G
```

輸出範例：

```
Device: Intel 82G33/G31 Express Integrated Graphics
Driver: i915
Display: X.Org modesetting
OpenGL: 2.1 Mesa i915
```

---

# 2. Intel 驅動程式在 Linux 上

所有 Intel GPU 使用：

```
kernel driver: i915
```

您不需要安裝專有驅動程式。在 Intel，**所有驅動程式都是自由的**。

## 2.1 i915、i965 和 iHD：不要混淆

它們是不同的層次，並且常常認為它們是彼此的替代方案：

| 名稱 | 是什麼 | 來自哪裡 |
|------|---------|----------|
| `i915` | **內核** 驅動程式（視頻/模式設定） | 已包含在內核中，無需安裝任何東西 |
| `i965` | 驅動程式 **VA-API**（視頻解碼）適用於 GPU，直到 ~Coffee Lake | 套件 `libva-intel-driver` |
| `iHD` | Broadwell（第5代）及更新款的 **VA-API** 驅動程式 | `intel-media-driver` 套件 |
| `crocus` / `iris` | Mesa **OpenGL** 驅動程式 (crocus: Gen4 至 Haswell; iris: Broadwell+) | `mesa-dri` 套件 |

也就是說：擁有 HD 2000/3000 的人同時使用 `i915`（內核）**和** `i965`（VA-API）。

## 2.2 Xorg 驅動程式

有兩個可能的 Xorg 驅動程式：

- 模式設定 → 推薦
- xf86-video-intel → 在大多數情況下已過時

確認哪一個正在使用：

```bash
inxi -G
```

如果顯示：

```
driver: modesetting
```

正確。

---

# 3. 對於舊款 Intel GPU（GMA 950、3000、3100、X3100、X4500）

這些 GPU 有限制：

- OpenGL 2.1 最大
- DRI3 不穩定
- 組件導致當機
- 現代加速功能無法正常運作

---

# 4. 建議的穩定配置

創建：

```bash
sudo mkdir -p /etc/X11/xorg.conf.d
sudo nano /etc/X11/xorg.conf.d/20-intel.conf
```

內容：

```conf
Section "Device"
    Identifier "Intel Graphics"
    Driver "modesetting"
    Option "DRI" "2"
EndSection
```

---

# 5. 停用 XFCE 組合器（在舊型 GPU 上為必要）

執行：

```bash
xfconf-query -c xfwm4 -p /general/use_compositing -s false
```

請檢查：

```bash
xfconf-query -c xfwm4 -p /general/use_compositing
```

預期結果：

```
false
```

---

# 6. 為舊款 Intel GPU 配置 Firefox（僅限 A 組 – GMA）

> **僅適用於 GMA 950/3000/3100/X3100/X4500。** 如果您的 GPU 是 Intel HD（HD 2000/3000 或更新型號），**請勿執行此步驟**：它會關閉影片加速。請直接跳到第 7 節。

創建：

```bash
nano ~/.mozilla/firefox/*.default-release/user.js
```

內容：

```js
user_pref("layers.acceleration.disabled", true);
user_pref("gfx.webrender.force-disabled", true);
user_pref("gfx.xrender.enabled", true);
user_pref("media.ffmpeg.vaapi.enabled", false);
```

這可以避免當機。

---

# 7. 視頻加速 (VA-API) 適用於 Intel HD（B組）

使用 VA-API，GPU 會解碼影片，而 CPU 可以保持空閒。這在使用第二或第三代 Core i3/i5 的筆記本電腦觀看 YouTube 時差異很大。

## 7.1 每個 GPU 在硬體中解碼的內容

| GPU | 支援加速的編碼器 | VA-API 驅動程式 |
|-----|----------------------|---------------|
| GMA 950/3000/3100/X3100 | 無 | — |
| GMA X4500 | MPEG-2 | `i965` |
| HD 圖形 (Ironlake，第1代 Core) | MPEG-2, H.264, VC-1 | `i965` |
| HD 2000/3000（Sandy Bridge）| MPEG-2, H.264, VC-1 | `i965` |
| HD 2500/4000（Ivy Bridge）| MPEG-2、H.264、VC-1、JPEG | `i965` |
| HD 4200–5200（Haswell）| MPEG-2、H.264、VC-1、JPEG | `i965` |
| Broadwell 及更新 | 根據世代支援 VP8/HEVC/VP9/AV1 | `iHD` |

**直到 Haswell 為止，沒有任何 GPU 支援 VP9 或 AV1 解碼。** YouTube 預設使用 VP9/AV1，因此在這些情況下，請在瀏覽器安裝一個強制使用 H.264 的擴充功能（例如：*h264ify* 或 *enhanced-h264ify*）。沒有安裝的話，影片將會回到由 CPU 解碼。

## 7.2 安裝套件

```bash
sudo xbps-install -S mesa-dri libva-intel-driver libva-utils intel-gpu-tools
```

確保使用者在 `video` 群組中（之後請退出並重新進入會話）：

```bash
sudo usermod -aG video $USER
```

## 7.3 強制使用正確的 VA-API 驅動程式

請將其添加到 `~/.profile`（或在 `/etc/environment` 中，無需 `export`）：

```bash
export LIBVA_DRIVER_NAME=i965
```

## 7.4 測試

```bash
vainfo
```

預期結果（簡述）:

```
vainfo: Driver version: Intel i965 driver for Intel(R) Sandybridge Mobile
      VAProfileH264Main               : VAEntrypointVLD
      VAProfileH264High               : VAEntrypointVLD
```

`VAEntrypointVLD` = 可用的硬體解碼。

## 7.5 火狐瀏覽器

在 `about:config`:

```
media.ffmpeg.vaapi.enabled = true
```

在更新版本的 Firefox 中，如果解碼沒有自動啟動，也可以：

```
media.hardware-video-decoding.force-enabled = true
```

如果您應用了第6節的`user.js`，請將其刪除。重新啟動Firefox，然後在`about:support`的**媒體**部分檢查H264是否顯示硬體解碼。

## 7.6 確認 GPU 正在運作

打開 YouTube 視頻（啟用 H.264 擴展）並運行：

```bash
sudo intel_gpu_top
```

如果 **Video** 線路在影片播放時被佔用，則解碼在 GPU 上進行。CPU 使用率會大幅下降。

---

# 8. 檢查 OpenGL 支援

執行：

```bash
glxinfo | grep "OpenGL version"
```

範例：

```
OpenGL version string: 2.1 Mesa 25.3.3
```

這是 GMA 3100 支援的最大值。

---

# 9. 檢查主動加速度

執行：

```bash
glxinfo | grep "direct rendering"
```

正確結果：

```
direct rendering: Yes
```

---

# 10. 檢查已載入的驅動程式

執行：

```bash
lsmod | grep i915
```

結果：

```
i915
```

---

# 11. 檢查 Xorg 狀態

執行：

```bash
grep -i driver /var/log/Xorg.0.log
```

正確結果：

```
modesetting
```

---

# 12. 不要在非常舊的硬體上使用 xf86-video-intel

如果已安裝，請移除：

```bash
sudo xbps-remove xf86-video-intel
```

modesetting 驅動程式更穩定。

---

# 13. 檢查 DRI

執行：

```bash
glxinfo | grep DRI
```

對於舊款 GPU，請使用：

```
DRI2
```

---

# 14. 檢查解析度

執行：

```bash
xrandr
```

---

# 15. 常見問題及解決方法

## 進入 XFCE 時重新啟動

停用合成器：

```bash
xfconf-query -c xfwm4 -p /general/use_compositing -s false
```

---

## Firefox 當機

透過 user.js 停用加速（步驟 6）。**僅限 GMA** — 在 Intel HD 上，建議保留加速。

---

## 使用大量 CPU（Intel HD）的 YouTube 影片

- 確認強制 H.264 的擴展是否已啟用（GPU 不解碼 VP9/AV1）。
- 運行 `vainfo` 並查看是否出現 `VAProfileH264...: VAEntrypointVLD`。
- 確認 `LIBVA_DRIVER_NAME=i965` 與 `echo $LIBVA_DRIVER_NAME`。
- 確認第6節的 `user.js` 未被啟用（它會關閉 VA-API）。

---

## 黑屏

移除 xf86-video-intel：

```bash
sudo xbps-remove xf86-video-intel
```

---

# 16. 完整的圖形系統檢查

執行：

```bash
inxi -G
glxinfo | grep OpenGL
lsmod | grep i915
xfconf-query -c xfwm4 -p /general/use_compositing
vainfo          # somente Intel HD (Grupo B)
```

---

# 17. Intel GPU 及 OpenGL 支援

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

**A組 – GMA (950、3000、3100、X3100、X4500):**

- 驅動程式核心：i915
- 驅動程式 Xorg：模式設置
- DRI2
- 作曲家已停用
- Firefox 加速已停用

**B組 – Intel HD（Ironlake、HD 2000/3000、HD 2500/4000、Haswell）：**

- 驅動程式核心：i915
- 驅動程式 Xorg：模式設置
- 驅動程式 VA-API: i965 (`libva-intel-driver`) + `LIBVA_DRIVER_NAME=i965`
- Firefox 中已啟用 VA-API
- 在 YouTube 強制使用 H.264 的擴充功能

系統將保持穩定且可運作。
