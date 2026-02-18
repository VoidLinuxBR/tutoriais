# Complete Tutorial: Intel GPUs on Linux (Void Linux, XFCE and old hardware)

This guide explains how to identify, configure and stabilize Intel GPUs on Linux, especially older hardware like GMA 3100, GMA X3100, HD 2000/3000, etc.

---

# 1. Identify your Intel GPU

Execute:

```bash
lspci -k | grep -A3 VGA
```

or:

```bash
inxi -G
```

Example output:

```
Device: Intel 82G33/G31 Express Integrated Graphics
Driver: i915
Display: X.Org modesetting
OpenGL: 2.1 Mesa i915
```

---

# 2. Drivers Intel no Linux

All Intel GPUs use:

```
kernel driver: i915
```

You DO NOT need to install proprietary driver.

There are two possible Xorg drivers:

- modesetting → recommended
- xf86-video-intel → obsolete in most cases

Check which one is in use:

```bash
inxi -G
```

If it shows:

```
driver: modesetting
```

It's correct.

---

# 3. For older Intel GPUs (GMA 950, 3000, 3100, X3100, X4500)

These GPUs have limitations:

- OpenGL 2.1 maximum
- DRI3 unstable
- composer causes crash
- modern acceleration does not work correctly

---

# 4. Recommended stable configuration

Create:

```bash
sudo mkdir -p /etc/X11/xorg.conf.d
sudo nano /etc/X11/xorg.conf.d/20-intel.conf
```

Content:

```conf
Section "Device"
    Identifier "Intel Graphics"
    Driver "modesetting"
    Option "DRI" "2"
EndSection
```

---

# 5. Disable XFCE composer (REQUIRED on old GPUs)

Execute:

```bash
xfconf-query -c xfwm4 -p /general/use_compositing -s false
```

Check:

```bash
xfconf-query -c xfwm4 -p /general/use_compositing
```

Expected result:

```
false
```

---

# 6. Setting up Firefox for older Intel GPUs

Create:

```bash
nano ~/.mozilla/firefox/*.default-release/user.js
```

Content:

```js
user_pref("layers.acceleration.disabled", true);
user_pref("gfx.webrender.force-disabled", true);
user_pref("gfx.xrender.enabled", true);
user_pref("media.ffmpeg.vaapi.enabled", false);
```

This prevents crashes.

---

# 7. Check OpenGL support

Execute:

```bash
glxinfo | grep "OpenGL version"
```

Example:

```
OpenGL version string: 2.1 Mesa 25.3.3
```

This is the maximum supported by the GMA 3100.

---

# 8. Check active acceleration

Execute:

```bash
glxinfo | grep "direct rendering"
```

Correct result:

```
direct rendering: Yes
```

---

# 9. Check loaded driver

Execute:

```bash
lsmod | grep i915
```

Result:

```
i915
```

---

# 10. Check Xorg status

Execute:

```bash
grep -i driver /var/log/Xorg.0.log
```

Correct result:

```
modesetting
```

---

# 11. Don't use xf86-video-intel on very old hardware

Remove if installed:

```bash
sudo xbps-remove xf86-video-intel
```

The modesetting driver is more stable.

---

# 12. Check DRI

Execute:

```bash
glxinfo | grep DRI
```

For older GPUs use:

```
DRI2
```

---

# 13. Check resolution

Execute:

```bash
xrandr
```

---

# 14. Common Problems and Solution

## reboot when entering XFCE

Disable composer:

```bash
xfconf-query -c xfwm4 -p /general/use_compositing -s false
```

---

## Firefox crashes

Disable acceleration via user.js (step 6)

---

## black screen

Remove xf86-video-intel:

```bash
sudo xbps-remove xf86-video-intel
```

---

# 15. Full graphics system check

Execute:

```bash
inxi -G
glxinfo | grep OpenGL
lsmod | grep i915
xfconf-query -c xfwm4 -p /general/use_compositing
```

---

# 16. Intel GPUs and OpenGL support

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

# 17. Conclusion

For older Intel GPUs, use:

- driver kernel: i915
- driver Xorg: modesetting
- DRI2
- composer disabled
- Firefox acceleration disabled

System will be stable and functional.
