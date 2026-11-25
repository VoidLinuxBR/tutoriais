# 完整教程：Linux 上的 Intel GPU（Void Linux、XFCE 和旧硬件）

本指南说明了如何在 Linux 上识别、配置和稳定 Intel GPU，特别是像 GMA 3100、GMA X3100、HD 2000/3000 等旧硬件。

> **注意：** 旧款英特尔 GPU 分为两组，配置有所不同：
>
> - **A组 – GMA (950, 3000, 3100, X3100, X4500)：** 按照稳定性步骤（DRI2、合成器和Firefox加速已禁用）。没有有用的硬件视频解码。
> - **B组 – Intel HD（第1代Core/Ironlake, HD 2000/3000/Sandy Bridge, HD 2500/4000/Ivy Bridge, Haswell）：** 通常不需要进行稳定性调整，**可以且应当使用视频加速（VA-API）**，使用 `i965` 驱动（参见第7节）。
>
> 使用第1部分来确定你的GPU属于哪个组。

---

# 1. 识别您的英特尔 GPU

执行：

```bash
lspci -k | grep -A3 VGA
```

或：

```bash
inxi -G
```

输出示例：

```
Device: Intel 82G33/G31 Express Integrated Graphics
Driver: i915
Display: X.Org modesetting
OpenGL: 2.1 Mesa i915
```

---

# 2. Linux上的Intel驱动程序

所有英特尔 GPU 都使用：

```
kernel driver: i915
```

您不需要安装专有驱动。在英特尔，**所有驱动都是开源的**。

## 2.1 i915、i965 和 iHD：不要混淆

它们是不同的层次，人们常常认为它们是相互替代的：

| 名称 | 是什么 | 来自哪里 |
|------|---------|----------|
| `i915` | **内核** 驱动程序（视频/模式设置） | 已包含在内核中，无需安装任何东西 |
| `i965` | 驱动程序 **VA-API**（视频解码）适用于最多 ~Coffee Lake 的 GPU | `libva-intel-driver` 包 |
| `iHD` | Broadwell（第5代）及更新型号的 **VA-API** 驱动程序 | `intel-media-driver` 软件包 |
| `crocus` / `iris` | Mesa **OpenGL** 驱动程序 (crocus: Gen4 到 Haswell; iris: Broadwell+) | `mesa-dri` 包 |

也就是说：拥有 HD 2000/3000 的人同时使用 `i915`（内核）**和** `i965`（VA-API）。

## 2.2 驱动程序 Xorg

有两种可能的 Xorg 驱动程序：

- modesetting → 推荐
- xf86-video-intel → 在大多数情况下已过时

检查哪个正在使用：

```bash
inxi -G
```

如果显示：

```
driver: modesetting
```

正确。

---

# 3. 对于旧款英特尔GPU（GMA 950、3000、3100、X3100、X4500）

这些 GPU 有限制：

- OpenGL 2.1 最大
- DRI3 不稳定
- 作曲器导致卡顿
- 现代加速不起作用

---

# 4. 建议的稳定配置

创建：

```bash
sudo mkdir -p /etc/X11/xorg.conf.d
sudo nano /etc/X11/xorg.conf.d/20-intel.conf
```

内容：

```conf
Section "Device"
    Identifier "Intel Graphics"
    Driver "modesetting"
    Option "DRI" "2"
EndSection
```

---

# 5. 禁用 XFCE 合成器（在老旧 GPU 上必需）

执行：

```bash
xfconf-query -c xfwm4 -p /general/use_compositing -s false
```

检查：

```bash
xfconf-query -c xfwm4 -p /general/use_compositing
```

预期结果：

```
false
```

---

# 6. 为旧款英特尔 GPU 配置 Firefox（仅限 A 组 – GMA）

> **仅适用于 GMA 950/3000/3100/X3100/X4500。** 如果您的 GPU 是 Intel HD（HD 2000/3000 或更新型号），**请不要执行此步骤**：此步骤会关闭视频加速。请直接跳到第 7 部分。

创建：

```bash
nano ~/.mozilla/firefox/*.default-release/user.js
```

内容：

```js
user_pref("layers.acceleration.disabled", true);
user_pref("gfx.webrender.force-disabled", true);
user_pref("gfx.xrender.enabled", true);
user_pref("media.ffmpeg.vaapi.enabled", false);
```

这可以避免卡顿。

---

# 7. 视频加速 (VA-API) 用于 Intel HD（B组）

使用 VA-API，GPU 解码视频，CPU 可以空闲。这在使用第 2/3 代 Core i3/i5 的笔记本电脑观看 YouTube 时差别很大。

## 7.1 每块GPU在硬件中解码的内容

| GPU | 带加速的编解码器 | VA-API 驱动 |
|-----|----------------------|---------------|
| GMA 950/3000/3100/X3100 | 无 | — |
| GMA X4500 | MPEG-2 | `i965` |
| 高清显卡 (Ironlake，第一代 Core) | MPEG-2，H.264，VC-1 | `i965` |
| HD 2000/3000（Sandy Bridge）| MPEG-2, H.264, VC-1 | `i965` |
| HD 2500/4000（常春藤桥）| MPEG-2、H.264、VC-1、JPEG | `i965` |
| HD 4200–5200（Haswell）| MPEG-2, H.264, VC-1, JPEG | `i965` |
| Broadwell 及更新型号 | 根据代际支持 VP8/HEVC/VP9/AV1 | `iHD` |

**直到 Haswell，没有任何 GPU 可以解码 VP9 或 AV1。**YouTube 默认使用 VP9/AV1，因此在这些情况下，请在浏览器中安装一个强制使用 H.264 的扩展（例如：*h264ify* 或 *enhanced-h264ify*）。没有它，视频将再次由 CPU 解码。

## 7.2 安装软件包

```bash
sudo xbps-install -S mesa-dri libva-intel-driver libva-utils intel-gpu-tools
```

确保用户在 `video` 组中（退出然后重新进入会话）：

```bash
sudo usermod -aG video $USER
```

## 7.3 强制使用正确的 VA-API 驱动

添加到 `~/.profile`（或在 `/etc/environment` 中，不包括 `export`）：

```bash
export LIBVA_DRIVER_NAME=i965
```

## 7.4 测试

```bash
vainfo
```

预期结果（简要）：

```
vainfo: Driver version: Intel i965 driver for Intel(R) Sandybridge Mobile
      VAProfileH264Main               : VAEntrypointVLD
      VAProfileH264High               : VAEntrypointVLD
```

`VAEntrypointVLD` = 硬件解码可用。

## 7.5 火狐

在 `about:config`：

```
media.ffmpeg.vaapi.enabled = true
```

在更新的 Firefox 版本中，如果解码没有自动开启，也可以：

```
media.hardware-video-decoding.force-enabled = true
```

如果你在第6节应用了`user.js`，请将其删除。重新启动Firefox，并在`about:support`，第**媒体**节，检查H264是否显示硬件解码。

## 7.6 确认 GPU 正在工作

在 YouTube 上打开一个视频（启用 H.264 扩展）并运行：

```bash
sudo intel_gpu_top
```

如果在视频播放时 **视频** 线路显示忙，说明解码是在 GPU 上进行的。CPU 使用率会大幅下降。

---

# 8. 检查 OpenGL 支持

执行：

```bash
glxinfo | grep "OpenGL version"
```

例子：

```
OpenGL version string: 2.1 Mesa 25.3.3
```

这是GMA 3100所支持的最大值。

---

# 9. 检查主动加速

执行：

```bash
glxinfo | grep "direct rendering"
```

正确结果：

```
direct rendering: Yes
```

---

# 10. 检查已加载的驱动程序

执行：

```bash
lsmod | grep i915
```

结果：

```
i915
```

---

# 11. 检查 Xorg 状态

执行：

```bash
grep -i driver /var/log/Xorg.0.log
```

正确结果：

```
modesetting
```

---

# 12. 在非常老的硬件上不要使用 xf86-video-intel

如果已安装，请移除：

```bash
sudo xbps-remove xf86-video-intel
```

modesetting 驱动程序更稳定。

---

# 13. 检查 DRI

执行：

```bash
glxinfo | grep DRI
```

对于旧显卡，请使用：

```
DRI2
```

---

# 14. 检查分辨率

执行：

```bash
xrandr
```

---

# 15. 常见问题及解决方法

## 进入XFCE时重启

禁用合成器：

```bash
xfconf-query -c xfwm4 -p /general/use_compositing -s false
```

---

## Firefox 崩溃

通过 user.js 禁用加速（步骤 6）。**仅限 GMA** — 对于 Intel HD，建议保持加速。

---

## 使用大量 CPU（Intel HD）的 YouTube 视频

- 检查强制 H.264 的扩展是否已启用（GPU 不解码 VP9/AV1）。
- 运行 `vainfo`，看看是否出现 `VAProfileH264...: VAEntrypointVLD`。
- 请将 `LIBVA_DRIVER_NAME=i965` 与 `echo $LIBVA_DRIVER_NAME` 确认。
- 确认第6节的 `user.js` 未被应用（它会关闭 VA-API）。

---

## 黑屏

移除 xf86-video-intel:

```bash
sudo xbps-remove xf86-video-intel
```

---

# 16. 图形系统的完整检查

执行：

```bash
inxi -G
glxinfo | grep OpenGL
lsmod | grep i915
xfconf-query -c xfwm4 -p /general/use_compositing
vainfo          # somente Intel HD (Grupo B)
```

---

# 17. 英特尔 GPU 及 OpenGL 支持

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

# 18. 结论

**A组 – GMA (950, 3000, 3100, X3100, X4500):**

- 驱动内核：i915
- 驱动程序 Xorg：模式设置
- DRI2
- 作曲家已禁用
- Firefox 加速已禁用

**B组 – Intel HD（Ironlake, HD 2000/3000, HD 2500/4000, Haswell）：**

- 驱动内核：i915
- 驱动程序 Xorg：模式设置
- 驱动程序 VA-API: i965 (`libva-intel-driver`) + `LIBVA_DRIVER_NAME=i965`
- Firefox 中启用了 VA-API
- 在 YouTube 上强制使用 H.264 的扩展

系统将保持稳定且可运行。
