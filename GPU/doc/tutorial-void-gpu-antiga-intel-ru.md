# Полное руководство: графические процессоры Intel в Linux (Void Linux, XFCE и старое оборудование)

В этом руководстве объясняется, как идентифицировать, настраивать и стабилизировать графические процессоры Intel в Linux, особенно на старом оборудовании, таком как GMA 3100, GMA X3100, HD 2000/3000 и т. д.

---

# 1. Определите свой графический процессор Intel

Выполнять:

```bash
lspci -k | grep -A3 VGA
```

или:

```bash
inxi -G
```

Пример вывода:

```
Device: Intel 82G33/G31 Express Integrated Graphics
Driver: i915
Display: X.Org modesetting
OpenGL: 2.1 Mesa i915
```

---

# 2. Драйвера Intel без Linux

Все графические процессоры Intel используют:

```
kernel driver: i915
```

Вам НЕ нужно устанавливать проприетарный драйвер.

Есть два возможных драйвера Xorg:

- настройка режима → рекомендуется
- xf86-video-intel → в большинстве случаев устарел

Проверьте, какой из них используется:

```bash
inxi -G
```

Если он показывает:

```
driver: modesetting
```

Это правильно.

---

# 3. Для старых графических процессоров Intel (GMA 950, 3000, 3100, X3100, X4500)

Эти графические процессоры имеют ограничения:

- OpenGL 2.1 максимум
- DRI3 нестабильный
- композитор вызывает сбой
- современное ускорение работает некорректно

---

# 4. Рекомендуемая стабильная конфигурация.

Создавать:

```bash
sudo mkdir -p /etc/X11/xorg.conf.d
sudo nano /etc/X11/xorg.conf.d/20-intel.conf
```

Содержание:

```conf
Section "Device"
    Identifier "Intel Graphics"
    Driver "modesetting"
    Option "DRI" "2"
EndSection
```

---

# 5. Отключите композитор XFCE (ТРЕБУЕТСЯ на старых графических процессорах).

Выполнять:

```bash
xfconf-query -c xfwm4 -p /general/use_compositing -s false
```

Проверять:

```bash
xfconf-query -c xfwm4 -p /general/use_compositing
```

Ожидаемый результат:

```
false
```

---

# 6. Настройка Firefox для старых графических процессоров Intel

Создавать:

```bash
nano ~/.mozilla/firefox/*.default-release/user.js
```

Содержание:

```js
user_pref("layers.acceleration.disabled", true);
user_pref("gfx.webrender.force-disabled", true);
user_pref("gfx.xrender.enabled", true);
user_pref("media.ffmpeg.vaapi.enabled", false);
```

Это предотвращает сбои.

---

# 7. Проверьте поддержку OpenGL.

Выполнять:

```bash
glxinfo | grep "OpenGL version"
```

Пример:

```
OpenGL version string: 2.1 Mesa 25.3.3
```

Это максимум, поддерживаемый GMA 3100.

---

# 8. Проверьте активное ускорение

Выполнять:

```bash
glxinfo | grep "direct rendering"
```

Правильный результат:

```
direct rendering: Yes
```

---

# 9. Проверьте загруженный драйвер

Выполнять:

```bash
lsmod | grep i915
```

Результат:

```
i915
```

---

# 10. Проверьте статус Xorg

Выполнять:

```bash
grep -i driver /var/log/Xorg.0.log
```

Правильный результат:

```
modesetting
```

---

# 11. Не используйте xf86-video-intel на очень старом оборудовании.

Удалить, если установлено:

```bash
sudo xbps-remove xf86-video-intel
```

Драйвер настройки режима более стабилен.

---

# 12. Проверьте ДРИ

Выполнять:

```bash
glxinfo | grep DRI
```

Для старых графических процессоров используйте:

```
DRI2
```

---

# 13. Проверьте разрешение

Выполнять:

```bash
xrandr
```

---

# 14. Распространенные проблемы и их решение

## перезагрузка при входе в XFCE

Отключить композитор:

```bash
xfconf-query -c xfwm4 -p /general/use_compositing -s false
```

---

## Firefox вылетает

Отключите ускорение через user.js (шаг 6)

---

## черный экран

Удалить xf86-video-intel:

```bash
sudo xbps-remove xf86-video-intel
```

---

# 15. Полная проверка графической системы

Выполнять:

```bash
inxi -G
glxinfo | grep OpenGL
lsmod | grep i915
xfconf-query -c xfwm4 -p /general/use_compositing
```

---

# 16. Графические процессоры Intel и поддержка OpenGL.

| графический процессор | OpenGL |
|----|--------|
| ГМА 950 | 2.1 |
| ГМА 3100 | 2.1 |
| Х3100 | 2.1 |
| X4500 | 2.1 |
| HD 2000 | 3.3 |
| HD 3000 | 3.3 |
| HD 4000 | 4.0 |
| UHD 620 | 4,6 |

---

# 17. Заключение

Для старых графических процессоров Intel используйте:

- ядро драйвера: i915
- драйвер Xorg: настройка режима
- ДРИ2
- композитор отключен
- Ускорение Firefox отключено

Система будет стабильной и функциональной.
