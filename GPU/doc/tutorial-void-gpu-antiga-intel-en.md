# Complete tutorial: Intel GPUs on Linux (Void Linux, XFCE, and old hardware)

This guide explains how to identify, configure, and stabilize Intel GPUs on Linux, especially old hardware like GMA 3100, GMA X3100, HD 2000/3000, etc.

> **Attention:** old Intel GPUs are divided into two groups, and the configuration changes:
>
> - **Group A – GMA (950, 3000, 3100, X3100, X4500):** follow the stability steps (DRI2, compositor, and Firefox acceleration disabled). They do not have useful hardware video decoding.
> - **Group B – Intel HD (1st generation Core/Ironlake, HD 2000/3000/Sandy Bridge, HD 2500/4000/Ivy Bridge, Haswell):** generally do not need stability adjustments and **can and should use video acceleration (VA-API)** with the `i965` driver (see section 7).
>
> Use section 1 to find out which group your GPU is in.

---

# Quick way: `voidbr-gpu-config`

In VoidBR, the utility `voidbr-gpu-config` (`voidbr-utils` package) automatically does what this tutorial explains. It works with **Intel and AMD** GPUs:

- detects the GPU and the generation (GMA, Intel HD, AMD);
- install the right packages (`libva-intel-driver`, `intel-media-driver` or `mesa-vaapi`);
- test with `vainfo` which VA-API driver really works and sets `LIBVA_DRIVER_NAME`;
- add the user to the `video` and `render` groups;
- configures **Firefox**, **Google Chrome**, and **Brave** (VA-API enabled on Intel HD/AMD, or stability profile on GMA);
- backs up everything and allows undoing.

First, only the diagnosis (it doesn't change anything):

```bash
voidbr-gpu-config --check
```

Apply the recommended setting:

```bash
sudo voidbr-gpu-config --apply
```

Undo:

```bash
sudo voidbr-gpu-config --revert
```

It is also in the applications menu as **VoidBR GPU Config**. After applying, log out and log back into the session.

If something doesn't work, run `voidbr-gpu-config --check` and post the output in the Telegram group: **https://t.me/voidlinux_br**

If you prefer to do everything manually, follow the sections below.

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

Example of output:

```
Device: Intel 82G33/G31 Express Integrated Graphics
Driver: i915
Display: X.Org modesetting
OpenGL: 2.1 Mesa i915
```

---

# 2. Intel Drivers on Linux

All Intel GPUs use:

```
kernel driver: i915
```

You do NOT need to install a proprietary driver. In Intel, **all drivers are free**.

## 2.1 i915, i965 and iHD: do not confuse

They are different layers, and it is common to think that they are alternatives to each other:

| Name | What it is | Where it comes from |
|------|---------|----------|
| `i915` | **Kernel** driver (video/modesetting) | Already comes in the kernel, does not install anything |
| `i965` | Driver **VA-API** (video decoding) for GPUs up to ~Coffee Lake | Package `libva-intel-driver` |
| `iHD` | **VA-API** driver for Broadwell (5th generation) and newer | `intel-media-driver` package |
| `crocus` / `iris` | Mesa **OpenGL** driver (crocus: Gen4 to Haswell; iris: Broadwell+) | `mesa-dri` package |

That is: those who have HD 2000/3000 use `i915` (kernel) **and** `i965` (VA-API) at the same time.

## 2.2 Xorg Drivers

There are two possible Xorg drivers:

- modesetting → recommended
- xf86-video-intel → obsolete in most cases

Check which one is in use:

```bash
inxi -G
```

If showing:

```
driver: modesetting
```

It's correct.

---

# 3. For old Intel GPUs (GMA 950, 3000, 3100, X3100, X4500)

These GPUs have limitations:

- OpenGL 2.1 maximum
- Unstable DRI3
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

# 5. Disable XFCE compositor (MANDATORY on old GPUs)

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

# 6. Firefox Setup for Old Intel GPUs (Group A Only – GMA)

> **Only for GMA 950/3000/3100/X3100/X4500.** If your GPU is Intel HD (HD 2000/3000 or newer), **do not apply this step**: it turns off video acceleration. Go straight to section 7.

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

# 7. Video acceleration (VA-API) for Intel HD (Group B)

With VA-API, the GPU decodes the video and the CPU is free. This makes a big difference on laptops with 2nd/3rd generation Core i3/i5 watching YouTube.

## 7.1 What each GPU decodes in hardware

| GPU | Codecs with acceleration | VA-API Driver |
|-----|----------------------|---------------|
| GMA 950/3000/3100/X3100 | none | — |
| GMA X4500 | MPEG-2 | `i965` |
| HD Graphics (Ironlake, 1st generation Core) | MPEG-2, H.264, VC-1 | `i965` |
| HD 2000/3000 (Sandy Bridge) | MPEG-2, H.264, VC-1 | `i965` |
| HD 2500/4000 (Ivy Bridge) | MPEG-2, H.264, VC-1, JPEG | `i965` |
| HD 4200–5200 (Haswell) | MPEG-2, H.264, VC-1, JPEG | `i965` |
| Broadwell and newer | + VP8/HEVC/VP9/AV1 depending on the generation | `iHD` |

**No GPU up to Haswell decodes VP9 or AV1.** YouTube uses VP9/AV1 by default, so in these cases, install a browser extension that forces H.264 (e.g., *h264ify* or *enhanced-h264ify*). Without it, the video will revert to CPU decoding.

## 7.2 Install the packages

```bash
sudo xbps-install -S mesa-dri libva-intel-driver libva-utils intel-gpu-tools
```

Make sure the user is in the `video` group (log out and log back in afterwards):

```bash
sudo usermod -aG video $USER
```

## 7.3 Force the correct VA-API driver

Add in `~/.profile` (or in `/etc/environment`, without `export`):

```bash
export LIBVA_DRIVER_NAME=i965
```

## 7.4 Test

```bash
vainfo
```

Expected result (summarized):

```
vainfo: Driver version: Intel i965 driver for Intel(R) Sandybridge Mobile
      VAProfileH264Main               : VAEntrypointVLD
      VAProfileH264High               : VAEntrypointVLD
```

`VAEntrypointVLD` = hardware decoding available.

## 7.5 Firefox

In `about:config`:

```
media.ffmpeg.vaapi.enabled = true
```

In the newer versions of Firefox, if the decoding doesn't turn on by itself, also:

```
media.hardware-video-decoding.force-enabled = true
```

If you applied `user.js` from section 6, delete it. Restart Firefox and check in `about:support`, **Media** section, if H264 appears with hardware decoding.

## 7.6 Confirm that the GPU is working

Open a video on YouTube (with the H.264 extension active) and run:

```bash
sudo intel_gpu_top
```

If the **Video** line is busy while the video is playing, the decoding is on the GPU. CPU usage drops considerably.

---

# 8. Check OpenGL support

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

# 9. Check active acceleration

Execute:

```bash
glxinfo | grep "direct rendering"
```

Correct result:

```
direct rendering: Yes
```

---

# 10. Check loaded driver

Execute:

```bash
lsmod | grep i915
```

Result:

```
i915
```

---

# 11. Check Xorg status

Execute:

```bash
grep -i driver /var/log/Xorg.0.log
```

Correct result:

```
modesetting
```

---

# 12. Do not use xf86-video-intel on very old hardware

Remove if installed:

```bash
sudo xbps-remove xf86-video-intel
```

The modesetting driver is more stable.

---

# 13. Check DRI

Execute:

```bash
glxinfo | grep DRI
```

For older GPUs, use:

```
DRI2
```

---

# 14. Check resolution

Execute:

```bash
xrandr
```

---

# 15. Common problems and solution

## reboot when entering XFCE

Disable compositor:

```bash
xfconf-query -c xfwm4 -p /general/use_compositing -s false
```

---

## Firefox crashes

Disable acceleration via user.js (step 6). **GMA only** — on Intel HD, it is preferable to keep acceleration.

---

## YouTube video using a lot of CPU (Intel HD)

- Check if the extension that forces H.264 is active (the GPU does not decode VP9/AV1).
- Run `vainfo` and see if `VAProfileH264...: VAEntrypointVLD` appears.
- Confirm `LIBVA_DRIVER_NAME=i965` with `echo $LIBVA_DRIVER_NAME`.
- Confirm that `user.js` in section 6 is not applied (it turns off VA-API).

---

## black screen

Remove xf86-video-intel:

```bash
sudo xbps-remove xf86-video-intel
```

---

# 16. Complete check of the graphics system

Execute:

```bash
inxi -G
glxinfo | grep OpenGL
lsmod | grep i915
xfconf-query -c xfwm4 -p /general/use_compositing
vainfo          # somente Intel HD (Grupo B)
```

---

# 17. Intel GPUs and OpenGL support

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

# 18. Conclusion

**Group A – GMA (950, 3000, 3100, X3100, X4500):**

- driver kernel: i915
- driver Xorg: modesetting
- DRI2
- compositor disabled
- Firefox acceleration disabled

**Group B – Intel HD (Ironlake, HD 2000/3000, HD 2500/4000, Haswell):**

- driver kernel: i915
- driver Xorg: modesetting
- driver VA-API: i965 (`libva-intel-driver`) + `LIBVA_DRIVER_NAME=i965`
- VA-API enabled in Firefox
- extension that forces H.264 on YouTube

System will be stable and functional.
