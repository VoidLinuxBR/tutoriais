# Komplettes Tutorial: Intel-GPUs unter Linux (Void Linux, XFCE und alte Hardware)

Dieser Leitfaden erklärt, wie man Intel-GPUs unter Linux erkennt, konfiguriert und stabilisiert, insbesondere ältere Hardware wie GMA 3100, GMA X3100, HD 2000/3000 usw.

> **Achtung:** Alte Intel-GPUs werden in zwei Gruppen unterteilt, und die Konfiguration ändert sich:
>
> - **Gruppe A – GMA (950, 3000, 3100, X3100, X4500):** folgen den Stabilitätsschritten (DRI2, Compositor und Hardwarebeschleunigung von Firefox deaktiviert). Haben keine nützliche Hardware-Video-Decodierung.
> - **Gruppe B – Intel HD (1. Generation Core/Ironlake, HD 2000/3000/Sandy Bridge, HD 2500/4000/Ivy Bridge, Haswell):** im Allgemeinen benötigen sie keine Stabilitätsanpassungen und **können und sollten die Video-Beschleunigung (VA-API)** mit dem Treiber `i965` verwenden (siehe Abschnitt 7).
>
> Verwenden Sie Abschnitt 1, um herauszufinden, in welcher Gruppe sich Ihre GPU befindet.

---

# Schneller Weg: `voidbr-gpu-config`

In VoidBR erledigt das Dienstprogramm `voidbr-gpu-config` (Paket `voidbr-utils`) automatisch, was dieses Tutorial erklärt. Es funktioniert mit **Intel- und AMD-GPUs**:

- erkennt die GPU und die Generation (GMA, Intel HD, AMD);
- installiere die richtigen Pakete (`libva-intel-driver`, `intel-media-driver` oder `mesa-vaapi`);
- teste mit `vainfo`, welcher VA-API-Treiber wirklich funktioniert, und setze `LIBVA_DRIVER_NAME`;
- Fügen Sie den Benutzer zu den Gruppen `video` und `render` hinzu;
- konfiguriere **Firefox**, **Google Chrome** und **Brave** (VA-API aktiviert auf Intel HD/AMD, oder Stabilitätsprofil auf GMA);
- Sichert alles und ermöglicht das Rückgängigmachen.

Zuerst, nur die Diagnose (ändert nichts):

```bash
voidbr-gpu-config --check
```

Die empfohlene Einstellung anwenden:

```bash
sudo voidbr-gpu-config --apply
```

Rückgängig machen:

```bash
sudo voidbr-gpu-config --revert
```

Es ist auch im Anwendungsmenü als **VoidBR GPU Config** zu finden. Nach dem Anwenden, melden Sie sich ab und wieder an der Sitzung an.

Wenn etwas nicht funktioniert, führen Sie `voidbr-gpu-config --check` aus und posten Sie die Ausgabe in der Telegram-Gruppe: **https://t.me/voidlinux_br**

Wenn Sie alles manuell machen möchten, folgen Sie den untenstehenden Abschnitten.

---

# 1. Ihre Intel-GPU identifizieren

Ausführen:

```bash
lspci -k | grep -A3 VGA
```

oder:

```bash
inxi -G
```

Beispielausgabe:

```
Device: Intel 82G33/G31 Express Integrated Graphics
Driver: i915
Display: X.Org modesetting
OpenGL: 2.1 Mesa i915
```

---

# 2. Intel-Treiber unter Linux

Alle Intel-GPUs verwenden:

```
kernel driver: i915
```

Sie müssen keinen proprietären Treiber installieren. Bei Intel **sind alle Treiber frei**.

## 2.1 i915, i965 und iHD: nicht verwechseln

Es sind verschiedene Schichten, und es ist üblich zu denken, dass sie Alternativen zueinander sind:

| Name | Was ist es | Woher kommt es |
|------|---------|----------|
| `i915` | **Kernel**-Treiber (Video/Modesetting) | Kommt bereits im Kernel, installiert nichts |
| `i965` | Treiber **VA-API** (Video-Decodierung) für GPUs bis ~Coffee Lake | Paket `libva-intel-driver` |
| `iHD` | Treiber **VA-API** für Broadwell (5. Generation) und neuere | Paket `intel-media-driver` |
| `crocus` / `iris` | Mesa **OpenGL**-Treiber (crocus: Gen4 auf Haswell; iris: Broadwell+) | Paket `mesa-dri` |

Das heißt: Wer eine HD 2000/3000 hat, verwendet gleichzeitig das `i915` (Kernel) **und** das `i965` (VA-API).

## 2.2 Treiber Xorg

Es gibt zwei mögliche Xorg-Treiber:

- modesetting → empfohlen
- xf86-video-intel → in den meisten Fällen veraltet

Überprüfen Sie, welches verwendet wird:

```bash
inxi -G
```

Wenn angezeigt:

```
driver: modesetting
```

Es ist richtig.

---

# 3. Für alte Intel-GPUs (GMA 950, 3000, 3100, X3100, X4500)

Diese GPUs haben Einschränkungen:

- OpenGL 2.1 maximum
- Instabiles DRI3
- Komponist verursacht Absturz
- moderne Beschleunigung funktioniert nicht richtig

---

# 4. Empfohlene stabile Konfiguration

Erstellen:

```bash
sudo mkdir -p /etc/X11/xorg.conf.d
sudo nano /etc/X11/xorg.conf.d/20-intel.conf
```

Inhalt:

```conf
Section "Device"
    Identifier "Intel Graphics"
    Driver "modesetting"
    Option "DRI" "2"
EndSection
```

---

# 5. XFCE-Composer deaktivieren (OBLIGATORISCH bei alten GPUs)

Ausführen:

```bash
xfconf-query -c xfwm4 -p /general/use_compositing -s false
```

Überprüfen:

```bash
xfconf-query -c xfwm4 -p /general/use_compositing
```

Erwartetes Ergebnis:

```
false
```

---

# 6. Firefox-Konfiguration für ältere Intel-GPUs (nur Gruppe A – GMA)

> **Nur für GMA 950/3000/3100/X3100/X4500.** Wenn Ihre GPU Intel HD (HD 2000/3000 oder neuer) ist, **führen Sie diesen Schritt nicht aus**: Er schaltet die Video-Beschleunigung aus. Gehen Sie direkt zu Abschnitt 7.

Erstellen:

```bash
nano ~/.mozilla/firefox/*.default-release/user.js
```

Inhalt:

```js
user_pref("layers.acceleration.disabled", true);
user_pref("gfx.webrender.force-disabled", true);
user_pref("gfx.xrender.enabled", true);
user_pref("media.ffmpeg.vaapi.enabled", false);
```

Das verhindert Abstürze.

---

# 7. Videobeschleunigung (VA-API) für Intel HD (Gruppe B)

Mit VA-API decodiert die GPU das Video und die CPU bleibt frei. Das macht einen großen Unterschied bei Notebooks mit Core i3/i5 der 2./3. Generation beim Ansehen von YouTube.

## 7.1 Was jede GPU in Hardware decodiert

| GPU | Beschleunigte Codecs | VA-API-Treiber |
|-----|----------------------|---------------|
| GMA 950/3000/3100/X3100 | keine | — |
| GMA X4500 | MPEG-2 | `i965` |
| HD-Grafik (Ironlake, 1. Generation Core) | MPEG-2, H.264, VC-1 | `i965` |
| HD 2000/3000 (Sandy Bridge) | MPEG-2, H.264, VC-1 | `i965` |
| HD 2500/4000 (Ivy Bridge) | MPEG-2, H.264, VC-1, JPEG | `i965` |
| HD 4200–5200 (Haswell) | MPEG-2, H.264, VC-1, JPEG | `i965` |
| Broadwell und neuer | + VP8/HEVC/VP9/AV1 je nach Generation | `iHD` |

**Keine GPU vor Haswell decodiert VP9 oder AV1.** YouTube verwendet standardmäßig VP9/AV1, also installieren Sie in diesen Fällen eine Erweiterung im Browser, die H.264 erzwingt (z.B.: *h264ify* oder *enhanced-h264ify*). Ohne sie wird das Video wieder von der CPU decodiert.

## 7.2 Die Pakete installieren

```bash
sudo xbps-install -S mesa-dri libva-intel-driver libva-utils intel-gpu-tools
```

Stellen Sie sicher, dass der Benutzer in der Gruppe `video` ist (melden Sie sich ab und wieder an der Sitzung an):

```bash
sudo usermod -aG video $USER
```

## 7.3 Den richtigen VA-API-Treiber erzwingen

Fügen Sie in `~/.profile` (oder in `/etc/environment`, ohne `export`) hinzu:

```bash
export LIBVA_DRIVER_NAME=i965
```

## 7.4 Testen

```bash
vainfo
```

Erwartetes Ergebnis (zusammengefasst):

```
vainfo: Driver version: Intel i965 driver for Intel(R) Sandybridge Mobile
      VAProfileH264Main               : VAEntrypointVLD
      VAProfileH264High               : VAEntrypointVLD
```

`VAEntrypointVLD` = Hardwaredekodierung verfügbar.

## 7,5 Firefox

In `about:config`:

```
media.ffmpeg.vaapi.enabled = true
```

In den neueren Versionen von Firefox, wenn die Dekodierung sich nicht von selbst einschaltet, auch:

```
media.hardware-video-decoding.force-enabled = true
```

Wenn Sie das `user.js` aus Abschnitt 6 angewendet haben, löschen Sie es. Starten Sie Firefox neu und prüfen Sie in `about:support`, Abschnitt **Medien**, ob H264 mit Hardware-Dekodierung angezeigt wird.

## 7.6 Bestätigen, dass die GPU arbeitet

Öffnen Sie ein Video auf YouTube (mit aktiviertem H.264-Codec) und führen Sie aus:

```bash
sudo intel_gpu_top
```

Wenn die **Video**-Leitung während der Wiedergabe des Videos belegt ist, erfolgt die Dekodierung auf der GPU. Die CPU-Auslastung sinkt deutlich.

---

# 8. OpenGL-Unterstützung überprüfen

Ausführen:

```bash
glxinfo | grep "OpenGL version"
```

Beispiel:

```
OpenGL version string: 2.1 Mesa 25.3.3
```

Das ist das Maximum, das von der GMA 3100 unterstützt wird.

---

# 9. Überprüfung der aktiven Beschleunigung

Ausführen:

```bash
glxinfo | grep "direct rendering"
```

Richtiges Ergebnis:

```
direct rendering: Yes
```

---

# 10. Überprüfen des geladenen Treibers

Ausführen:

```bash
lsmod | grep i915
```

Ergebnis:

```
i915
```

---

# 11. Überprüfen Sie den Status von Xorg

Ausführen:

```bash
grep -i driver /var/log/Xorg.0.log
```

Richtiges Ergebnis:

```
modesetting
```

---

# 12. Verwenden Sie xf86-video-intel nicht auf sehr alter Hardware

Entfernen, falls installiert:

```bash
sudo xbps-remove xf86-video-intel
```

Der Treiber Modesetting ist stabiler.

---

# 13. DRI überprüfen

Ausführen:

```bash
glxinfo | grep DRI
```

Für alte GPUs verwenden Sie:

```
DRI2
```

---

# 14. Auflösung überprüfen

Ausführen:

```bash
xrandr
```

---

# 15. Häufige Probleme und Lösungen

## Neustart beim Einloggen in XFCE

Compositor deaktivieren:

```bash
xfconf-query -c xfwm4 -p /general/use_compositing -s false
```

---

## Firefox stürzt ab

Deaktivieren Sie die Beschleunigung über user.js (Schritt 6). **Nur GMA** — bei Intel HD ist es besser, die Beschleunigung beizubehalten.

---

## YouTube-Video, das viel CPU verwendet (Intel HD)

- Überprüfen Sie, ob die Erweiterung, die H.264 erzwingt, aktiviert ist (die GPU dekodiert VP9/AV1 nicht).
- Führen Sie `vainfo` aus und sehen Sie, ob `VAProfileH264...: VAEntrypointVLD` angezeigt wird.
- Bestätigen Sie `LIBVA_DRIVER_NAME=i965` mit `echo $LIBVA_DRIVER_NAME`.
- Bestätigen Sie, dass `user.js` aus Abschnitt 6 nicht angewendet wird (er schaltet die VA-API aus).

---

## schwarzer Bildschirm

Entfernen Sie xf86-video-intel:

```bash
sudo xbps-remove xf86-video-intel
```

---

# 16. Vollständige Überprüfung des Grafiksystems

Ausführen:

```bash
inxi -G
glxinfo | grep OpenGL
lsmod | grep i915
xfconf-query -c xfwm4 -p /general/use_compositing
vainfo          # somente Intel HD (Grupo B)
```

---

# 17. Intel-GPUs und OpenGL-Unterstützung

| GPU | OpenGL |
|----|--------|
| GMA 950 | 2,1 |
| GMA 3100 | 2.1 |
| X3100 | 2,1 |
| X4500 | 2.1 |
| HD 2000 | 3,3 |
| HD 3000 | 3,3 |
| HD 4000 | 4,0 |
| UHD 620 | 4,6 |

---

# 18. Fazit

**Gruppe A – GMA (950, 3000, 3100, X3100, X4500):**

- Treiber-Kernel: i915
- Treiber Xorg: Modi-Einstellung
- DRI2
- Deaktivierter Komponist
- Firefox-Beschleunigung deaktiviert

**Gruppe B – Intel HD (Ironlake, HD 2000/3000, HD 2500/4000, Haswell):**

- Treiber-Kernel: i915
- Treiber Xorg: Modi-Einstellung
- Treiber VA-API: i965 (`libva-intel-driver`) + `LIBVA_DRIVER_NAME=i965`
- VA-API im Firefox aktiviert
- Erweiterung, die H.264 auf YouTube erzwingt

Das System wird stabil und funktional bleiben.
