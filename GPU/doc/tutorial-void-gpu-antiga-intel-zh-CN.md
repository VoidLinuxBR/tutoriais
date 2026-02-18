# 完整教程：Linux 上的英特尔 GPU（Void Linux、XFCE 和旧硬件）

本指南介绍了如何在 Linux 上识别、配置和稳定 Intel GPU，尤其是 GMA 3100、GMA X3100、HD 2000/3000 等较旧的硬件。

---

# 1. 识别您的 Intel GPU

执行：

```bash
lspci -k | grep -A3 VGA
```

或者：

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

# 2. 驱动程序 Intel 无 Linux

所有英特尔 GPU 均使用：

```
kernel driver: i915
```

您不需要安装专有驱动程序。

有两种可能的 Xorg 驱动程序：

- 模式设置 → 推荐
- xf86-video-intel → 在大多数情况下已过时

检查哪一个正在使用：

```bash
inxi -G
```

如果显示：

```
driver: modesetting
```

这是正确的。

---

# 3. 对于较旧的 Intel GPU（GMA 950、3000、3100、X3100、X4500）

这些 GPU 有局限性：

- OpenGL 2.1 最高
- DRI3不稳定
- 作曲家导致崩溃
- 现代加速无法正常工作

---

# 4. 推荐稳定配置

创造：

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

# 5. 禁用 XFCE Composer（旧 GPU 上需要）

执行：

```bash
xfconf-query -c xfwm4 -p /general/use_compositing -s false
```

查看：

```bash
xfconf-query -c xfwm4 -p /general/use_compositing
```

预期结果：

```
false
```

---

# 6.为较旧的 Intel GPU 设置 Firefox

创造：

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

这可以防止崩溃。

---

# 7.检查OpenGL支持

执行：

```bash
glxinfo | grep "OpenGL version"
```

例子：

```
OpenGL version string: 2.1 Mesa 25.3.3
```

这是 GMA 3100 支持的最大值。

---

# 8. 检查主动加速

执行：

```bash
glxinfo | grep "direct rendering"
```

正确结果：

```
direct rendering: Yes
```

---

# 9. 检查加载的驱动程序

执行：

```bash
lsmod | grep i915
```

结果：

```
i915
```

---

# 10.检查Xorg状态

执行：

```bash
grep -i driver /var/log/Xorg.0.log
```

正确结果：

```
modesetting
```

---

# 11. 不要在非常旧的硬件上使用 xf86-video-intel

如果已安装，请删除：

```bash
sudo xbps-remove xf86-video-intel
```

模式设置驱动程序更加稳定。

---

# 12.检查DRI

执行：

```bash
glxinfo | grep DRI
```

对于较旧的 GPU，请使用：

```
DRI2
```

---

# 13. 检查分辨率

执行：

```bash
xrandr
```

---

# 14. 常见问题及解决方法

## 进入XFCE时重新启动

禁用作曲家：

```bash
xfconf-query -c xfwm4 -p /general/use_compositing -s false
```

---

## 火狐崩溃

通过 user.js 禁用加速（第 6 步）

---

## 黑屏

删除 xf86-video-intel：

```bash
sudo xbps-remove xf86-video-intel
```

---

# 15.完整的图形系统检查

执行：

```bash
inxi -G
glxinfo | grep OpenGL
lsmod | grep i915
xfconf-query -c xfwm4 -p /general/use_compositing
```

---

# 16. Intel GPU 和 OpenGL 支持

|图形处理器 | OpenGL |
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

# 17. 结论

对于较旧的 Intel GPU，请使用：

- 驱动内核：i915
- 驱动程序 Xorg：模式设置
- 直接还原酶2
- 作曲家已禁用
- 火狐加速已禁用

系统将稳定且正常运行。
