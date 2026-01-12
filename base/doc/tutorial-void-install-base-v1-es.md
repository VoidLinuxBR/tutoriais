# 🔥 Tutorial de instalación de Base Void Linux

# Antes de empezar

Este tutorial describe una **instalación manual de Void Linux**, utilizando partición directa del disco, `chroot` y configuración explícita del sistema.
**No es un instalador automático**.

## ⚠️ Leer atentamente

- Esta guía **asume familiaridad con Linux**, terminal y conceptos básicos del sistema (discos, particiones, arranque, servicios).
- Varios comandos **borran datos permanentemente** (`parted`, `mkfs`, `umount -R`).
- Un error al definir el disco (`/dev/sdX`, `/dev/nvmeX`) puede resultar en una **pérdida total de datos**.
- Lea **el tutorial completo antes de ejecutar cualquier comando**.

## 🖥️ Entorno recomendado

- **VM (VirtualBox, QEMU, KVM, etc.)** para pruebas y aprendizaje.
- Hardware dedicado **sin datos importantes**.
- Entorno de laboratorio o instalación consciente.

❌ **No recomendado** para uso directo en producción sin adaptaciones.

## 🔐 Acerca de la seguridad

Durante el proceso de instalación, algunas configuraciones **priorizan la practicidad**, no la seguridad:
- El inicio de sesión del usuario `root` a través de SSH se puede habilitar temporalmente.
- La autenticación de contraseña puede estar activa.
- Es posible que se permita la compatibilidad heredada (por ejemplo, `ssh-rsa`).

👉 **Estas configuraciones deben revisarse después de la instalación**, especialmente en sistemas expuestos a la red.

## 🧠 Importante saber

- Ejecutar los comandos **uno por uno**, comprobando la salida.
- Ajuste los nombres de los discos, las interfaces de red y los usuarios según su sistema.
- **No copiar y pegar a ciegas**.
- En caso de duda, **deténgase** y revise el paso actual.

## 🛠️ En caso de error

Si algo sale mal:
- No reiniciar a ciegas.
- Volver a montar los tabiques.
- Vuelva a iniciar sesión en el sistema con `chroot`.
- Verifique GRUB, EFI y `initramfs`.

Cometer errores es parte de ello. Comprender el error es lo que separa al usuario del operador.

---

> Esta guía está dirigida a usuarios que prefieren **control total** sobre la instalación, siguiendo el enfoque clásico de Unix:
> **comprender → configurar → validar → continuar**.

## Iniciar instalación
Comience con la ISO de Void Linux (x86_64 glibc o musl).

1. Inicia sesión como root
```bash
iniciar sesión: raíz
contraseña: voidlinux
```
2. Cambie el shell de *sh* a *bash*.
*dash/sh* **NO implementa** varias características que utilizan muchos scripts.
```bash
intento
```
3. Cambie la distribución del teclado a **ABNT2**, asegurando la asignación correcta de acentos y símbolos:
```bash
llaves de carga br-abnt2
```

4. Conéctate a Internet
- Para **Wi-Fi** *(si es por cable, omita este paso)*:
```bash
wpa_passphrase "WIFI_NETWORK_NAME" "NETWORK_PASSWORD" > wifi.conf
wpa_supplicant -B -i wlan0 -c wifi.conf
dhcpcd wlan0
```
> 📌 **Nota:** `wlan0` puede variar (`wlp2s0`, `wlp0s3`, etc.).
> Utilice el siguiente comando para identificar la interfaz correcta:
>
> ```golpecito
> ip -br a
> ```

5. Pruebe la conexión:
```bash
hacer ping -c3 8.8.8.8
ping -c3 repositorio-default.voidlinux.org
```

## Habilite el inicio de sesión de usuario **root** a través de SSH (opcional).
Este paso solo se aplica cuando el sistema se ejecuta en una VM; en caso de arranque local (sin VM), la instalación puede continuar normalmente a través del terminal local.
- Esto es necesario para acceder a la **VM desde el host** y continuar la instalación de forma remota; después de eso, los comandos se pueden pegar/ejecutar directamente en la terminal a través de SSH.

1. Configurar ssh
```bash
echo 'PermitRootLogin sí' >> /etc/ssh/sshd_config
```
2. Reinicie el servicio ssh
```bash
sv reiniciar sshd
```

3. Ver la IP de la interfaz de red
```bash
ruta ip -4 obtener 1.1.1.1 | awk '{imprimir $7}'
```
>Anote la IP de la interfaz de red y úsela para conectarse a la VM a través de SSH.

4. Acceda a la VM a través de SSH desde el host.
```bash
sudo ssh <ip-da-vm>
```
> Contraseña predeterminada: `voidlinux`

## Configure un mensaje de color en la terminal (opcional)
Mostrará usuario, host, ruta actual y el estado del último comando:
```bash
exportar PROMPT_COMMAND='RET=$?'
exportar PS1='\[\e[1;33m\]\u\[\e[0m\]@\[\e[1;35m\]\h\[\e[0m\]:\[\e[0;37m\]\w\[\e[0m\] \[\e[1;32m\]$( [ $RET -eq 0 ] && printf ✔ || printf "\e[1;31m✘$RET" )\[\e[0m\] \$ '
```
> 📌 Este aviso solo es válido para la sesión actual; para hacerlo permanente agréguelo a `.bashrc`.

## Instalar los paquetes necesarios
⚠️ **IMPORTANTE:**
```bash
xbps-install -Sy xbps partió nano vim zstd xz finalización de bash
```

## Particionar el disco
1. Identificar el disco
```bash
discof -l | grep -E '^(Disco|Disco) '
```
> Asumiremos para el tutorial `/dev/sda`

2. Ajuste las variables a continuación según el disco que se utilizará (**IMPORTANTE**):
```bash
# Discos SATA/SCSI (sdX)
exportar DISPOSITIVO=/dev/sda
exportar DEV_EFI=${DEVICE}2
exportar DEV_ROOT=${DEVICE}3
```

> 📌 **Nota:**
> Para discos **NVMe**, el sufijo de la partición cambia (`p`):
> ```golpecito
> exportar DISPOSITIVO=/dev/nvme0n1
> exportar DEV_EFI=${DEVICE}p2
> exportar DEV_RAIZ=${DEVICE}p3
> ```

3. Particione el disco usando **parted** (modo automático).
Este esquema crea:
- Partición BIOS (bios_grub)
- Partición EFI (ESP)
- Partición raíz (ROOT)
```bash
limpiafs -a "${DEVICE}"
separado --script "${DEVICE}" --\
etiqueta gpt \
mkpart primario 1MiB 2MiB nombre 1 BIOS establecido 1 bios_grub en \
mkpart primario fat32 2MiB 514MiB nombre 2 EFI set 2 esp on \
mkpart primario 514MiB 100% nombre 3 RAÍZ \
alinear-comprobar óptimo 1 \
alinear-comprobar óptimo 2 \
alinear-comprobar óptimo 3
separado --script "${DEVICE}" -- imprimir
```

## Formatear particiones
```bash
# Formatear la partición raíz (ext4)
mkfs.ext4 -F ${DEV_RAIZ}

# Formatee la partición EFI (FAT32)
mkfs.fat -F32 -I ${DEV_EFI}
```

## Montar los volúmenes en `/mnt`
```bash
#Montar la partición raíz
montaje ${DEV_RAIZ} /mnt

# Crea los puntos de montaje necesarios
/t /mnt/{hame,boot/efi,var/log,var/cache, proced, proc, Proc,)

# Montar la partición EFI
montar ${DEV_EFI} /mnt/boot/efi
```

## Instalar el sistema base
Instala el sistema base Void Linux en el entorno montado `/mnt`, incluido el kernel, el firmware, el gestor de arranque, las redes y las herramientas esenciales.
```bash
instalación-xbps -Sy -R https://repo-default.voidlinux.org/current \
-r /mnt \
sistema base e2fsprogs grub-x86_64-efi dracut linux \
encabezados-linux firmware-linux red-firmware-linux glibc-locales \
xtools dhcpcd openssh vim nano grc zstd xz finalización de bash vpm vsv \
socklog-void wget net-tools tmate ncurses chrony
```

> 📌 **Nota:**
> - `grub-x86_64-efi` → gestor de arranque UEFI
> - `linux` → núcleo
> - `linux-firmware-network` → controladores de red
> - `xtools` → requerido para usar `xgenfstab` sin falta

## Crear `fstab`
Genera automáticamente el archivo de montaje permanente del sistema.
```bash
xgenfstab -U /mnt > /mnt/etc/fstab
```

##Entrar y sistema (chroot)
Acceda al sistema instalado en `/mnt` para continuar con la configuración.
```bash
xchroot /mnt /bin/bash
```

## Generar INITRAMFS
Configuración de Dracut para entornos de virtualización (VM-safe)
```bash
gato > /etc/dracut.conf.d/99-vm-safe.conf << 'EOF'
# /etc/dracut.conf.d/99-vm-safe.conf
solo host=no
comprimir="gzip"
add_drivers+=" virtio virtio_pci virtio_blk virtio_net virtio_scsi "
EOF
```

Detecta automáticamente la versión del kernel instalada y genera el `initramfs` correspondiente usando **dracut**.
```bash
mods=(/usr/lib/modules/*)
KVER=$(nombre base "${mods[0]}")
eco ${KVER}
dracut --force --kver ${KVER}
```

## Configurar GRUB

> 📌 Ambos métodos (BIOS y UEFI) se instalan expresamente.
> Esto permite que el mismo disco arranque en sistemas **Legacy BIOS** y **UEFI**, lo que aumenta la portabilidad entre máquinas.

1. Cree el directorio de soporte de GRUB:
```bash
mkdir -p /arranque/grub
```

2. Instale GRUB para **BIOS (heredado)**:
```bash
instalación de grub --target=i386-pc ${DEVICE}
```

3. Instale GRUB para **UEFI**:
```bash
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=void
```

4. Cree un respaldo UEFI (arranque universal).
Este archivo garantiza el arranque incluso si se borra la NVRAM:
```bash
mkdir -p /arranque/efi/EFI/ARRANQUE
cp -f /boot/efi/EFI/void/grubx64.efi /boot/efi/EFI/BOOT/BOOTX64.EFI
```

5. Genere el archivo de configuración final de GRUB:
```bash
grub-mkconfig -o /boot/grub/grub.cfg
```

## Crear y configurar usuarios

⚠️ **IMPORTANTE:** define el nombre de usuario real a continuación.
```bash
exportar NUEVOUSUARIO=tu_usuario_aquí
```

Cree el usuario con el directorio de inicio, los grupos básicos y el shell Bash:
```bash
usuarioadd -m -G audio,vídeo,rueda,tty -s /bin/bash ${NEWUSER}
```

Establece tu contraseña de usuario (***IMPORTANTE***)
```bash
contraseña ${NEWUSER}
```

Establecer contraseña de usuario root (***IMPORTANTE***)
```bash
raíz de contraseña
```

Cambie el shell predeterminado del usuario root a Bash
```bash
chsh -s /bin/bash raíz
```

## Configuración básica
```bash
# Establecer nombre de host
eco vacío > /etc/nombrehost

# Establecer hora local
ln -sf /usr/share/zoneinfo/America/Sao_Paulo /etc/localtime

# Setar Locales
sed -i 's/#en_US.UTF-8/en_US.UTF-8/' /etc/default/libc-locales
sed -i 's/#pt_BR.UTF-8/pt_BR.UTF-8/' /etc/default/libc-locales

# Generar configuraciones regionales:
xbps-reconfigure -f glibc-locales

# Corregir posible error en el enlace simbólico /var/service (importante):
rm -f /var/servicio
ln -sf /etc/runit/runsvdir/default /var/service

# Activar algunos servicios:
ln -sf /etc/sv/dbus /var/servicio/
ln -sf /etc/sv/dhcpcd /var/servicio/
ln -sf /etc/sv/sshd /var/servicio/
ln -sf /etc/sv/nanoklogd /var/servicio/
ln -sf /etc/sv/socklog-unix /var/servicio/
ln -sf /etc/sv/chronyd /var/servicio/

# descargar svlogtail personalizado (opcional, pero recomendado):
wget --quiet --no-check-certificate -O /usr/bin/svlogtail \
"https://raw.githubusercontent.com/voidlinux-br/void-install/refs/heads/main/svlogtail" &&\
chmod +x /usr/bin/svlogtail

# Crea un archivo resolv.conf
printf 'servidor de nombres 1.1.1.1\nservidor de nombres 8.8.8.8\n' > /etc/resolv.conf

#Configurar sudo - grupo de ruedas (opcional, pero recomendado)
gato << 'EOF' > /etc/sudoers.d/g_wheel
%wheel TODOS=(TODOS:TODOS) NOPASSWD: TODOS
EOF

#Permisos requeridos
chmod 440 /etc/sudoers.d/g_wheel
```

## Personalizar `/etc/xbps.d/00-repository-main.conf`
*(Opcional, pero recomendado)*

Crea el directorio de configuración **XBPS** (si aún no existe) y define una lista de repositorios oficiales y alternativos.
Los repositorios **repo-fastly** tienden a tener una mejor latencia.

```bash
mkdir -pv /etc/xbps.d

gato << 'EOF' > /etc/xbps.d/00-repository-main.conf
# Repositorio oficial (Fastly – mejor latencia)
repositorio=https://repo-fastly.voidlinux.org/current
#repositorio=https://repo-fastly.voidlinux.org/current/nonfree
#repositorio=https://repo-fastly.voidlinux.org/current/multilib
#repositorio=https://repo-fastly.voidlinux.org/current/multilib/nonfree

# Repositorio alternativo (Chili Linux)
repositorio=https://void.chililinux.com/voidlinux/current
#repositorio=https://void.chililinux.com/voidlinux/current/extras
#repositorio=https://void.chililinux.com/voidlinux/current/nonfree
#repositorio=https://void.chililinux.com/voidlinux/current/multilib
#repositorio=https://void.chililinux.com/voidlinux/current/multilib/nonfree
EOF
```

## Personalizar `/etc/rc.conf`
Establece la zona horaria, la distribución del teclado y la fuente predeterminadas de la consola.
Cambie según sea necesario.
```bash
gato << 'EOF' > /etc/rc.conf
ZONA HORARIA=América/Sao_Paulo
MAPA DE TECLAS=br-abnt2
FUENTE=Lat2-Terminus16
EOF
```

Módulos Virtio (máquina virtual).
```bash
gato > /etc/modules-load.d/virtio.conf << 'EOF'
virtio
virtio_pci
virtio_net
virtio_blk
virtio_scsi
EOF
```

## Personalizar usuario `.bashrc`
Crea un `.bash_profile` predeterminado y garantiza que `.bashrc` se cargue automáticamente al iniciar sesión.
> ⚠️ Asegúrate de que el usuario haya sido creado en el paso anterior.

```bash
# Descargar .bashrc predeterminado a /etc/skel
wget --quiet --no-check-certificado \
-O /etc/skel/.bashrc \
"https://raw.githubusercontent.com/voidlinux-br/void-install/refs/heads/main/.bashrc"

raíz de chown: raíz /etc/skel/.bashrc
chmod 644 /etc/skel/.bashrc

# Crear .bash_profile predeterminado
cat << 'EOF' > /etc/skel/.bash_profile
# ~/.bash_profile — carga .bashrc en Void

# Si .bashrc existe, cargar
si [ -f ~/.bashrc ]; entonces
fuente ~/.bashrc
ser
EOF

# Copiar a root y usuario
for d in /root "/home/${NEWUSER}"; do
cp -f /etc/skel/.bash_profile "$d/"
cp -f /etc/skel/.bashrc "$d/"
hecho

# Ajustar los permisos de usuario
chown "${NEWUSER}:${NEWUSER}" \
"/home/${NEWUSER}/.bash_profile" \
"/home/${NEWUSER}/.bashrc"

chmod 644 \
"/home/${NEWUSER}/.bash_profile" \
"/home/${NEWUSER}/.bashrc"
```

## Configurar SSH
*(Opcional, pero recomendado)*

Crea un archivo de configuración complementario para **sshd**, dejando el archivo principal intacto.
```bash
mkdir -pv /etc/ssh/sshd_config.d

gato << 'EOF' > /etc/ssh/sshd_config.d/10-custom.conf
# Configuraciones generales
PermitirTTY sí
ImprimirMotd si
ImprimirÚltimoRegistro sí
Banner /etc/issue.net

# Autenticación
PermitRootLogin sí
ContraseñaAutenticación sí
KbdInteractiveAuthentication sí
DesafíoRespuestaAutenticación sí
Autenticación Pubkey sí
PubkeyAcceptedKeyTypes=+ssh-rsa
Archivo de claves autorizadas .ssh/claves_autorizadas
Usar PAM si

# Características
X11Reenvío sí
Subsistema sftp interno-sftp
EOF
```
> ⚠️ Se recomienda revisar y reforzar estas configuraciones SSH después del primer arranque, especialmente en sistemas expuestos a Internet.

## Finalizar la instalación
Sair hacer chroot
```bash
salida
```
Desmonte todas las particiones montadas en `/mnt` (incluidos los subvolúmenes y `/boot/efi`):
```bash
desmontar -R /mnt
```
Reinicie la máquina física o VM para probar el arranque real:
```bash
reiniciar
```
> 📌 **Nota: No olvides quitar el medio de instalación.

# 🎉 ¡Disfruta!
**Void Linux** ya está instalado y listo para usar.

# DESCARGO DE RESPONSABILIDAD

> Este tutorial es gratuito: puedes usarlo, copiarlo, modificarlo y redistribuirlo como desees.
> El contenido está disponible bajo la **Licencia MIT** y puede incluir extractos o comandos derivados de software de código abierto, sujeto a sus propias licencias.
>
> No se ofrecen garantías: aquí todo se entrega **“tal cual”**.
> Úselo bajo su propia responsabilidad. Ni el autor, ni los contribuyentes, ni Void Linux son responsables de pérdidas, daños, fallas del sistema o cualquier consecuencia del uso de este material.
>
> Eres libre de revisar, adaptar y generar tu propia versión de este tutorial.

