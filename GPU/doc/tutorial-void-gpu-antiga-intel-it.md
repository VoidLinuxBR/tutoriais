# Tutorial completo: GPU Intel su Linux (Void Linux, XFCE e vecchio hardware)

Questa guida spiega come identificare, configurare e stabilizzare le GPU Intel su Linux, in particolare l'hardware più vecchio come GMA 3100, GMA X3100, HD 2000/3000, ecc.

---

# 1. Identifica la tua GPU Intel

Eseguire:

```bash
lspci -k | grep -A3 VGA
```

O:

```bash
inxi -G
```

Esempio di output:

```
Device: Intel 82G33/G31 Express Integrated Graphics
Driver: i915
Display: X.Org modesetting
OpenGL: 2.1 Mesa i915
```

---

# 2. Driver Intel, non Linux

Tutte le GPU Intel utilizzano:

```
kernel driver: i915
```

NON è necessario installare driver proprietari.

Esistono due possibili driver Xorg:

- impostazione modalità → consigliata
- xf86-video-intel → obsoleto nella maggior parte dei casi

Controlla quale è in uso:

```bash
inxi -G
```

Se mostra:

```
driver: modesetting
```

È corretto.

---

# 3. Per GPU Intel meno recenti (GMA 950, 3000, 3100, X3100, X4500)

Queste GPU hanno limitazioni:

- OpenGL 2.1 massimo
- DRI3 instabile
- il compositore causa un arresto anomalo
- l'accelerazione moderna non funziona correttamente

---

# 4. Configurazione stabile consigliata

Creare:

```bash
sudo mkdir -p /etc/X11/xorg.conf.d
sudo nano /etc/X11/xorg.conf.d/20-intel.conf
```

Contenuto:

```conf
Section "Device"
    Identifier "Intel Graphics"
    Driver "modesetting"
    Option "DRI" "2"
EndSection
```

---

# 5. Disabilita il compositore XFCE (RICHIESTO su vecchie GPU)

Eseguire:

```bash
xfconf-query -c xfwm4 -p /general/use_compositing -s false
```

Controllo:

```bash
xfconf-query -c xfwm4 -p /general/use_compositing
```

Risultato atteso:

```
false
```

---

# 6. Configurazione di Firefox per GPU Intel meno recenti

Creare:

```bash
nano ~/.mozilla/firefox/*.default-release/user.js
```

Contenuto:

```js
user_pref("layers.acceleration.disabled", true);
user_pref("gfx.webrender.force-disabled", true);
user_pref("gfx.xrender.enabled", true);
user_pref("media.ffmpeg.vaapi.enabled", false);
```

Ciò impedisce arresti anomali.

---

# 7. Controlla il supporto OpenGL

Eseguire:

```bash
glxinfo | grep "OpenGL version"
```

Esempio:

```
OpenGL version string: 2.1 Mesa 25.3.3
```

Questo è il massimo supportato dal GMA 3100.

---

# 8. Controllare l'accelerazione attiva

Eseguire:

```bash
glxinfo | grep "direct rendering"
```

Risultato corretto:

```
direct rendering: Yes
```

---

# 9. Controllare il driver caricato

Eseguire:

```bash
lsmod | grep i915
```

Risultato:

```
i915
```

---

# 10. Controlla lo stato di Xorg

Eseguire:

```bash
grep -i driver /var/log/Xorg.0.log
```

Risultato corretto:

```
modesetting
```

---

# 11. Non utilizzare xf86-video-intel su hardware molto vecchio

Rimuovere se installato:

```bash
sudo xbps-remove xf86-video-intel
```

Il driver con impostazione mode è più stabile.

---

# 12. Controlla il DRI

Eseguire:

```bash
glxinfo | grep DRI
```

Per le GPU più vecchie utilizzare:

```
DRI2
```

---

# 13. Controllare la risoluzione

Eseguire:

```bash
xrandr
```

---

# 14. Problemi comuni e soluzioni

## riavviare quando si accede a XFCE

Disabilita compositore:

```bash
xfconf-query -c xfwm4 -p /general/use_compositing -s false
```

---

## Firefox si blocca

Disabilita l'accelerazione tramite user.js (passaggio 6)

---

## schermo nero

Rimuovere xf86-video-intel:

```bash
sudo xbps-remove xf86-video-intel
```

---

# 15. Controllo completo del sistema grafico

Eseguire:

```bash
inxi -G
glxinfo | grep OpenGL
lsmod | grep i915
xfconf-query -c xfwm4 -p /general/use_compositing
```

---

# 16. GPU Intel e supporto OpenGL

| GPU | OpenGL |
|----|--------|
| GMA950 | 2.1 |
| GMA3100 | 2.1 |
| X3100| 2.1 |
| X4500| 2.1 |
| HD2000 | 3.3 |
| HD3000| 3.3 |
| HD4000| 4.0 |
| UHD620| 4.6|

---

# 17. Conclusione

Per le GPU Intel meno recenti, utilizzare:

- kernel del driver: i915
- driver Xorg: impostazione mode
- DRI2
- compositore disabilitato
- Accelerazione di Firefox disabilitata

Il sistema sarà stabile e funzionale.
