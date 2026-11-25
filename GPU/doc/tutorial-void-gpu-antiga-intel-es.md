# Tutorial completo: GPUs Intel en Linux (Void Linux, XFCE y hardware antiguo)

Esta guía explica cómo identificar, configurar y estabilizar GPUs Intel en Linux, especialmente hardware antiguo como GMA 3100, GMA X3100, HD 2000/3000, etc.

> **Atención:** las GPUs Intel antiguas se dividen en dos grupos, y la configuración cambia:
>
> - **Grupo A – GMA (950, 3000, 3100, X3100, X4500):** siguen los pasos de estabilidad (DRI2, compositor y aceleración de Firefox desactivados). No tienen decodificación de vídeo útil por hardware.
> - **Grupo B – Intel HD (1.ª generación Core/Ironlake, HD 2000/3000/Sandy Bridge, HD 2500/4000/Ivy Bridge, Haswell):** en general no necesitan ajustes de estabilidad y **pueden y deben usar aceleración de vídeo (VA-API)** con el controlador `i965` (ver sección 7).
>
> Usa la sección 1 para descubrir en qué grupo está tu GPU.

---

# Manera rápida: `voidbr-gpu-config`

En VoidBR, la utilidad `voidbr-gpu-config` (paquete `voidbr-utils`) hace automáticamente lo que este tutorial explica. Funciona con GPUs **Intel y AMD**:

- detecta la GPU y la generación (GMA, Intel HD, AMD);
- instala los paquetes correctos (`libva-intel-driver`, `intel-media-driver` o `mesa-vaapi`);
- prueba con `vainfo` qué controlador VA-API realmente funciona y establece el `LIBVA_DRIVER_NAME`;
- coloca al usuario en los grupos `video` y `render`;
- configura **Firefox**, **Google Chrome** y **Brave** (VA-API activado en Intel HD/AMD, o perfil de estabilidad en GMA);
- hace copia de seguridad de todo y permite deshacer.

Primero, solo el diagnóstico (no cambia nada):

```bash
voidbr-gpu-config --check
```

Aplicar la configuración recomendada:

```bash
sudo voidbr-gpu-config --apply
```

Deshacer:

```bash
sudo voidbr-gpu-config --revert
```

También está en el menú de aplicaciones como **VoidBR GPU Config**. Después de aplicar, salga y vuelva a entrar en la sesión.

Si algo no funciona, ejecute `voidbr-gpu-config --check` y publique la salida en el grupo de Telegram: **https://t.me/voidlinux_br**

Si prefiere hacer todo manualmente, siga las secciones a continuación.

---

# 1. Identificar su GPU Intel

Ejecutar:

```bash
lspci -k | grep -A3 VGA
```

ou:

```bash
inxi -G
```

Ejemplo de salida:

```
Device: Intel 82G33/G31 Express Integrated Graphics
Driver: i915
Display: X.Org modesetting
OpenGL: 2.1 Mesa i915
```

---

# 2. Controladores Intel en Linux

Todas las GPUs Intel usan:

```
kernel driver: i915
```

Usted NO necesita instalar un controlador propietario. En Intel, **todos los controladores son libres**.

## 2.1 i915, i965 e iHD: no confundas

Son capas diferentes, y es común pensar que son alternativas entre sí:

| Nombre | Qué es | De dónde viene |
|------|---------|----------|
| `i915` | Controlador del **kernel** (vídeo/modesetting) | Ya viene en el kernel, no instala nada |
| `i965` | Controlador **VA-API** (decodificación de video) para GPUs hasta ~Coffee Lake | Paquete `libva-intel-driver` |
| `iHD` | Controlador **VA-API** para Broadwell (5.ª generación) y más recientes | Paquete `intel-media-driver` |
| `crocus` / `iris` | Controlador **OpenGL** de Mesa (crocus: Gen4 a Haswell; iris: Broadwell+) | Paquete `mesa-dri` |

Es decir: quien tiene HD 2000/3000 usa el `i915` (kernel) **y** el `i965` (VA-API) al mismo tiempo.

## 2.2 Controladores Xorg

Existen dos controladores Xorg posibles:

- modesetting → recomendado
- xf86-video-intel → obsoleto en la mayoría de los casos

Verifique cuál está en uso:

```bash
inxi -G
```

Se mostrará:

```
driver: modesetting
```

Está correcto.

---

# 3. Para GPUs Intel antiguas (GMA 950, 3000, 3100, X3100, X4500)

Esas GPUs tienen limitaciones:

- OpenGL 2.1 máximo
- DRI3 inestable
- compositor causa bloqueo
- la aceleración moderna no funciona correctamente

---

# 4. Configuración estable recomendada

Cree:

```bash
sudo mkdir -p /etc/X11/xorg.conf.d
sudo nano /etc/X11/xorg.conf.d/20-intel.conf
```

Contenido:

```conf
Section "Device"
    Identifier "Intel Graphics"
    Driver "modesetting"
    Option "DRI" "2"
EndSection
```

---

# 5. Desactivar el compositor XFCE (OBLIGATORIO en GPU antiguas)

Ejecutar:

```bash
xfconf-query -c xfwm4 -p /general/use_compositing -s false
```

Verifique:

```bash
xfconf-query -c xfwm4 -p /general/use_compositing
```

Resultado esperado:

```
false
```

---

# 6. Configuración de Firefox para GPUs Intel antiguas (solo Grupo A – GMA)

> **Solo para GMA 950/3000/3100/X3100/X4500.** Si tu GPU es Intel HD (HD 2000/3000 o más reciente), **no apliques este paso**: desactiva la aceleración de video. Ve directamente a la sección 7.

Cree:

```bash
nano ~/.mozilla/firefox/*.default-release/user.js
```

Contenido:

```js
user_pref("layers.acceleration.disabled", true);
user_pref("gfx.webrender.force-disabled", true);
user_pref("gfx.xrender.enabled", true);
user_pref("media.ffmpeg.vaapi.enabled", false);
```

Eso evita bloqueos.

---

# 7. Aceleración de video (VA-API) para Intel HD (Grupo B)

Con VA-API, la GPU decodifica el video y la CPU queda libre. Esto hace mucha diferencia en los portátiles con Core i3/i5 de 2.ª/3.ª generación al ver YouTube.

## 7.1 Qué decodifica cada GPU en hardware

| GPU | Códecs con aceleración | Controlador VA-API |
|-----|----------------------|---------------|
| GMA 950/3000/3100/X3100 | ninguno | — |
| GMA X4500 | MPEG-2 | `i965` |
| HD Graphics (Ironlake, 1ª generación Core) | MPEG-2, H.264, VC-1 | `i965` |
| HD 2000/3000 (Sandy Bridge) | MPEG-2, H.264, VC-1 | `i965` |
| HD 2500/4000 (Ivy Bridge) | MPEG-2, H.264, VC-1, JPEG | `i965` |
| HD 4200–5200 (Haswell) | MPEG-2, H.264, VC-1, JPEG | `i965` |
| Broadwell y más recientes | + VP8/HEVC/VP9/AV1 según la generación | `iHD` |

**Ninguna GPU hasta Haswell decodifica VP9 o AV1.** YouTube usa VP9/AV1 por defecto, así que en esos casos instala en el navegador una extensión que fuerce H.264 (ej.: *h264ify* o *enhanced-h264ify*). Sin ella el vídeo vuelve a ser decodificado por la CPU.

## 7.2 Instalar los paquetes

```bash
sudo xbps-install -S mesa-dri libva-intel-driver libva-utils intel-gpu-tools
```

Asegúrese de que el usuario esté en el grupo `video` (salga y vuelva a entrar en la sesión después):

```bash
sudo usermod -aG video $USER
```

## 7.3 Forzar el controlador VA-API correcto

Agregue en `~/.profile` (o en `/etc/environment`, sin el `export`):

```bash
export LIBVA_DRIVER_NAME=i965
```

## 7.4 Probar

```bash
vainfo
```

Resultado esperado (resumido):

```
vainfo: Driver version: Intel i965 driver for Intel(R) Sandybridge Mobile
      VAProfileH264Main               : VAEntrypointVLD
      VAProfileH264High               : VAEntrypointVLD
```

`VAEntrypointVLD` = decodificación por hardware disponible.

## 7.5 Firefox

En `about:config`:

```
media.ffmpeg.vaapi.enabled = true
```

En las versiones más recientes de Firefox, si la decodificación no se activa sola, también:

```
media.hardware-video-decoding.force-enabled = true
```

Si aplicaste el `user.js` de la sección 6, elimínalo. Reinicia Firefox y verifica en `about:support`, sección **Medios**, si H264 aparece con decodificación por hardware.

## 7.6 Confirmar que la GPU está trabajando

Abra un video en YouTube (con la extensión H.264 activada) y ejecute:

```bash
sudo intel_gpu_top
```

Si la línea **Video** se mantiene ocupada mientras el video se reproduce, la decodificación está en la GPU. El uso de CPU disminuye bastante.

---

# 8. Verificar soporte OpenGL

Ejecutar:

```bash
glxinfo | grep "OpenGL version"
```

Ejemplo:

```
OpenGL version string: 2.1 Mesa 25.3.3
```

Esto es lo máximo soportado por la GMA 3100.

---

# 9. Verificar aceleración activa

Ejecutar:

```bash
glxinfo | grep "direct rendering"
```

Resultado correcto:

```
direct rendering: Yes
```

---

# 10. Verificar controlador cargado

Ejecutar:

```bash
lsmod | grep i915
```

Resultado:

```
i915
```

---

# 11. Verificar el estado del Xorg

Ejecutar:

```bash
grep -i driver /var/log/Xorg.0.log
```

Resultado correcto:

```
modesetting
```

---

# 12. No utilice xf86-video-intel en hardware muy antiguo

Retire si está instalado:

```bash
sudo xbps-remove xf86-video-intel
```

El controlador modesetting es más estable.

---

# 13. Verificar DRI

Ejecutar:

```bash
glxinfo | grep DRI
```

Para GPUs antiguas, use:

```
DRI2
```

---

# 14. Verificar resolución

Ejecutar:

```bash
xrandr
```

---

# 15. Problemas comunes y solución

## reiniciar al entrar en XFCE

Desactivar compositor:

```bash
xfconf-query -c xfwm4 -p /general/use_compositing -s false
```

---

## Firefox se bloquea

Desactive la aceleración vía user.js (paso 6). **Solo GMA** — en Intel HD, prefiera mantener la aceleración.

---

## video de YouTube usando mucha CPU (Intel HD)

- Verifique si la extensión que fuerza H.264 está activa (la GPU no decodifica VP9/AV1).
- Ejecute `vainfo` y vea si aparece `VAProfileH264...: VAEntrypointVLD`.
- Confirme `LIBVA_DRIVER_NAME=i965` con `echo $LIBVA_DRIVER_NAME`.
- Confirme que el `user.js` de la sección 6 no está aplicado (él desactiva el VA-API).

---

## pantalla negra

Remueva xf86-video-intel:

```bash
sudo xbps-remove xf86-video-intel
```

---

# 16. Verificación completa del sistema gráfico

Ejecutar:

```bash
inxi -G
glxinfo | grep OpenGL
lsmod | grep i915
xfconf-query -c xfwm4 -p /general/use_compositing
vainfo          # somente Intel HD (Grupo B)
```

---

# 17. GPUs Intel y soporte OpenGL

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

# 18. Conclusión

**Grupo A – GMA (950, 3000, 3100, X3100, X4500):**

- controlador del núcleo: i915
- controlador Xorg: modos de configuración
- DRI2
- compositor desactivado
- aceleración de Firefox desactivada

**Grupo B – Intel HD (Ironlake, HD 2000/3000, HD 2500/4000, Haswell):**

- controlador del núcleo: i915
- controlador Xorg: modos de configuración
- controlador VA-API: i965 (`libva-intel-driver`) + `LIBVA_DRIVER_NAME=i965`
- VA-API activado en Firefox
- extensión que fuerza H.264 en YouTube

El sistema permanecerá estable y funcional.
