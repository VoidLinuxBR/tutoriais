# 🐧 Void Linux + GNOME — Tutorial

> ⚠️ **IMPORTANTE — LEER ANTES DE COMENZAR**
>
> Este tutorial **NO debe ejecutarse como `root`**, excepto cuando **se indique explícitamente**.
>
> Todos los comandos fueron diseñados para ser ejecutados por **un usuario común**, usando `sudo` cuando sea necesario.
>
> Ejecute todo el tutorial iniciando sesión como `root`:
> - rompe la lógica de permisos
> - invalida pasos como la configuración `sudo`
> - puede generar errores silenciosos o comportamiento inesperado
>
> 👉 **Recomendación**
> Si acaba de instalar el sistema y ha iniciado sesión como `root`:
>
> 1. Crear un usuario común
> 2. Inicia sesión con este usuario
> 3. Sigue el tutorial normalmente.
>
> Regla clásica para sistemas Unix/Linux:
>
> **`root` es una excepción. El usuario común es la regla.**

---

## 0. Configure sudo - grupo de ruedas - para evitar solicitar la contraseña de root
```
sudo tee -a /etc/sudoers.d/g_wheel >/dev/null << EOF
%wheel TODOS=(TODOS:TODOS) NOPASSWD: TODOS
EOF

#Permisos requeridos
sudo chmod 440 /etc/sudoers.d/g_wheel
```

## 1. Actualizar el sistema
```
sudo xbps-install -Syu
```

## 2. Instale GNOME completo (metapaquete)
```
sudo xbps-install -y gnome \
tema-icono-gnomo \
papeles \
network-manager-applet \
administrador de extensiones \
nautilo \
extensión-de-papeles-nautilus \
extensión-consola-nautilus-gnome \
extensión-terminal-nautilus-gnome \
terminal-gnome \
tema de arco \
Firefox\
firefox-i18n-pt-BR\
xarchiver\
utilidad-disco-gnome \
separado \
gvfs\
p7zip\
descomprimir \
eog \
noto-fuentes-emoji \
arriba
```

## 3. Instale GDM (administrador de pantalla oficial)
```
sudo xbps-install -y gdm
```

## 4. Controladores de pantalla

### Intel
```
sudo xbps-install -y mesa-dri linux-firmware-intel
```

### nuevo AMD (amdgpu)
```
sudo xbps-install -y mesa-dri xf86-video-amdgpu
```

### AMD antiguo
```
sudo xbps-install -y mesa-dri xf86-video-ati
```

### Nvidia (controlador abierto)
```
sudo xbps-install -y mesa-nouveau-dri
```

## 5. Instale PipeWire (sonido vacío moderno)
```
sudo xbps-install -y \
alambre de tubería \
fontanero \
alsa-plugins-pulseaudio \
alsa-pipewire \
libjack-pipewire \
pulseaudio-utils \
alsa-utils \
pavucontrol
```

## 6. Integrar ALSA → PipeWire
```
sudo mkdir -p /etc/alsa/conf.d
sudo ln -sf /usr/share/alsa/alsa.conf.d/50-pipewire.conf /etc/alsa/conf.d
sudo ln -sf /usr/share/alsa/alsa.conf.d/99-pipewire-default.conf /etc/alsa/conf.d
```

## 7. Habilite el servidor pipewire-pulse (compatible con PulseAudio)
```
sudo mkdir -p /etc/pipewire/pipewire.conf.d
sudo ln -sf /usr/share/examples/pipewire/20-pipewire-pulse.conf /etc/pipewire/pipewire.conf.d/
```

## 8. Habilite el inicio automático de PipeWire en la sesión
```
mkdir -p ~/.config/autostart
ln -sf /usr/share/applications/pipewire.desktop ~/.config/autostart/
ln -sf /usr/share/applications/pipewire-pulse.desktop ~/.config/autostart/
ln -sf /usr/share/applications/wireplumber.desktop ~/.config/autostart/
```

## 9. (Opcional) Cree .xinitrc para startx
```
cat <<EOF > ~/.xinitrc
#!/bin/sh
setxkbmap -diseño br -variante abnt2 &
sesión ejecutiva de gnome
EOF
```

## 10. configurar zona horaria: define la zona horaria
```
sudo ln -sfv /usr/share/zoneinfo/America/Sao_Paulo /etc/localtime
```

## 11. configurar configuraciones regionales
```
Sudo -i -i -e 's/s/#\(en_.us.utf-8 utf-8\)/' -e 's/^#\pt_br.br.br.
```

## 12. Personaliza /etc/rc.conf. Establece la zona horaria, la distribución del teclado y la fuente predeterminadas de la consola. Cambie según sea necesario.
```
sudo tee -a /etc/rc.conf >/dev/null << EOF
TIMEZONE="América/Sao_Paulo"
MAPA DE CLAVE="br-abnt2"
FUENTE=Lat2-Terminus16
EOF
```

## 13. Personaliza /etc/locale.conf. Establece el idioma. Cambie según sea necesario.
```
sudo tee /etc/locale.conf >/dev/null << EOF
LANG=pt_BR.UTF-8
IDIOMA=pt_BR.UTF-8
LC_COLLATE=pt_BR.UTF-8
EOF
```

## 14. Reconfigurar
```
sudo xbps-reconfigure -fa
```

## 15. Activar servicios obligatorios (runit)
```
sudo ln -s /etc/sv/dbus /var/servicio/
sudo ln -s /etc/sv/elogind /var/servicio/
sudo ln -s /etc/sv/polkitd /var/servicio/
sudo ln -s /etc/sv/NetworkManager /var/servicio/
sudo ln -s /etc/sv/gdm /var/servicio/
```

## Finalización
- Usando GDM → el sistema se inicia directamente en GNOME.
- Sin GDM → use `startx` (si existe `.xinitrc`).
