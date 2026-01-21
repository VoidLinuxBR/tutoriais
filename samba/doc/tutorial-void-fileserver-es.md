
# Servidor de archivos ejecutando Samba4 en Void Linux Server; D

## 🎯 Objetivo: implementar un servidor de archivos en Void Linux (glibc) compilando Samba4 desde el código fuente, la integración de AD, las ACL, los servicios y toda la pila necesaria para que un servidor de archivos atienda a los clientes de la red.

## 🔧 Laboratorio de networking con QEMU/Virtmanager y Proxmox. Ajuste el tutorial para que coincida con su propio entorno.

---

## 📡 Diseño de red local

- Dominio: EDUCATUX.EDU

- Nombre de host: servidor de archivos

- Cortafuegos 192.168.70.254 (DNS/GW)

- Dirección IP: 192.168.70.251

## Instalar vacío Linux

## Cambiar el shell predeterminado en Void

```bash
chsh -s /bin/bash
```

## 🧩 Instale paquetes de dependencia para compilar Samba4 en Void

```bash
xbps-install -S \
 net-tools rsync acl attr attr-devel autoconf automake libtool \
 binutils bison gcc make ccache chrpath curl \
 docbook-xml docbook-xsl flex gdb git htop \
 mit-krb5 mit-krb5-client mit-krb5-devel \
 libarchive-devel avahi avahi-libs libblkid-devel \
 libbsd-devel libcap-devel cups-devel dbus-devel glib-devel \
 gnutls-devel gpgme-devel icu-devel jansson-devel \
 lmdb lmdb-devel libldap-devel ncurses-devel pam-devel perl \
 perl-Text-ParseWords perl-JSON perl-Parse-Yapp \
 libpcap-devel popt-devel readline-devel \
 libtasn1 libtasn1-devel libunwind-devel python3 python3-devel \
 python3-dnspython python3-cryptography \
 python3-matplotlib python3-pexpect python3-pyasn1 \
 tree libuuid-devel wget xfsprogs-devel zlib-devel \
 bind ldns pkg-config vim
```

## 🖥️ Establecer nombre de host

```bash
echo "fileserver" > /etc/hostname
```

## 🏠 /etc/hosts

```bash
vim /etc/hosts
```

## Contenido:

```bash
127.0.0.1      localhost
127.0.1.1      fileserver.educatux.edu fileserver
192.168.70.251 fileserver.educatux.edu fileserver
```

## 🌐 Configurar IP estática

## 👉 Usaremos el método Void estándar, /etc/dhcpcd.conf

```bash
vim /etc/dhcpcd.conf
```

## Agregue IP, puerta de enlace (enrutador) y DNS (AD):

```bash
interface eth0
static ip_address=192.168.70.251/24
static routers=192.168.70.254
static domain_name_servers=192.168.70.253
```

## Reinicie la interfaz de red:

```bash
sv restart dhcpcd
```

## 🧭 Establecer dirección DNS: apunte al PDC

```bash
vim /etc/resolv.conf
```

## Contenido:

```bash
domain educatux.edu
search educatux.edu
nameserver 192.168.70.253
```

## Bloquear resolución.conf

```bash
chattr +i /etc/resolv.conf
```

## 🔍 Validar la dirección de la interfaz de red

```bash
ip -c addr
ip -br link
```

## 📥 Descargue y extraiga el código fuente de Samba4

```bash
wget https://download.samba.org/pub/samba/samba-4.23.4.tar.gz
```

```bash
tar -xvzf samba-4.23.4.tar.gz
```

## Compilar e instalar desde la fuente

```bash
cd samba-4.23.4
```

```bash
./configure --prefix=/opt/samba
```

```bash
make -j$(nproc) && make install
```

## Notas:

- Void no interfiere porque Samba está compilado en /opt/samba.

- make -j acelera enormemente la compilación; aún así, ve a tomar un café.

- Después de la instalación, Samba4 compilado no tiene ningún servicio runit.

- Crearemos los servicios manualmente.

## 📁 Agregue Samba4 a la RUTA del sistema y recargue el entorno

```bash
echo 'export PATH=/opt/samba/bin:/opt/samba/sbin:$PATH' >> /etc/profile
```

```bash
source /etc/profile
```

## Pruebe la inserción de Samba4 PATH en el sistema operativo

```bash
samba-tool -V
```

## Producción:

```bash
4.23.4
```

## ⚠️ Advertencia: ¡NO utilice el comando de aprovisionamiento en el servidor de archivos!

## 📝 Crea el archivo smb.conf

```bash
vim /opt/samba/etc/smb.conf
```

```bash
[global]
   workgroup = EDUCATUX
   security = ads
   realm = EDUCATUX.EDU
   netbios name = fileserver
   encrypt passwords = yes
   # point to the services, the active interfaces
   interfaces = eth0
   bind interfaces only = yes

   log file = /opt/samba/var/log.%m
   max log size = 50

   winbind use default domain = yes
   winbind enum users = yes
   winbind enum groups = yes
   winbind refresh tickets = yes

   idmap config * : backend = tdb
   idmap config * : range = 3000-7999
   idmap config EDUCATUX : backend = rid
   idmap config EDUCATUX : range = 10000-999999

   template shell = /bin/bash
   template homedir = /home/%U

[Public]
   path = /srv/samba/public
   browsable = yes
   writable = yes
``` 

## Crear el archivo de registro

```bash
mkdir /opt/samba/var
```

## 📂 Crea la ruta para compartir

```bash
sudo mkdir -p /srv/samba/arquivos
sudo chown -R root:"Domain Admins" /srv/samba/arquivos
sudo chmod -R 0770 /srv/samba/arquivos
```

## Recargar la configuración de Samba4

```bash
smbcontrol all reload-config
```

## 🕒 Servidor NTP / Chrony

## El controlador de dominio debe ser el servidor de hora local, porque con una diferencia de 5 minutos, Kerberos ya no autenticará a los clientes.

## Instalar Chrony

```bash
xbps-install -Syu chrony
```

## Editar configuración y permitir red interna

```bash
vim /etc/chrony.conf
```

## Configure el Control de dominio en los servidores horarios:

```bash
# Comment the external line
#pool pool.ntp.org iburst

# PDC Time Servers
server 192.168.70.253 iburst
```

## Habilitar chronyd en runit

```bash
ln -sf /etc/sv/chronyd/ /var/service/
```

## Reiniciar el servicio:

```bash
sv restart chronyd
```

## Validar servidores:

```bash
chronyc sources -v
```

## 🔐 Cree el archivo Kerberos: apunte al PDC

```bash
vim /etc/krb5.conf
```

## que contiene

```bash
[libdefaults]
    default_realm = EDUCATUX.EDU
    dns_lookup_realm = true
    dns_lookup_kdc = true
    rdns = false
    forwardable = true
    proxiable = true

[realms]
    EDUCATUX.EDU = {
        kdc = 192.168.70.253
        admin_server = 192.168.70.253
        default_domain = educatux.edu
    }

[domain_realm]
    .educatux.edu = EDUCATUX.EDU
    educatux.edu = EDUCATUX.EDU
```

## prueba de Kerberos

```bash
kinit Administrator
```

```bash
klist
```

## Resultado obtenido

```bash
Ticket cache: FILE:/tmp/krb5cc_0
Default principal: administrator@EDUCATUX.EDU

Valid starting       Expires              Service principal
11/12/2025 09:43:40  11/12/2025 19:43:40  krbtgt/EDUCATUX.EDU@EDUCATUX.EDU
	renew until 12/12/2025 09:43:36
```

## 🔗 Vincular bibliotecas Winbind al sistema

## Validar la ruta de libdir:

```bash
/opt/samba/sbin/smbd -b | grep LIBDIR
```

## Esperado:

```bash
LIBDIR: /opt/samba/lib
```

## Cree enlaces de biblioteca (prefiera escribir manualmente):

```bash
ln -sf /opt/samba/lib/libnss_winbind.so.2 /usr/lib/
```

```bash
ln -sf /usr/lib/libnss_winbind.so.2 /usr/lib/libnss_winbind.so
```

## Recargar caché de la biblioteca:

```bash
ldconfig
```

## Actualice nsswitch para el intercambio de tickets de Kerberos (agregue winbind):

```bash
vim /etc/nsswitch.conf
```

```bash
passwd: files winbind
group:  files winbind
```

## Deja el resto intacto

## Únete al dominio

```bash
net ads join -U Administrator
```

## Resultado obtenido

```bash
Password for [EDUCATUX\Administrator]:
Using short domain name -- EDUCATUX
Joined 'VOIDFILES' to dns domain 'educatux.edu'
```

## 📦 Cree los servicios RUNIT para smbd, winbindd y, opcionalmente, nmbd

### Void Linux usa runit como sistema de inicio y su registrador nativo es svlogd, incluido en el paquete base de runit. No se requieren paquetes adicionales.

## SMBD: servicio y registro

## Crear directorios de servicios y registros

```bash
mkdir -p /etc/sv/smbd/log
mkdir -p /var/log/smbd
```

## Crear /etc/sv/smbd/run

```bash
vim /etc/sv/smbd/run
```

## Contenido

```bash
#!/bin/sh
exec 2>&1
exec /opt/samba/sbin/smbd --foreground --no-process-group
```

## Permiso

```bash
chmod +x /etc/sv/smbd/run
```

## Crear /etc/sv/smbd/log/run

```bash
vim /etc/sv/smbd/log/run
```

## Contenido

```bash
#!/bin/sh
exec svlogd -tt /var/log/smbd
```

## Permiso

```bash
chmod +x /etc/sv/smbd/log/run
```

## Depurar (opcional)

```bash
/opt/samba/sbin/smbd -i
```

## Resultado obtenido

```bash
smbd version 4.23.4 started.
Copyright Andrew Tridgell and the Samba Team 1992-2025
daemon 'smbd' : Starting process ...
```

## WINBINDD: servicio y registro

## Crear directorios de servicios y registros

```bash
mkdir -p /etc/sv/winbindd/log
mkdir -p /var/log/winbindd
```

## Crear /etc/sv/winbindd/run

```bash
vim /etc/sv/winbindd/run
```

## Contenido

```bash
#!/bin/sh
exec 2>&1
exec /opt/samba/sbin/winbindd --foreground --no-process-group
```

## Permiso

```bash
chmod +x /etc/sv/winbindd/run
```

## Crear /etc/sv/winbindd/log/run

```bash
vim /etc/sv/winbindd/log/run
```

## Contenido

```bash
#!/bin/sh
exec 2>&1
exec /opt/samba/sbin/winbindd --foreground --no-process-group
```

## Permiso

```bash
chmod +x /etc/sv/winbindd/log/run
```

## Depurar (opcional)

```bash
/opt/samba/sbin/winbindd -i
```

## Resultado obtenido

```bash
winbindd version 4.23.4 started.
Copyright Andrew Tridgell and the Samba Team 1992-2025
daemon 'winbindd' : Starting process ...
```

## NMBD: servicio y registro (opcional)

### Habilítelo solo si su entorno requiere navegación NetBIOS/SMB1.

## Crear directorios de servicios y registros

```bash
mkdir -p /etc/sv/nmbd/log
mkdir -p /var/log/nmbd
```

## Crear /etc/sv/nmbd/run

```bash
vim /etc/sv/nmbd/run
```

## Contenido

```bash
#!/bin/sh
exec 2>&1
exec /opt/samba/sbin/nmbd --foreground --no-process-group
```

## Permiso

```bash
chmod +x /etc/sv/nmbd/run
```

## Crear /etc/sv/nmbd/log/run

```bash
vim /etc/sv/nmbd/log/run
```
 
## Contenido

```bash
#!/bin/sh
exec svlogd -tt /var/log/nmbd
```

## Permiso

```bash
chmod +x /etc/sv/nmbd/log/run
```

## Habilitar servicios

```bash
ln -sf /etc/sv/smbd /var/service/
ln -sf /etc/sv/winbindd /var/service/
```

## Opcional: habilítelo solo si usa NetBIOS:

```bash
ln -sf /etc/sv/nmbd /var/service/
```

## Validar servicios

```bash
sv status smbd winbindd nmbd
```

## 🧪 Validar integración

```bash
net ads testjoin
```

## Resultado obtenido

```bash
Join is OK
```

```bash
wbinfo -u
```

## Resultado obtenido

```bash
guest
krbtgt
administrator```

```bash
wbinfo-g
```

## Result obtained

```bash
controladores de dominio empresariales de solo lectura
usuarios protegidos
controladores de dominio
invitados del dominio
controladores de dominio de solo lectura
administradores de esquema
dnsupdateproxy
administradores de dominio
propietarios del creador de políticas de grupo
servidores ras e ias
administradores de DNS
grupo de replicación de contraseña rodc permitido
administradores empresariales
editores de certificados
usuarios del dominio
grupo de replicación de contraseña de rodc denegado
computadoras de dominio
```

```bash
wbinfo --ping-dc
```

## Result obtained

```bash
la verificación de NETLOGON para la conexión dc del dominio [EDUCATUX] a "voiddc.educatux.edu" fue exitosa
```

## ✅ FINAL SUMMARY

## 🎉 Congratulations — you have successfully deployed a fully functional File Server on Void Linux!

## 👉 REMEMBER: While Samba4 can be managed via CLI, it was designed to be managed via RSAT Remote Server Administration Tools, which can be installed on a Windows 10 machine without issues!

---

🎯 THAT'S ALL FOLKS!

👉 Contact: zerolies@disroot.org

👉 https://t.me/z3r0l135
