# Tutorial Distrobox no Void Linux

Distrobox te permite usar otras distribuciones de Linux dentro de tu Void Linux
sin comprometer el sistema base (host).
La lógica es la de siempre: **sistema limpio, pruebas aisladas, cero soluciones**.

---

## ¿Cuál es la propuesta aquí?

Instale distrobox en Void Linux y utilice contenedores para ejecutar otras distribuciones de forma segura.

Esto elimina el riesgo de:
- romper las dependencias del sistema
- contaminar el host con paquetes de otras distribuciones
- convertir el vacío en frankenstein

Nota: esta guía asume que Void Linux ya está instalado.

---

## Primero que nada: repositorio chililinux

El paquete `distrobox` no existe en los repositorios oficiales de Void.
Pero fue empaquetado por la Comunidad VoidLinuxBR, por lo que es necesario agregar el repositorio chililinux (espejo oficial de Void en Brasil - <https://xmirror.voidlinux.org/>).

Ejecute **exactamente** los siguientes comandos:
```bash
sudo sh -c "{
  echo 'repository=https://repo-fastly.voidlinux.org/current'
  echo 'repository=https://void.chililinux.com/voidlinux/current'
} > /etc/xbps.d/00-repository-main.conf"
```

---

## Actualización del sistema base

Antes de instalar cualquier cosa, asegúrese de que su sistema esté actualizado:

```bash
sudo xbps-install -Syu xbps
sudo xbps-install -Syu libssh2 xtools
sudo xbps-install -Suy
xcheckrestart
```
Si `xcheckrestart` indica reiniciar, reinicie.

---

## Instalación de Distrobox y dependencias

Ahora, instale los paquetes necesarios:

```bash
sudo xbps-install -Syf voidbr-distrobox podman docker crun
```

Importante:
Después de instalar `crun`, es obligatorio reiniciar el sistema:

```bash
sudo reboot
```

---

## Acerca de la compatibilidad de distribución

No todas las distribuciones funcionan bien en contenedores.
Antes de elegir, consulta la lista oficial:

CHILE_REF_0_CHILI

Esto evita pérdidas de tiempo y dolores de cabeza.

---

## Creando el primer contenedor (Debian)

Como ejemplo se utilizará Debian Testing.

```bash
distrobox create -Y --name debian --image docker.io/library/debian:testing
```

¿Qué está pasando aquí?
- `distrobox create` crea el contenedor
- `-Y` evita preguntas interactivas
- `--name` define el nombre del contenedor
- `--image` define la imagen base

Para ver todas las opciones disponibles:

```bash
distrobox --help
```

---

## Entrando al contenedor

Después de extraer la imagen, ingrese al contenedor:

```bash
distrobox enter debian
```

Dentro de Debian el uso es normal:

```bash
sudo apt update
sudo apt upgrade
sudo apt autoremove
sudo apt install firefox
```

Estás literalmente dentro de otra distribución.

---

## Ejecutar comandos sin ingresar al contenedor

También puede ejecutar comandos directamente desde el host.

Ejemplo: instalar Firefox en Debian sin entrar en él:

```bash
distrobox enter debian -- sudo apt install -y firefox-esr-l10n-pt-br
```

Práctico, rápido y tradicional.

---

## Exportar aplicaciones al sistema host

Distrobox te permite exportar aplicaciones desde el contenedor
al menú gráfico de VoidLinuxBR.

Ejemplo: exportar Firefox desde el contenedor de Debian:

```bash
distrobox enter debian -- distrobox-export --app firefox
```

La aplicación aparecerá en el menú del entorno gráfico.
como si fuera nativo.

---

## Actualizando todos los contenedores

Para actualizar todos los contenedores a la vez,
ejecutar sin host:

```bash
distrobox-upgrade --all -v
```

---

## Listado de contenedores existentes

Para ver todos los contenedores creados:

```bash
distrobox list
```

Se muestran el nombre, el estado y la imagen utilizada.

---

## Detener un contenedor

Si solo necesita detener el contenedor:

```bash
distrobox stop debian
```

---

## Quitar un contenedor

Para eliminar el contenedor de Distrobox:

```bash
distrobox rm debian
```

Si desea eliminar también la imagen de Podman:

```bash
podman rmi -f [IMAGE ID]
```

---

## Observaciones finales

- Utilice contenedores para realizar pruebas, no para saturar el host
- Ajuste nombres e imágenes según sea necesario
- Consulta siempre la documentación oficial:
CHILE_REF_0_CHILI
- Pruebe primero en una máquina virtual o de laboratorio

Distrobox es una herramienta para quienes gustan del control,
Aislamiento y sistema en buen estado.

---

## 📜 Créditos

Creado por: Robson Nakane <theblizzard1983@hotmail.com>
Comunidad: Void Linux Brasil <https://github.com/voidlinuxbr>
Distribución: Chili Linux <https://chililinux.com>
Distribución: VoidBR <https://github.com/voidlinuxbr>

---

## ⚖️ Disclaimer (Aviso Legal)

ESTE SOFTWARE/TUTORIAL SE PROPORCIONA "TAL CUAL" SIN NINGUNA GARANTÍA
DE CUALQUIER TIPO, YA SEA EXPRESO O IMPLÍCITO, INCLUYENDO, PERO NO LIMITADO A,
GARANTÍAS DE COMERCIABILIDAD O IDONEIDAD PARA UN PROPÓSITO PARTICULAR.

EL USO DE ESTE SOFTWARE ES TOTAL RESPONSABILIDAD DEL USUARIO.

EN NINGÚN MOMENTO EL AUTOR O LOS COLABORADORES SERÁN RESPONSABLES DE
CUALQUIER DAÑO, PÉRDIDA DE DATOS O FALLA DEL SISTEMA QUE SURJA DEL USO
DE ESTE PROGRAMA.

---

Copyright (C) 2026 Robson Nakane
