# Configuración de LightDM para unir Linux Mint al dominio

## 🎯 Objetivo Unir Linux al dominio Samba4 con Winbind a través de la integración NSS/PAM

---

## Requisitos previos

- Samba4 como controlador de dominio (PDC)
- Linux con DNS y tiempo alineado con PDC (192.168.70.253)
- Conectividad al servidor

---

## 🛠️ Pasos

## 1. Instale los paquetes necesarios

```bash
sudo apt update && sudo apt install samba winbind libpam-winbind libnss-winbind krb5-user
```

## 2. Configurar /etc/krb5.conf

```bash
[libdefaults]
    default_realm = EDUCATUX.EDU
    dns_lookup_realm = false
    dns_lookup_kdc = true
    kdc_timesync = 1
    ccache_type = 4
    forwardable = true
    proxiable = true
    rdns = false
    fcc-mit-ticketflags = true

[realms]
    EDUCATUX.EDU = {
        kdc = 192.168.70.253
        admin_server = 192.168.70.253
        default_domain = educatux.edu
    }

[domain_realm]
    .officinas.edu = EDUCATUX.EDU
    educatux.edu = EDUCATUX.EDU
```

## 3. Configurar /etc/samba/smb.conf

```bash
[global]
   workgroup = EDUCATUX
   security = ads
   realm = EDUCATUX.EDU
   server role = member server
   interfaces = lo eth0
   bind interfaces only = yes


   winbind use default domain = true
   winbind enum users = yes
   winbind enum groups = yes
   winbind refresh tickets = yes
   winbind offline logon = yes

   idmap config * : backend = tdb
   idmap config * : range = 10000-19999

   idmap config EDUCATUX : backend = rid
   idmap config EDUCATUX : range = 20000-999999

   template shell = /bin/bash
   template homedir = /home/%U
```

## 4. Configurar /etc/nsswitch.conf

```bash
passwd:         compat winbind
group:          compat winbind
shadow:         compat
```

## 5. Únase al dominio

```bash
sudo net ads join -U Administrator
```

## 6. Reinicio del sistema

```bash
sudo reboot
```

## 7. Cheques

```bash
sudo net ads testjoin
wbinfo -u
wbinfo -g
getent passwd usuario
```

## 8. Crear automáticamente directorios HOME


## Edite /etc/pam.d/common-session y agregue:

```bash
session required pam_mkhomedir.so skel=/etc/skel umask=0022
```

## 9. Reiniciar los servicios

```bash
sudo systemctl restart smbd nmbd winbind
sudo systemctl enable winbind
```

## 10. Sincronización horaria

```bash
sudo timedatectl set-ntp true
```

## SI es un usuario de Lightdm, como es el caso de Mint, configúrelo para iniciar sesión con un usuario de red, en lugar de solo con un usuario local.

## 🛠️ Paso a paso: configurar LightDM para aceptar usuarios de dominio

## 1. Edite el archivo de configuración de LightDM

```bash
sudo vim /etc/lightdm/lightdm.conf
```

## Edite las siguientes líneas en el archivo:

```bash
[Seat:*]
greeter-show-manual-login=true
greeter-hide-users=true
allow-guest=false
```

## Explicaciones:

- greeter-show-manual-login=true: Le permite ingresar el nombre de usuario manualmente.
- greeter-hide-users=true: Oculta la lista local de usuarios (útil para entornos corporativos).
- enable-guest=false: Impide el inicio de sesión de invitado (por seguridad).

## 2. Asegúrese de que PAM permita usuarios del dominio

## Si usó SSSD o Winbind, PAM ya debería estar integrado correctamente. Pero asegúrese de que el módulo de inicio esté presente:

```bash
sudo vim /etc/pam.d/common-session
```

## Confirme que esta línea existe O agréguela:

```bash
session required pam_mkhomedir.so skel=/etc/skel umask=0022
```

## 3. Reinicie LightDM

```bash
sudo timedatectl set-ntp true
```

## ⚠️ Para Linux MInt basado en Ubuntu, puede desactivar systemd-resolved para controlar DNS manualmente.

```bash
systemctl status systemd-resolved
```

```bash
systemctl stop systemd-resolved
systemctl disable systemd-resolved.service
sudo systemctl mask systemd-resolved
```

## ELIMINAR EL ARCHIVO SIN PERMISO DE EDICIÓN CREADO POR SYSTEMD-RESOLVED:

```bash
rm -f /etc/resolv.conf
```

## Creando un nuevo archivo con permiso de edición:

```bash
vim /etc/resolv.conf
```

```bash
domain educatux.edu
search educatux.edu.
nameserver 192.168.70.253
```

## Bloquear el archivo contra la edición automática

```bash
sudo chattr +i /etc/resolv.conf
```

## Reinicio del servicio

```bash
sudo systemctl restart NetworkManager
```

---

🎯 ¡ESO ES TODO AMIGOS!

👉 Contacto: zerolies@disroot.org
👉https://t.me/z3r0l135

