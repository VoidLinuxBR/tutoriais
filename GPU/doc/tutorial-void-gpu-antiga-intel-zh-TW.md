# 完整教學：Linux 上的英特爾 GPU（Void Linux、XFCE 和舊硬體）

本指南介紹如何在 Linux 上識別、配置和穩定 Intel GPU，尤其是 GMA 3100、GMA X3100、HD 2000/3000 等較舊的硬體。

---

# 1. 識別您的 Intel GPU

執行：

```bash
lspci -k | grep -A3 VGA
```

或者：

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

# 2. 驅動程式 Intel 無 Linux



```
kernel driver: i915
```

您不需要安裝專有驅動程式。

有兩種可能的 Xorg 驅動程式：

- 模式設定 → 推薦
- xf86-video-intel → 在大多數情況下已過時

檢查哪一個正在使用：

```bash
inxi -G
```

如果顯示：

```
driver: modesetting
```

這是正確的。

---

# 3. 對於較舊的 Intel GPU（GMA 950、3000、3100、X3100、X4500）

這些 GPU 有限制：

- 
- DRI3不穩定
- 作曲家導致崩潰
- 現代加速無法正常運作

---

# 4. 推薦穩定配置

創造：

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

# 5. 停用 XFCE Composer（舊 GPU 上需要）

執行：

```bash
xfconf-query -c xfwm4 -p /general/use_compositing -s false
```

查看：

```bash
xfconf-query -c xfwm4 -p /general/use_compositing
```

預期結果：

```
false
```

---

# 6.為較舊的 Intel GPU 設定 Firefox

創造：

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

這可以防止崩潰。

---

# 7.檢查OpenGL支持

執行：

```bash
glxinfo | grep "OpenGL version"
```

例子：

```
OpenGL version string: 2.1 Mesa 25.3.3
```

這是 GMA 3100 支援的最大值。

---

# 8. 檢查主動加速

執行：

```bash
glxinfo | grep "direct rendering"
```

正確結果：

```
direct rendering: Yes
```

---

# 9. 檢查已載入的驅動程式

執行：

```bash
lsmod | grep i915
```

結果：

```
i915
```

---

# 10.檢查Xorg狀態

執行：

```bash
grep -i driver /var/log/Xorg.0.log
```

正確結果：

```
modesetting
```

---

# 11. 不要在非常舊的硬體上使用 xf86-video-intel

如果已安裝，請刪除：

```bash
sudo xbps-remove xf86-video-intel
```

模式設定驅動程式更加穩定。

---

# 12.檢查DRI

執行：

```bash
glxinfo | grep DRI
```

對於較舊的 GPU，請使用：

```
DRI2
```

---

# 13. 檢查分辨率

執行：

```bash
xrandr
```

---

# 14. 常見問題及解決方法

## 進入XFCE時重新啟動

禁用作曲家：

```bash
xfconf-query -c xfwm4 -p /general/use_compositing -s false
```

---

## 火狐崩潰

透過 user.js 停用加速（第 6 步）

---

## 黑螢幕

刪除 xf86-video-intel：

```bash
sudo xbps-remove xf86-video-intel
```

---

# 15.完整的圖形系統檢查

執行：

```bash
inxi -G
glxinfo | grep OpenGL
lsmod | grep i915
xfconf-query -c xfwm4 -p /general/use_compositing
```

---

# 16. Intel GPU 和 OpenGL 支持

|圖形處理器 | OpenGL |
|----|--------|
| GMA 950 | 2.1 | 2.1
| GMA 3100 | 2.1 | 2.1
| X3100 | 2.1 | 2.1
| X4500 | 2.1 | 2.1
|高清2000 | 3.3 |
|高清3000 | 3.3 |
|高清4000 | 4.0 |
|超高清 620 | 4.6 | 4.6

---

# 17. 結論

對於較舊的 Intel GPU，請使用：

- 驅動核心：i915
- 驅動程式 Xorg：模式設置
- 直接還原酶2
- 作曲家已禁用
- 火狐加速已停用

系統將穩定且正常運作。
