# 🧩 TUTORIAL VOID LINUX — INSTALACIÓN HÍBRIDA (UEFI + BIOS) CON EXT4, XFS, JFS O BTRFS (SUBVOLUMENES), LUKS, HIBERNACIÓN Y ZRAM
### VERSIÓN REVISADA Y VALIDADA — PARTICIONAMIENTO CORRECTO + ARRANQUE UNIVERSAL

Esta guía instala un Void Linux completamente **híbrido**, capaz de arrancar cualquier tipo de máquina, ya sea antigua, nueva o problemática:

- 💾 **UEFI moderno** (con entrada normal y respaldo)
- 🧮 **BIOS/Legacy** (compatibilidad total)
- 🧰 **GPT con BIOS Boot (EF02)**: soporte máximo para hardware antiguo
- 🚀 **Btrfs con subvolúmenes** (opcional), instantáneas listas para usar
- 🔐 **LUKS1 totalmente compatible con GRUB**
- 🌙 **Hibernación real mediante archivo de intercambio**
- 🧊 **ZRAM configurada para rendimiento**
- 🧱 **Soporte total para EXT4, XFS, JFS y BTRFS**
- 💡 **Initramfs/GRUB configurado automáticamente (LUKS + currículum)**

📌 **Sin compromisos, sin reinstalar GRUB, sin perder tiempo.**
📌 **Arranque garantizado incluso en una máquina con NVRAM borrada (reserva BOOTX64.EFI).**

---

# ▶️ 1. Bootar o Live ISO

Sugerencia: use la versión glibc para una compatibilidad superior:
- descargar la iso desde:
```
CHILE_REF_0_CHILI
```
- o busque la última versión en:
```
CHILE_REF_0_CHILI
```

1. Inicie sesión como root.
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

4. Pegue en la terminal (opcional): mensaje con colores, usuario@host: ruta y estado del último comando (✔/✘). Útil y hermoso.
```bash
exportar PS1='\[\e[1;32m\]\u\[\e[1;33m\]@\[\e[1;36m\]\h\[\e[1;31m\]:\w \
$([[ $? -eq 0 ]] && echo -e "\e[1;32m✔" || echo -e "\e[1;31m✘$?") \
\[\e[0m\]\$'
```

# ▶️ 2. Conéctate a Internet
- Para **Wi-Fi** *(si es por cable, omita este paso)*:
```bash
wpa_passphrase "SSID" "CONTRASEÑA" > wifi.conf
wpa_supplicant -B -i wlan0 -c wifi.conf
dhcpcd wlan0
```

1. Pruebe la conexión:
```bash
hacer ping -c3 8.8.8.8
ping -c3 repositorio-default.voidlinux.org
```

2. Instale los paquetes necesarios:
⚠️ **IMPORTANTE:**
```bash
xbps-install -Sy xbps partió jfsutils xfsprogs nano zstd xz bash-completion
```
---

# ▶️ 3. Identifica el disco
1. Enumere los discos disponibles y anote el nombre del dispositivo (por ejemplo: `/dev/sda`, `/dev/vda`, `/dev/nvme0n1`):
```bash
discof -l | grep -E '^(Disco|Disco) '
```

# ▶️ 4. Defina las variables utilizadas en el tutorial:
⚠️ **IMPORTANTE:**

1. Defina los dispositivos **ANTES** de usarlos:
> 1. **Asumiremos** para el tutorial `/dev/sda` (normal) o `/dev/nvme0n1` (nvme)
> 2. **Ajusta** según tu disco (elige solo **uno** u **otro** modelo)

Para discos **normales** (por ejemplo, /dev/sda)
```bash
exportar DISPOSITIVO=/dev/sda
exportar DEV_BIOS=${DEVICE}1
exportar DEV_EFI=${DEVICE}2
exportar DEV_ROOT=${DEVICE}3
exportar DEV_LUKS=/dev/mapper/cryptroot
```
Para discos **NVMe** (por ejemplo, /dev/nvme0n1), el sufijo de la partición cambia (`p`)
```bash
exportar DISPOSITIVO=/dev/nvme0n1
exportar DEV_BIOS=${DEVICE}p1
exportar DEV_EFI=${DEVICE}p2
exportar DEV_RAIZ=${DEVICE}p3
exportar DEV_LUKS=/dev/mapper/cryptroot
```

> 📌 **Nota:**
> DISPOSITIVO → disco completo
DEV_BIOS → Partición de arranque del BIOS (1–2 MiB, sin FS, no se monta)
DEV_EFI → Partición EFI (FAT32)
DEV_ROOT → partición raíz (normal o LUKS)
DEV_LUKS → Mapeo de LUKS (/dev/mapper/cryptroot)

- 👉 Aquí defines la anatomía del disco. Todo lo demás en la guía simplemente sigue estas variables.
- 🔎 ¿Por qué es esto necesario?
Porque declarar todo al principio hace que el siguiente proceso sea a prueba de errores tipográficos.

2. Defina **MAPA DE TECLAS** y **ZONA HORARIA** (cambie según sea necesario):
```bash
exportar MAPA DE CLAVES=br-abnt2
```
```bash
exportar TIMEZONE=América/Sao_Paulo
```

---

# ▶️ 5. Disco de partición
> - La partición del BIOS **DEBE** ser la primera. Esto aumenta la compatibilidad con placas base más antiguas, cargadores de arranque problemáticos y BIOS que esperan código de arranque en las primeras áreas del disco.
> - ESP puede venir más tarde sin ningún problema; a UEFI no le importa la posición.

### Orden ideal y correcto:

- 1️⃣ Arranque del BIOS (EF02)
- 2️⃣ ESP (Sistema EFI, FAT32)
- 3️⃣ Btrfs/Ext4/Xfs/Jfs (raíz)

### Partición usando parted (automático)
> Aquí el **DISPOSITIVO** ya está definido ahí arriba, por lo que no existe una variable “mágica”.
```
limpiafs -a "${DEVICE}"
separado --script "${DEVICE}" --\
etiqueta gpt \
mkpart primario 1MiB 2MiB nombre 1 BIOS establecido 1 bios_grub en \
mkpart primario fat32 2MiB 514MiB nombre 2 EFI set 2 esp on \
mkpart primario 514MiB 100% nombre 3 RAÍZ \
alinear-comprobar óptimo 1 \
alinear-comprobar óptimo 2 \
alinear-comprobar óptimo 1
separado --script "${DEVICE}" -- imprimir
```
> - Partición 1 → Arranque del BIOS (bios_grub, sin FS, no se monta)
> - Partición 2 → EFI (FAT32)
> - Partición 3 → ROOT (la formatearemos luego con EXT4/XFS/JFS/BTRFS, con o sin LUKS)
> - Utilicé mkpart primario 514MiB 100% sin especificar FS precisamente para evitar bloquear el FS. Eliges FS más tarde.
---

# ▶️ 6. Elija el modo de instalación (NORMAL o LUKS)
⚠️ **IMPORTANTE:**
> Elija SÓLO UNO de los dos bloques siguientes.
**NO** para ejecutar ambos pasos.

1. INSTALACIÓN NORMAL **(sin LUKS)**
```bash
exportar DISCO="${DEV_RAIZ}"
```
- Establece DISCO en el dispositivo real /dev/sda3

2. INSTALACIÓN **CON LUKS** (raíz cifrada)
```
# Cifre SÓLO la partición raíz en LUKS1 (compatible con GRUB), nunca todo el disco
# Cifre la partición confirmando con SÍ:
cryptsetup luksFormat --escriba luks1 "${DEV_RAIZ}"

# Abra la partición con su contraseña.
cryptsetup open "${DEV_RAIZ}" cryptroot

# De ahora en adelante, la raíz real es el dispositivo mapeado
exportar DISCO="${DEV_LUKS}"
```
- LUKS se encuentra encima de /dev/sda3, no de todo el disco
- El sistema se instalará en /dev/mapper/cryptroot

👉 De aquí en adelante TODO usa $DISK.

---

# ▶️ 7. Crea el sistema de archivos (FS) y monta la raíz
⚠️ **IMPORTANTE:**
> Elija SÓLO UNO de los dos bloques siguientes.

1. **EXT4**
```
mkfs.ext4 -F "${DISK}" -L RAÍZ
mount -v "${DISK}" /mnt
```
2. **XFS**
```
mkfs.xfs -f "${DISK}"
mount -v "${DISK}" /mnt
```
3. **JFS**
```
mkfs.jfs -f "${DISK}"
mount -v "${DISK}" /mnt
```
4. **BTRFS simple**
```
mkfs.btrfs -f "${DISK}" -L RAÍZ
mount -v "${DISK}" /mnt
```
5. **BTRFS con subvolúmenes**
```
mkfs.btrfs -f "${DISK}" -L RAÍZ

montaje ${DISK} /mnt
subvolumen btrfs crear /mnt/@
btrfs subvolume create /mnt/@home
subvolumen btrfs crear /mnt/@log
btrfs subvolume create /mnt/@cache
subvolumen btrfs crear /mnt/@snapshots
umount /mnt

mount -o defaults,noatime,ssd,compress=zstd:3,discard=async,space_cache=v2,commit=300,subvol=/@ ${DISK} /mnt
mkdir -p /mnt/{boot/efi,home,var/log,var/cache,.snapshots,swap}

mount -o defaults,noatime,ssd,compress=zstd:3,discard=async,space_cache=v2,commit=300,subvol=/@home ${DISK} /mnt/home
mount -o defaults,noatime,ssd,compress=zstd:3,discard=async,space_cache=v2,commit=300,subvol=/@cache ${DISK} /mnt/var/cache
mount -o defaults,noatime,ssd,compress=zstd:3,discard=async,space_cache=v2,commit=300,subvol=/@log ${DISK} /mnt/var/log
mount -o defaults,noatime,ssd,compress=zstd:3,discard=async,space_cache=v2,commit=300,subvol=/@snapshots ${DISK} /mnt/.snapshots
```
---

# ▶️ 8. Preparar y montar el ESP (EFI)
```
mkfs.fat -F32 -I "${DEV_EFI}"
mkdir -p /mnt/arranque/efi
montar -v "${DEV_EFI}" /mnt/boot/efi
```
>💡 La partición del BIOS (${DEV_BIOS}) no tiene sistema de archivos, no formatea, no se monta.
---

# ▶️    9. Instalar o Void Linux no chroot

1. Copie las claves del repositorio (claves XBPS) que se utilizarán en el chroot (/mnt)
```
mkdir -p /A{tc, vapor/xbps/xbps)
cp -rpaf /var/db/xbps/keys/*.plist /mnt/var/db/xbps/keys/
cp -fpa /etc/resolv.conf /mnt/etc/resolv.conf
```

2. Instale el sistema base en el disco recién montado:
```
instalación-xbps -Sy -R https://repo-default.voidlinux.org/current \
-r /mnt \
sistema base btrfs-progs cryptsetup grub grub-x86_64-efi dracut linux \
encabezados-linux firmware-linux red-firmware-linux glibc-locales \
xtools dhcpcd openssh vim nano grc zstd xz finalización de bash vpm vsv \
socklog-void wget net-tools tmate ncurses jfsutils xfsprogs duf tree eza chrony
```
---

# ▶️ 10. Gerar fstab no /mnt (chroot)
```bash
# Generar fstab en /mnt/etc/fstab
xgenfstab -U /mnt > /mnt/etc/fstab
```

```bash
# comprobar si se generó correctamente
cat /mnt/etc/fstab
```

# ▶️ 11. Accede al sistema instalado usando chroot

1. No emplear croit:
```
xchroot /mnt /bin/bash
```
---

# ▶️ 12. Configuración inicial (en chroot)
```
# configurar el nombre de host: define el nombre de host
eco vacío > /etc/nombrehost

# configurar zona horaria: define la zona horaria
ln -sfv /usr/share/zoneinfo/"${TIMEZONE}" /etc/localtime

# configurar configuraciones regionales
-i -e 's/^#\(en_.Utf-8 UTF-8\)/)/' \
-E 's/^#\pt_br.br.utf-8 UTF-8\)/' \' \
/etc/default/libc-locales

# generar configuraciones regionales
xbps-reconfigure -f glibc-locales

# Corregir posible error en el enlace simbólico /var/service (importante):
rm -f /var/servicio
ln -sf /etc/runit/runsvdir/default /var/service

# Activar algunos servicios
ln -sf /etc/sv/dbus /var/servicio/
ln -sf /etc/sv/dhcpcd /var/servicio/
ln -sf /etc/sv/sshd /var/servicio/
ln -sf /etc/sv/nanoklogd /var/servicio/
ln -sf /etc/sv/socklog-unix /var/servicio/
ln -sf /etc/sv/chronyd /var/servicio/

# Configurar sudo - grupo de ruedas (opcional, pero recomendado)
gato << 'EOF' > /etc/sudoers.d/g_wheel
%wheel TODOS=(TODOS:TODOS) NOPASSWD: TODOS
EOF
#Permisos requeridos
chmod 440 /etc/sudoers.d/g_wheel
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
---

# ▶️ 13. Configurar UUID
⚠️ **IMPORTANTE:**
- Obtener los UUID de las particiones:
```
UUID_LUKS=$(blkid -s UUID -o valor "${DEV_RAIZ}")
UUID_ROOT=$(blkid -s UUID -o valor "${DISK}")
UUID_EFI=$(blkid -s UUID -o valor "${DEV_EFI}")
```
---

# ▶️ 14. Cree un archivo de intercambio con soporte de hibernación (opcional)

### Notas importantes
```
- El archivo de intercambio en Btrfs siempre aparece como **prealloc**, es normal.
- No es necesario que tenga el tamaño completo de RAM.
- El 60% es suficiente para la hibernación en la mayoría de los casos.
- Para cargas pesadas → utilizar 70% u 80%.
```

1. Calcule automáticamente el tamaño óptimo del archivo de intercambio
```
# Recomendación moderna para hibernación: 60% de la RAM total
SWAP_GB=$(LC_ALL=C awk '/MemTotal/ {print int($2 * 0.60/1024/1024)}' /proc/meminfo)
echo "Archivo de intercambio recomendado: ${SWAP_GB}G"
```
- o configurar manualmente el tamaño deseado:
```
SWAP_GB=4
echo "Archivo de intercambio definido por el usuario: ${SWAP_GB}G"
```
2. Cree un directorio para el archivo de intercambio.
```
mkdir -p /intercambio
swapoff -a 2>/dev/null
rm -f /swap/archivo de intercambio
```
3. Desactivar COW (requerido en Btrfs)
```
chattr +C /intercambiar
```
4. Crea el archivo de intercambio con el tamaño previamente definido.
```
fallocate -l ${SWAP_GB}G /intercambio/archivo de intercambio
chmod 600 /swap/archivo de intercambio
```
5. Formatee el archivo de intercambio y active el intercambio.
```
mkswap /swap/archivo de intercambio
swapon /swap/swapfile
```
6. Obtener compensación:
```
# Instalar el paquete para filefrag
xbps-install -Sy e2fsprogs

# Obtener el desplazamiento
offset=$(filefrag -v /swap/swapfile | awk '/^ *0:/{imprimir $4}')
```
---

# ▶️ 15. Configurar GRUB
⚠️ **IMPORTANTE:**
> Este bloque es inteligente:
- Detecta automáticamente si estás usando LUKS
- Detecta si creaste un archivo de intercambio con hibernación.
- Ajusta /etc/default/grub sin duplicar nada
- Crea las líneas necesarias sólo si faltan
- No cambies nada si no es necesario

Utilice exactamente el siguiente bloque:
```
HAS_RESUME=falso
HAS_LUKS=falso

[[ -n "${offset}" ]] && HAS_RESUME=true
[[ "${DISK}" = "${DEV_LUKS}" ]] && HAS_LUKS=true

# Eliminar línea antigua por seguridad
sed -i '/^[[:space:]]*GRUB_CMDLINE_LINUX_DEFAULT=/d' /etc/default/grub

#GRUB_CMDLINE_LINUX

# Valor base
BASE="nivel de registro=4"

# Agregar resumen
si $HAS_RESUME; entonces
BASE="$BASE resume=UUID=${UUID_ROOT} resume_offset=${offset}"
ser

# Agregar LUKS
si $HAS_LUKS; entonces
grep -q '^GRUB_ENABLE_CRYPTODISK=y' /etc/default/grub || echo 'GRUB_ENABLE_CRYPTODISK=y' >> /etc/default/grub
grep -q '^GRUB_PRELOAD_MODULES=' /etc/default/grub || echo 'GRUB_PRELOAD_MODULES="luks cryptodisk gcry_rijndael"' >> /etc/default/grub
BASE="$BASE rd.luks.uuid=${UUID_LUKS} rd.luks.name=${UUID_LUKS}=cryptroot root=/dev/mapper/cryptroot"
ser

# Recrea la línea final correctamente
echo "GRUB_CMDLINE_LINUX_DEFAULT=\"${BASE}\"" >> /etc/default/grub
```
---

# ▶️ 16. Recrea el initrd
⚠️ **IMPORTANTE:**
```
mods=(/usr/lib/modules/*)
KVER=$(nombre base "${mods[0]}")
eco ${KVER}
dracut --force --kver ${KVER}
```
---

# ▶️ 17. Cree un archivo de claves para evitar solicitar la contraseña dos veces al arrancar (solo LUKS)
> Si el sistema NO utiliza LUKS, omita este paso.
```
if [ "${DISK}" = "${DEV_LUKS}" ]; then
echo "LUKS detectado: creando archivo de claves para desbloqueo automático..."

# Crear archivo de claves seguro
dd if=/dev/urandom of=/boot/volume.key bs=64 count=1
chmod 000 /boot/volumen.clave

# Agregue un archivo de claves a LUKS (le pedirá su contraseña actual)
cryptsetup luksAddKey "${DEV_RAIZ}" /boot/volume.key

# Configurar /etc/crypttab
gato << EOF >> /etc/crypttab
cryptroot ${DEV_RAIZ} /boot/volume.key luks
EOF

# Incluir archivo de claves y crypttab en initramfs
mkdir -p /etc/dracut.conf.d
gato << EOF >> /etc/dracut.conf.d/10-crypt.conf
install_items+=" /boot/volume.key /etc/crypttab "
EOF

# Regenerar initramfs con soporte para archivos de claves
xbps-reconfigurar -fa
demás
echo "Sistema sin LUKS: omitiendo la creación del archivo de claves".
ser
```

# ▶️ 18. Instale GRUB en **BIOS** y **UEFI** (híbrido real)
1. Instale GRUB para BIOS (heredado)
```
instalación de grub --target=i386-pc ${DEVICE}
```
2. Instale GRUB para UEFI
```
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=void
```
3. Cree un respaldo UEFI (arranque universal). Este archivo garantiza el arranque incluso cuando se borra la NVRAM.
```
mkdir -p /arranque/efi/EFI/ARRANQUE
cp -f /boot/efi/EFI/void/grubx64.efi /boot/efi/EFI/BOOT/BOOTX64.EFI
```
4. Genere el archivo GRUB final
```
grub-mkconfig -o /boot/grub/grub.cfg
```

---

# ▶️ 19. Configuración de usuario personalizada:

1. Configuración del entorno:

```
# Personalizar /etc/xbps.d/00-repository-main.conf
mkdir -pv /etc/xbps.d
gato << 'EOF' >> /etc/xbps.d/00-repository-main.conf
repositorio=https://repo-fastly.voidlinux.org/current
#repositorio=https://repo-fastly.voidlinux.org/current/nonfree
#repositorio=https://repo-fastly.voidlinux.org/current/multilib
#repositorio=https://repo-fastly.voidlinux.org/current/multilib/nonfree

repositorio=https://void.chililinux.com/voidlinux/current
#repositorio=https://void.chililinux.com/voidlinux/current/extras
#repositorio=https://void.chililinux.com/voidlinux/current/nonfree
#repositorio=https://void.chililinux.com/voidlinux/current/multilib
#repositorio=https://void.chililinux.com/voidlinux/current/multilib/nonfree
EOF

# Personaliza /etc/rc.conf. Establece la zona horaria, la distribución del teclado y la fuente predeterminadas de la consola. Cambie según sea necesario.
gato << EOF >> /etc/rc.conf
TIMEZONE="${TIMEZONE}"
KEYMAP="${KEYMAP}"
FUENTE=Lat2-Terminus16
EOF

# Personaliza el .bashrc de root
wget --quiet --no-check-certificado \
-O /etc//skel/.bashrc \
"https://raw.githubusercontent.com/voidlinux-br/void-install/refs/heads/main/.bashrc"
raíz de chown: raíz /etc/skel/.bashrc
chmod 644 /etc/skel/.bashrc

cat << 'EOF' > /etc/skel/.bash_profile
# ~/.bash_profile — carga .bashrc en Void

# Si .bashrc existe, cargar
si [ -f ~/.bashrc ]; entonces
fuente ~/.bashrc
ser
EOF

# copiar a root y usuario
for d in /root "/home/${NEWUSER}"; do
cp -f /etc/skel/.bash_profile "$d/"
cp -f /etc/skel/.bashrc "$d/"
hecho

chown "${NEWUSER}:${NEWUSER}" "/home/${NEWUSER}/.bash_profile" "/home/${NEWUSER}/.bashrc"
chmod 644 "/home/${NEWUSER}/.bash_profile" "/home/${NEWUSER}/.bashrc"

# descargar svlogtail personalizado
wget --quiet --no-check-certificado \
-O /usr/bin/svlogtail\
"https://raw.githubusercontent.com/voidlinux-br/void-install/refs/heads/main/svlogtail"
chmod +x /usr/bin/svlogtail
```

2. Configure ssh (opcional, pero recomendado):
```
mkdir -pv /etc/ssh/sshd_config.d/
gato << 'EOF' > /etc/ssh/sshd_config.d/10-custom.conf
PermitirTTY sí
ImprimirMotd si
ImprimirÚltimoRegistro sí
Banner /etc/issue.net

PermitRootLogin sí
KbdInteractiveAuthentication sí
X11Reenvío sí
Autenticación Pubkey sí
PubkeyAcceptedKeyTypes=+ssh-rsa
Archivo de claves autorizadas .ssh/claves_autorizadas
ContraseñaAutenticación sí
DesafíoRespuestaAutenticación sí
Usar PAM si

Subsistema sftp interno-sftp
EOF
```
---

# ▶️ 20. Habilite ZRAM (opcional)
Void Linux utiliza el servicio zramen para habilitar ZRAM, creando un bloque de memoria comprimida que reduce el uso de intercambio de SSD y mejora el rendimiento bajo carga.
1. Instalar zramen
```
instalación xbps -Sy zramen
```
2. Configure ZRAM (configuración recomendada):
```
gato << 'EOF' > /etc/zramen.conf
fracción_zram=0.5
dispositivos_zram=1
zram_algorithm=zstd
EOF
```
3. Activar el servicio en runit.
```
en -s
```
> ZRAM se activará automáticamente en cada arranque

---

# ▶️ 21. Finalizar la instalación
1. Sair do chroot:
```
salida
```
2. Desmonte todas las particiones montadas en /mnt (subvolúmenes y /boot/efi):
```
desmontar -R /mnt
```
3. Deshabilite cualquier archivo de intercambio o partición de intercambio que se haya activado dentro del chroot:
```
intercambio -a
```
4. Reinicie la máquina física o VM para probar el arranque real:
```
reiniciar
```
> No olvide quitar el medio de instalación y arrancar desde el disco recién instalado.
¡Disfrutar!

---

# 🎉 SISTEMA COMPLETO, HÍBRIDO Y PREPARADO PARA EL FUTURO
- Arranque BIOS + UEFI
- UEFI alternativa
- Btrfs con instantáneas (preparado para Snapper/Timeshift)
- Hibernación real con archivo de intercambio
- Zram para el rendimiento

Este SSD arranca **cualquier máquina del planeta**.

# DESCARGO DE RESPONSABILIDAD

```
Este tutorial es gratuito: puedes usarlo, copiarlo, modificarlo y redistribuirlo como desees.
El contenido está disponible bajo la **Licencia MIT** y puede incluir fragmentos o comandos derivados de software de código abierto sujeto a sus propias licencias.

No se ofrecen garantías: aquí todo se entrega "tal cual".
Úselo bajo su propio riesgo. Ni el autor, ni los contribuyentes, ni Void Linux son responsables de pérdidas, daños, fallas del sistema o cualquier consecuencia del uso de este material.

Si lo deseas, puedes obtener el código fuente, revisar, adaptar y generar tu propia versión de este tutorial.
```

