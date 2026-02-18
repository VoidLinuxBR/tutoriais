# Vollständiges Tutorial: Intel-GPUs unter Linux (Void Linux, XFCE und alte Hardware)

In dieser Anleitung wird erläutert, wie Sie Intel-GPUs unter Linux identifizieren, konfigurieren und stabilisieren, insbesondere ältere Hardware wie GMA 3100, GMA X3100, HD 2000/3000 usw.

---

# 1. Identifizieren Sie Ihre Intel-GPU

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

# 2. Treiber Intel kein Linux

Alle Intel-GPUs verwenden:

```
kernel driver: i915
```

Sie müssen KEINEN proprietären Treiber installieren.

Es gibt zwei mögliche Xorg-Treiber:

- Moduseinstellung → empfohlen
- xf86-video-intel → in den meisten Fällen veraltet

Überprüfen Sie, welches verwendet wird:

```bash
inxi -G
```

Wenn Folgendes angezeigt wird:

```
driver: modesetting
```

Es ist richtig.

---

# 3. Für ältere Intel-GPUs (GMA 950, 3000, 3100, X3100, X4500)

Diese GPUs haben Einschränkungen:

- Maximal OpenGL 2.1
- DRI3 instabil
- Composer verursacht Absturz
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

# 5. Deaktivieren Sie XFCE Composer (auf alten GPUs ERFORDERLICH)

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

# 6. Firefox für ältere Intel-GPUs einrichten

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

Dies verhindert Abstürze.

---

# 7. Überprüfen Sie die OpenGL-Unterstützung

Ausführen:

```bash
glxinfo | grep "OpenGL version"
```

Beispiel:

```
OpenGL version string: 2.1 Mesa 25.3.3
```

Dies ist das Maximum, das vom GMA 3100 unterstützt wird.

---

# 8. Aktive Beschleunigung prüfen

Ausführen:

```bash
glxinfo | grep "direct rendering"
```

Richtiges Ergebnis:

```
direct rendering: Yes
```

---

# 9. Überprüfen Sie den geladenen Treiber

Ausführen:

```bash
lsmod | grep i915
```

Ergebnis:

```
i915
```

---

# 10. Überprüfen Sie den Xorg-Status

Ausführen:

```bash
grep -i driver /var/log/Xorg.0.log
```

Richtiges Ergebnis:

```
modesetting
```

---

# 11. Verwenden Sie xf86-video-intel nicht auf sehr alter Hardware

Entfernen, falls installiert:

```bash
sudo xbps-remove xf86-video-intel
```

Der Moduseinstellungstreiber ist stabiler.

---

# 12. DRI prüfen

Ausführen:

```bash
glxinfo | grep DRI
```

Für ältere GPUs verwenden Sie:

```
DRI2
```

---

# 13. Auflösung prüfen

Ausführen:

```bash
xrandr
```

---

# 14. Häufige Probleme und Lösungen

## Neustart beim Aufrufen von XFCE

Composer deaktivieren:

```bash
xfconf-query -c xfwm4 -p /general/use_compositing -s false
```

---

## Firefox stürzt ab

Beschleunigung über user.js deaktivieren (Schritt 6)

---

## schwarzer Bildschirm

xf86-video-intel entfernen:

```bash
sudo xbps-remove xf86-video-intel
```

---

# 15. Vollständige Überprüfung des Grafiksystems

Ausführen:

```bash
inxi -G
glxinfo | grep OpenGL
lsmod | grep i915
xfconf-query -c xfwm4 -p /general/use_compositing
```

---

# 16. Intel-GPUs und OpenGL-Unterstützung

| GPU | OpenGL |
|----|--------|
| GMA 950 | 2.1 |
| GMA 3100 | 2.1 |
| X3100 | 2.1 |
| X4500 | 2.1 |
| HD 2000 | 3.3 |
| HD 3000 | 3.3 |
| HD 4000 | 4,0 |
| UHD 620 | 4,6 |

---

# 17. Fazit

Verwenden Sie für ältere Intel-GPUs:

- Treiberkernel: i915
- Treiber Xorg: Moduseinstellung
- DRI2
- Komponist behindert
- Firefox-Beschleunigung deaktiviert

Das System wird stabil und funktionsfähig sein.
