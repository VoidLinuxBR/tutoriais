# Tutorial completo: GPU Intel en Linux (Void Linux, XFCE y hardware antiguo)

Esta guía explica cómo identificar, configurar y estabilizar GPU Intel en Linux, especialmente hardware antiguo como GMA 3100, GMA X3100, HD 2000/3000, etc.

---

# 1. Identifique su GPU Intel

Ejecutar:

```bash
lspci -k | grep -A3 VGA
```

o:

```bash
inxi -G
```

Salida de ejemplo:

```
Device: Intel 82G33/G31 Express Integrated Graphics
Driver: i915
Display: X.Org modesetting
OpenGL: 2.1 Mesa i915
```

---

# 2. Controladores Intel sin Linux

Todas las GPU Intel utilizan:

```
kernel driver: i915
```

NO es necesario instalar un controlador propietario.

Hay dos posibles controladores Xorg:

- configuración de modo → recomendado
- xf86-video-intel → obsoleto en la mayoría de los casos

Compruebe cuál está en uso:

```bash
inxi -G
```

Si muestra:

```
driver: modesetting
```

Es correcto.

---

# 3. Para GPU Intel más antiguas (GMA 950, 3000, 3100, X3100, X4500)

Estas GPU tienen limitaciones:

- OpenGL 2.1 máximo
- DRI3 inestable
- el compositor provoca un accidente
- la aceleración moderna no funciona correctamente

---

# 4. Configuración estable recomendada

Crear:

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

# 5. Deshabilite el compositor XFCE (REQUERIDO en GPU antiguas)

Ejecutar:

```bash
xfconf-query -c xfwm4 -p /general/use_compositing -s false
```

Controlar:

```bash
xfconf-query -c xfwm4 -p /general/use_compositing
```

Resultado esperado:

```
false
```

---

# 6. Configurar Firefox para GPU Intel más antiguas

Crear:

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

Esto evita accidentes.

---

# 7. Verifique la compatibilidad con OpenGL

Ejecutar:

```bash
glxinfo | grep "OpenGL version"
```

Ejemplo:

```
OpenGL version string: 2.1 Mesa 25.3.3
```

Este es el máximo admitido por el GMA 3100.

---

# 8. Verifique la aceleración activa

Ejecutar:

```bash
glxinfo | grep "direct rendering"
```

Resultado correcto:

```
direct rendering: Yes
```

---

# 9. Verifique el controlador cargado

Ejecutar:

```bash
lsmod | grep i915
```

Resultado:

```
i915
```

---

# 10. Verifique el estado de Xorg

Ejecutar:

```bash
grep -i driver /var/log/Xorg.0.log
```

Resultado correcto:

```
modesetting
```

---

# 11. No utilice xf86-video-intel en hardware muy antiguo

Quitar si está instalado:

```bash
sudo xbps-remove xf86-video-intel
```

El controlador de configuración de modo es más estable.

---

# 12. Verifique el DRI

Ejecutar:

```bash
glxinfo | grep DRI
```

Para GPU más antiguas, utilice:

```
DRI2
```

---

# 13. Verificar resolución

Ejecutar:

```bash
xrandr
```

---

# 14. Problemas comunes y solución

## reiniciar al entrar a XFCE

Deshabilitar compositor:

```bash
xfconf-query -c xfwm4 -p /general/use_compositing -s false
```

---

## Firefox falla

Deshabilite la aceleración a través de user.js (paso 6)

---

## pantalla negra

Eliminar xf86-video-intel:

```bash
sudo xbps-remove xf86-video-intel
```

---

# 15. Verificación completa del sistema de gráficos.

Ejecutar:

```bash
inxi -G
glxinfo | grep OpenGL
lsmod | grep i915
xfconf-query -c xfwm4 -p /general/use_compositing
```

---

# 16. GPU Intel y compatibilidad con OpenGL

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

# 17. Conclusión

Para GPU Intel más antiguas, utilice:

- núcleo del controlador: i915
- controlador Xorg: configuración de modo
- DRI2
- compositor discapacitado
- Aceleración de Firefox deshabilitada

El sistema será estable y funcional.
