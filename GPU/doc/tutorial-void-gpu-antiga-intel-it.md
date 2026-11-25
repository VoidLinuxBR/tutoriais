# Tutorial completo: GPU Intel su Linux (Void Linux, XFCE e hardware vecchio)

Questa guida spiega come identificare, configurare e stabilizzare le GPU Intel su Linux, in particolare hardware vecchio come GMA 3100, GMA X3100, HD 2000/3000, ecc.

> **Attenzione:** le vecchie GPU Intel sono divise in due gruppi, e la configurazione cambia:
>
> - **Gruppo A – GMA (950, 3000, 3100, X3100, X4500):** seguono i passaggi di stabilità (DRI2, compositore e accelerazione di Firefox disattivati). Non hanno decodifica video utile tramite hardware.
> - **Gruppo B – Intel HD (1ª generazione Core/Ironlake, HD 2000/3000/Sandy Bridge, HD 2500/4000/Ivy Bridge, Haswell):** in generale non necessitano degli aggiustamenti di stabilità e **possono e devono utilizzare l’accelerazione video (VA-API)** con il driver `i965` (vedere sezione 7).
>
> Usa la sezione 1 per scoprire in quale gruppo si trova la tua GPU.

---

# 1. Identificare la tua GPU Intel

Esegui:

```bash
lspci -k | grep -A3 VGA
```

ou:

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

# 2. Driver Intel su Linux

Tutte le GPU Intel usano:

```
kernel driver: i915
```

Non hai BISOGNO di installare driver proprietari. In Intel, **tutti i driver sono liberi**.

## 2.1 i915, i965 e iHD: non confondere

Sono strati diversi, ed è comune pensare che siano alternativi tra loro:

| Nome | Cos'è | Da dove viene |
|------|---------|----------|
| `i915` | Driver del **kernel** (video/modesetting) | Già incluso nel kernel, non installa nulla |
| `i965` | Driver **VA-API** (decodifica video) per GPU fino a ~Coffee Lake | Pacchetto `libva-intel-driver` |
| `iHD` | Driver **VA-API** per Broadwell (5ª generazione) e successive | Pacchetto `intel-media-driver` |
| `crocus` / `iris` | Driver **OpenGL** di Mesa (crocus: Gen4 a Haswell; iris: Broadwell+) | Pacchetto `mesa-dri` |

Cioè: chi ha HD 2000/3000 usa `i915` (kernel) **e** `i965` (VA-API) allo stesso tempo.

## 2.2 Driver Xorg

Esistono due driver Xorg possibili:

- modesetting → raccomandato
- xf86-video-intel → obsoleto nella maggior parte dei casi

Verifica quale è in uso:

```bash
inxi -G
```

Se mostrare:

```
driver: modesetting
```

È corretto.

---

# 3. Per GPU Intel vecchie (GMA 950, 3000, 3100, X3100, X4500)

Queste GPU hanno delle limitazioni:

- OpenGL 2.1 massimo
- DRI3 instabile
- il compositore causa blocco
- l'accelerazione moderna non funziona correttamente

---

# 4. Configurazione stabile raccomandata

Crea:

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

# 5. Disattivare il compositore XFCE (OBBLIGATORIO su GPU vecchie)

Esegui:

```bash
xfconf-query -c xfwm4 -p /general/use_compositing -s false
```

Verifica:

```bash
xfconf-query -c xfwm4 -p /general/use_compositing
```

Risultato atteso:

```
false
```

---

# 6. Configurazione di Firefox per GPU Intel vecchie (solo Gruppo A – GMA)

> **Solo per GMA 950/3000/3100/X3100/X4500.** Se la tua GPU è Intel HD (HD 2000/3000 o più recente), **non applicare questo passaggio**: disattiva l'accelerazione video. Vai direttamente alla sezione 7.

Crea:

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

Ciò evita i blocchi.

---

# 7. Accelerazione video (VA-API) per Intel HD (Gruppo B)

Con VA-API, la GPU decodifica il video e la CPU rimane libera. Questo fa molta differenza nei notebook con Core i3/i5 di 2ª/3ª generazione mentre guardano YouTube.

## 7.1 Cosa decodifica ogni GPU in hardware

| GPU | Codec con accelerazione | Driver VA-API |
|-----|----------------------|---------------|
| GMA 950/3000/3100/X3100 | nessuno | — |
| GMA X4500 | MPEG-2 | `i965` |
| HD Graphics (Ironlake, 1ª generazione Core) | MPEG-2, H.264, VC-1 | `i965` |
| HD 2000/3000 (Sandy Bridge) | MPEG-2, H.264, VC-1 | `i965` |
| HD 2500/4000 (Ivy Bridge) | MPEG-2, H.264, VC-1, JPEG | `i965` |
| HD 4200–5200 (Haswell) | MPEG-2, H.264, VC-1, JPEG | `i965` |
| Broadwell e più recenti | + VP8/HEVC/VP9/AV1 secondo la generazione | `iHD` |

**Nessuna GPU fino a Haswell decodifica VP9 o AV1.** YouTube usa VP9/AV1 di default, quindi in questi casi installa nel browser un'estensione che forza H.264 (es.: *h264ify* o *enhanced-h264ify*). Senza di essa il video torna a essere decodificato dalla CPU.

## 7.2 Installare i pacchetti

```bash
sudo xbps-install -S mesa-dri libva-intel-driver libva-utils intel-gpu-tools
```

Assicurati che l'utente sia nel gruppo `video` (esci e rientra nella sessione dopo):

```bash
sudo usermod -aG video $USER
```

## 7.3 Forzare il driver VA-API corretto

Aggiungi in `~/.profile` (o in `/etc/environment`, senza `export`):

```bash
export LIBVA_DRIVER_NAME=i965
```

## 7.4 Testare

```bash
vainfo
```

Risultato atteso (riassunto):

```
vainfo: Driver version: Intel i965 driver for Intel(R) Sandybridge Mobile
      VAProfileH264Main               : VAEntrypointVLD
      VAProfileH264High               : VAEntrypointVLD
```

`VAEntrypointVLD` = decodifica hardware disponibile.

## 7,5 Firefox

In `about:config`:

```
media.ffmpeg.vaapi.enabled = true
```

Nelle versioni più recenti di Firefox, se la decodifica non si attiva da sola, anche:

```
media.hardware-video-decoding.force-enabled = true
```

Se hai applicato il `user.js` della sezione 6, cancellalo. Riavvia Firefox e controlla in `about:support`, sezione **Media**, se H264 appare con decodifica hardware.

## 7.6 Confermare che la GPU stia funzionando

Apri un video su YouTube (con l'estensione H.264 attiva) e avvia:

```bash
sudo intel_gpu_top
```

Se la linea **Video** risulta occupata mentre il video è in riproduzione, la decodifica è sulla GPU. L'utilizzo della CPU diminuisce notevolmente.

---

# 8. Verificare il supporto OpenGL

Esegui:

```bash
glxinfo | grep "OpenGL version"
```

Esempio:

```
OpenGL version string: 2.1 Mesa 25.3.3
```

Questo è il massimo supportato dalla GMA 3100.

---

# 9. Verificare l'accelerazione attiva

Esegui:

```bash
glxinfo | grep "direct rendering"
```

Risultato corretto:

```
direct rendering: Yes
```

---

# 10. Verificare driver caricato

Esegui:

```bash
lsmod | grep i915
```

Risultato:

```
i915
```

---

# 11. Verificare lo stato di Xorg

Esegui:

```bash
grep -i driver /var/log/Xorg.0.log
```

Risultato corretto:

```
modesetting
```

---

# 12. Non usare xf86-video-intel su hardware molto vecchio

Rimuovere se installato:

```bash
sudo xbps-remove xf86-video-intel
```

Il driver modesetting è più stabile.

---

# 13. Verificare DRI

Esegui:

```bash
glxinfo | grep DRI
```

Per GPU vecchie, usa:

```
DRI2
```

---

# 14. Verificare la risoluzione

Esegui:

```bash
xrandr
```

---

# 15. Problemi comuni e soluzioni

## riavvio all'avvio di XFCE

Disattiva compositore:

```bash
xfconf-query -c xfwm4 -p /general/use_compositing -s false
```

---

## Firefox si blocca

Disattiva l'accelerazione tramite user.js (passo 6). **Solo GMA** — su Intel HD, è meglio mantenere l'accelerazione.

---

## video di YouTube che usa molta CPU (Intel HD)

- Verifica se l'estensione che forza H.264 è attiva (la GPU non decodifica VP9/AV1).
- Avvia `vainfo` e verifica se appare `VAProfileH264...: VAEntrypointVLD`.
- Conferma `LIBVA_DRIVER_NAME=i965` con `echo $LIBVA_DRIVER_NAME`.
- Confermi che il `user.js` della sezione 6 non sia applicato (esso disattiva il VA-API).

---

## tela nera

Rimuovere xf86-video-intel:

```bash
sudo xbps-remove xf86-video-intel
```

---

# 16. Verifica completa del sistema grafico

Esegui:

```bash
inxi -G
glxinfo | grep OpenGL
lsmod | grep i915
xfconf-query -c xfwm4 -p /general/use_compositing
vainfo          # somente Intel HD (Grupo B)
```

---

# 17. GPU Intel e supporto OpenGL

| GPU | OpenGL |
|----|--------|
| GMA 950 | 2.1 |
| GMA 3100 | 2.1 |
| X3100 | 2.1 |
| X4500 | 2.1 |
| HD 2000 | 3.3 |
| HD 3000 | 3.3 |
| HD 4000 | 4.0 |
| UHD 620 | 4,6 |

---

# 18. Conclusione

**Gruppo A – GMA (950, 3000, 3100, X3100, X4500):**

- driver kernel: i915
- driver Xorg: impostazione modalità
- DRI2
- compositore disattivato
- accelerazione Firefox disattivata

**Gruppo B – Intel HD (Ironlake, HD 2000/3000, HD 2500/4000, Haswell):**

- driver kernel: i915
- driver Xorg: impostazione modalità
- driver VA-API: i965 (`libva-intel-driver`) + `LIBVA_DRIVER_NAME=i965`
- VA-API attivato in Firefox
- estensione che forza H.264 su YouTube

Il sistema resterà stabile e funzionale.
