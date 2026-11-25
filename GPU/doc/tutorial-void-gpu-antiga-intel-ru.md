# Полное руководство: графические процессоры Intel в Linux (Void Linux, XFCE и старое оборудование)

Это руководство объясняет, как идентифицировать, настраивать и стабилизировать графические процессоры Intel в Linux, особенно старое оборудование, такое как GMA 3100, GMA X3100, HD 2000/3000 и т.д.

> **Внимание:** старые видеокарты Intel делятся на две группы, и настройка меняется:
>
> - **Группа A – GMA (950, 3000, 3100, X3100, X4500):** следуют шагам стабильности (DRI2, композитор и ускорение Firefox отключены). Не обладают полезным аппаратным декодированием видео.
> - **Группа B – Intel HD (1-е поколение Core/Ironlake, HD 2000/3000/Sandy Bridge, HD 2500/4000/Ivy Bridge, Haswell):** как правило, не требуют корректировки стабильности и **могут и должны использовать аппаратное ускорение видео (VA-API)** с драйвером `i965` (см. раздел 7).
>
> Используйте раздел 1, чтобы узнать, к какой группе относится ваша видеокарта.

---

# 1. Определите вашу видеокарту Intel

Выполнить:

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

# 2. Драйверы Intel на Linux

Все графические процессоры Intel используют:

```
kernel driver: i915
```

Вам НЕ нужно устанавливать собственный драйвер. В Intel **все драйверы свободные**.

## 2.1 i915, i965 и iHD: не путайте

Это разные уровни, и часто думают, что они являются взаимозаменяемыми:

| Название | Что это | Откуда берется |
|------|---------|----------|
| `i915` | Драйвер **ядра** (видео/modesetting) | Уже включен в ядро, ничего устанавливать не нужно |
| `i965` | Драйвер **VA-API** (видеодекодирование) для GPU до ~Coffee Lake | Пакет `libva-intel-driver` |
| `iHD` | Драйвер **VA-API** для Broadwell (5-го поколения) и новее | Пакет `intel-media-driver` |
| `crocus` / `iris` | Драйвер **OpenGL** Mesa (crocus: Gen4 a Haswell; iris: Broadwell+) | Пакет `mesa-dri` |

То есть: кто имеет HD 2000/3000, использует `i915` (ядро) **и** `i965` (VA-API) одновременно.

## 2.2 Драйверы Xorg

Существуют два возможных драйвера Xorg:

- modesetting → рекомендуется
- xf86-video-intel → устарело в большинстве случаев

Проверьте, какой используется:

```bash
inxi -G
```

Если показать:

```
driver: modesetting
```

Это правильно.

---

# 3. Для старых графических процессоров Intel (GMA 950, 3000, 3100, X3100, X4500)

Эти графические процессоры имеют ограничения:

- OpenGL 2.1 максимум
- Нестабильный DRI3
- композитор вызывает зависание
- современное ускорение не работает правильно

---

# 4. Рекомендуемая стабильная конфигурация

Создайте:

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

# 5. Отключить композитор XFCE (ОБЯЗАТЕЛЬНО на старых GPU)

Выполнить:

```bash
xfconf-query -c xfwm4 -p /general/use_compositing -s false
```

Проверьте:

```bash
xfconf-query -c xfwm4 -p /general/use_compositing
```

Ожидаемый результат:

```
false
```

---

# 6. Настройка Firefox для старых графических процессоров Intel (только группа A – GMA)

> **Только для GMA 950/3000/3100/X3100/X4500.** Если ваша видеокарта Intel HD (HD 2000/3000 или новее), **не выполняйте этот шаг**: он отключает ускорение видео. Перейдите прямо к разделу 7.

Создайте:

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

Это предотвращает зависания.

---

# 7. Аппаратное ускорение видео (VA-API) для Intel HD (Группа B)

С VA-API GPU декодирует видео, а CPU остаётся свободным. Это имеет большое значение для ноутбуков с Core i3/i5 2-го/3-го поколения при просмотре YouTube.

## 7.1 Что каждая GPU декодирует в аппаратуре

| GPU | Кодеки с ускорением | Драйвер VA-API |
|-----|----------------------|---------------|
| GMA 950/3000/3100/X3100 | нет | — |
| GMA X4500 | MPEG-2 | `i965` |
| HD Graphics (Ironlake, 1-е поколение Core) | MPEG-2, H.264, VC-1 | `i965` |
| HD 2000/3000 (Sandy Bridge) | MPEG-2, H.264, VC-1 | `i965` |
| HD 2500/4000 (Ivy Bridge) | MPEG-2, H.264, VC-1, JPEG | `i965` |
| HD 4200–5200 (Haswell) | MPEG-2, H.264, VC-1, JPEG | `i965` |
| Broadwell и новее | + VP8/HEVC/VP9/AV1 в зависимости от поколения | `iHD` |

**Ни одна GPU до Haswell не декодирует VP9 или AV1.** YouTube использует VP9/AV1 по умолчанию, поэтому в таких случаях установите в браузере расширение, которое принудительно включает H.264 (например, *h264ify* или *enhanced-h264ify*). Без него видео снова будет декодироваться процессором.

## 7.2 Установить пакеты

```bash
sudo xbps-install -S mesa-dri libva-intel-driver libva-utils intel-gpu-tools
```

Убедитесь, что пользователь находится в группе `video` (выйдите и войдите в сеанс после):

```bash
sudo usermod -aG video $USER
```

## 7.3 Принудительно использовать правильный драйвер VA-API

Добавьте в `~/.profile` (или в `/etc/environment`, без `export`):

```bash
export LIBVA_DRIVER_NAME=i965
```

## 7.4 Тестировать

```bash
vainfo
```

Ожидаемый результат (кратко):

```
vainfo: Driver version: Intel i965 driver for Intel(R) Sandybridge Mobile
      VAProfileH264Main               : VAEntrypointVLD
      VAProfileH264High               : VAEntrypointVLD
```

`VAEntrypointVLD` = декодирование с помощью аппаратного обеспечения доступно.

## 7.5 Фаерфокс

В `about:config`:

```
media.ffmpeg.vaapi.enabled = true
```

В новых версиях Firefox, если декодирование не включается автоматически, также:

```
media.hardware-video-decoding.force-enabled = true
```

Если вы применили `user.js` из раздела 6, удалите его. Перезапустите Firefox и проверьте в `about:support`, раздел **Медиа**, появляется ли H264 с аппаратным декодированием.

## 7.6 Подтвердить, что GPU работает

Откройте видео на YouTube (с активным расширением H.264) и запустите:

```bash
sudo intel_gpu_top
```

Если линия **Видео** занята во время воспроизведения видео, декодирование выполняется на GPU. Использование CPU значительно снижается.

---

# 8. Проверить поддержку OpenGL

Выполнить:

```bash
glxinfo | grep "OpenGL version"
```

Пример:

```
OpenGL version string: 2.1 Mesa 25.3.3
```

Это максимум, поддерживаемый GMA 3100.

---

# 9. Проверить активное ускорение

Выполнить:

```bash
glxinfo | grep "direct rendering"
```

Правильный результат:

```
direct rendering: Yes
```

---

# 10. Проверить загруженный драйвер

Выполнить:

```bash
lsmod | grep i915
```

Результат:

```
i915
```

---

# 11. Проверить статус Xorg

Выполнить:

```bash
grep -i driver /var/log/Xorg.0.log
```

Правильный результат:

```
modesetting
```

---

# 12. Не используйте xf86-video-intel на очень старом оборудовании

Удалите, если установлено:

```bash
sudo xbps-remove xf86-video-intel
```

Драйвер modesetting более стабильный.

---

# 13. Проверить DRI

Выполнить:

```bash
glxinfo | grep DRI
```

Для старых видеокарт используйте:

```
DRI2
```

---

# 14. Проверить разрешение

Выполнить:

```bash
xrandr
```

---

# 15. Общие проблемы и решение

## перезагрузка при входе в XFCE

Отключить композитор:

```bash
xfconf-query -c xfwm4 -p /general/use_compositing -s false
```

---

## Firefox зависает

Отключите ускорение через user.js (шаг 6). **Только GMA** — для Intel HD лучше оставить ускорение включённым.

---

## видео на YouTube использует много ЦП (Intel HD)

- Проверьте, активировано ли расширение, принуждающее использование H.264 (GPU не декодирует VP9/AV1).
- Запустите `vainfo` и посмотрите, появится ли `VAProfileH264...: VAEntrypointVLD`.
- Подтвердите `LIBVA_DRIVER_NAME=i965` с `echo $LIBVA_DRIVER_NAME`.
- Подтвердите, что `user.js` из раздела 6 не применен (он отключает VA-API).

---

## чёрный экран

Удалите xf86-video-intel:

```bash
sudo xbps-remove xf86-video-intel
```

---

# 16. Полная проверка графической системы

Выполнить:

```bash
inxi -G
glxinfo | grep OpenGL
lsmod | grep i915
xfconf-query -c xfwm4 -p /general/use_compositing
vainfo          # somente Intel HD (Grupo B)
```

---

# 17. ГПУ Intel и поддержка OpenGL

| ГПУ | OpenGL |
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

# 18. Заключение

**Группа A – GMA (950, 3000, 3100, X3100, X4500):**

- драйвер ядра: i915
- драйвер Xorg: modesetting
- DRI2
- композитор отключен
- аппаратное ускорение Firefox отключено

**Группа B – Intel HD (Ironlake, HD 2000/3000, HD 2500/4000, Haswell):**

- драйвер ядра: i915
- драйвер Xorg: modesetting
- драйвер VA-API: i965 (`libva-intel-driver`) + `LIBVA_DRIVER_NAME=i965`
- VA-API включен в Firefox
- расширение, которое принуждает H.264 на YouTube

Система станет стабильной и функциональной.
