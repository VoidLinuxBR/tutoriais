# 🐧 Void Linux + GNOME — Учебное пособие

> ⚠️ **ВАЖНО — ПРОЧТИТЕ ПЕРЕД НАЧАЛОМ**
>
> Это руководство **НЕ следует запускать от имени пользователя root**, за исключением случаев, когда **явно указано**.
>
> Все команды были разработаны для выполнения **обычным пользователем** с использованием `sudo` при необходимости.
>
> Запустите все руководство, войдя в систему как root:
> - нарушает логику разрешений
> - делает недействительными такие шаги, как конфигурация `sudo`
> - может генерировать тихие ошибки или неожиданное поведение
>
> 👉 **Рекомендация**
> Если вы только что установили систему и вошли в систему как root:
>
> 1. Создать обычного пользователя
> 2. Войти под этим пользователем
> 3. Следуйте инструкциям в обычном режиме.
>
> Классическое правило для систем Unix/Linux:
>
> **`root` — исключение. Обычный пользователь – это правило.**

---

## 0. Настройте sudo — группу колес — чтобы не запрашивать пароль root
```
sudo tee -a /etc/sudoers.d/g_wheel >/dev/null << EOF
%wheel ALL=(ALL:ALL) NOPASSWD: ALL
ЭОФ

#Необходимые разрешения
sudo chmod 440 /etc/sudoers.d/g_wheel
```

## 1. Обновить систему
```
sudo xbps-install -Syu
```

## 2. Установите полную версию GNOME (мета-пакет)
```
sudo xbps-install -y gnome \
gnome-icon-theme \
документы \
апплет сетевого менеджера \
менеджер расширений \
наутилус \
расширение nautilus-papers \
расширение nautilus-gnome-console \
расширение nautilus-gnome-terminal \
gnome-терминал \
дуговая тема \
Firefox \
Firefox-i18n-pt-BR \
xarchiver \
gnome-диск-утилита \
gparted \
гвфс\
p7zip \
разархивировать \
эог \
noto-fonts-emoji \
хтоп
```

## 3. Установите GDM (официальный менеджер дисплея)
```
sudo xbps-install -y gdm
```

## 4. Драйверы дисплея

### Интел
```
sudo xbps-install -y mesa-dri Linux-прошивка-Intel
```

### новый AMD (amdgpu)
```
sudo xbps-install -y mesa-dri xf86-video-amdgpu
```

### старый AMD
```
sudo xbps-install -y mesa-dri xf86-video-ati
```

### Nvidia (открытый драйвер)
```
sudo xbps-install -y mesa-nouveau-dri
```

## 5. Установите PipeWire (Modern Void Sound)
```
sudo xbps-install -y \
трубопровод\
сантехник \
alsa-plugins-pulseaudio \
alsa-pipewire \
libjack-pipewire \
PulseAudio-Utils \
alsa-utils \
павуконтроль
```

## 6. Интеграция ALSA → PipeWire
```
sudo mkdir -p /etc/alsa/conf.d
sudo ln -sf /usr/share/alsa/alsa.conf.d/50-pipewire.conf /etc/alsa/conf.d
sudo ln -sf /usr/share/alsa/alsa.conf.d/99-pipewire-default.conf /etc/alsa/conf.d
```

## 7. Включить сервер Pipewire-Pulse (совместим с PulseAudio)
```
sudo mkdir -p /etc/pipewire/pipewire.conf.d
sudo ln -sf /usr/share/examples/pipewire/20-pipewire-pulse.conf /etc/pipewire/pipewire.conf.d/
```

## 8. Включить автозапуск PipeWire во время сеанса.
```
mkdir -p ~/.config/автозапуск
ln -sf /usr/share/applications/pipewire.desktop ~/.config/autostart/
ln -sf /usr/share/applications/pipewire-pulse.desktop ~/.config/autostart/
ln -sf /usr/share/applications/wireplumber.desktop ~/.config/autostart/
```

## 9. (Необязательно) Создайте .xinitrc для startx
```
кот <<EOF > ~/.xinitrc
#!/бин/ш
setxkbmap -layout br -вариант abnt2 &
exec gnome-сессия
ЭОФ
```

##10. настроить часовой пояс - определяет часовой пояс
```
sudo ln -sfv /usr/share/zoneinfo/America/Sao_Paulo /etc/localtime
```

## 11. настроить локали
```
Sudo -i -i -e 's/s/#\(en_.us.utf-8 utf-8\)/' -e 's/^#\pt_br.br.br.
```

## 12. Настройте /etc/rc.conf. Устанавливает часовой пояс консоли по умолчанию, раскладку клавиатуры и шрифт. Меняйте по мере необходимости.
```
sudo tee -a /etc/rc.conf >/dev/null << EOF
TIMEZONE="Америка/Сан-Паулу"
KEYMAP="br-abnt2"
FONT=Lat2-Terminus16
ЭОФ
```

## 13. Настройте /etc/locale.conf. Устанавливает язык. Меняйте по мере необходимости.
```
sudo tee /etc/locale.conf >/dev/null << EOF
LANG=pt_BR.UTF-8
ЯЗЫК=pt_BR.UTF-8
LC_COLLATE=pt_BR.UTF-8
ЭОФ
```

## 14. Перенастроить
```
sudo xbps-reconfigure -fa
```

##15. Активировать обязательные службы (runit)
```
sudo ln -s /etc/sv/dbus /var/service/
sudo ln -s /etc/sv/elogind /var/service/
sudo ln -s /etc/sv/polkitd /var/service/
sudo ln -s /etc/sv/NetworkManager /var/service/
sudo ln -s /etc/sv/gdm /var/service/
```

## Завершение
- Usando GDM → o sistema inicia direto no GNOME.
- Без GDM → используйте startx (если существует .xinitrc).
