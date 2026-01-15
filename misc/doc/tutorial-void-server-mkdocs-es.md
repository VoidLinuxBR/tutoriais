# Instalación del servidor Mkdocs en Void Linux

## 🎯 Objetivo: cargar MkDocs Server, un generador de sitios web de documentación estática rápido, simple y centrado en proyectos. Convierte archivos Markdown simples en un sitio web de documentación profesional y totalmente navegable. La configuración se realiza a través de un único archivo YAML (mkdocs.yml) y el contenido se escribe en Markdown estándar. Es ideal para crear documentación técnica, manuales de usuario o bases de conocimientos, y ofrece un servidor de desarrollo integrado para visualización en tiempo real.

---

## Instalar dependencias del sistema (Python y pipx) a través de XBPS

```bash
sudo xbps-install -S python3 python3-pipx
```

## 🏠 Instale el paquete mkdocs en el entorno virtual Python

```bash
pipx install mkdocs
```

## Agregue la nueva ruta al Sistema Operativo, local o globalmente

## Local

```bash
pipx ensurepath
```

## Global

```bash
sudo pipx ensurepath --global
```

## La ubicación aparecerá en el .bashrc del usuario.

```bash
# Created by `pipx` on 2025-11-27 14:07:54
export PATH="$PATH:/home/suporte/.local/bin"
```

## Validar la nueva ruta del usuario al Sistema Operativo

```bash
source ~/.bashrc
```

## Validar la instalación del paquete.

```bash
mkdocs --version
```

## Instalación del tema Material en el entorno virtual Python

```bash
pipx inject mkdocs mkdocs-material
```

## La inyección instalará el paquete de temas en una ruta oculta, en la casa del usuario.

```bash
/home/suporte/.local/bin/mkdocs
```

## Secuencia de uso de la herramienta:

## 1. Crea un nuevo proyecto

## 🔧 Para iniciar un nuevo proyecto de documentación, navegue hasta el directorio donde desea crear el proyecto y ejecute:

```bash
mkdocs new Void_Artigos
```

## Esto creará un nuevo directorio llamado Void_Artigos con la estructura básica de MkDocs.

## 2. Utilice el tema Material (opcional)

## 🧩 Si creó un nuevo proyecto, edite el archivo de configuración mkdocs.yml dentro del directorio del proyecto (Void_Artigos/mkdocs.yml) y agregue la configuración del tema Material:

```bash
site_name: Void Artigos
nav:
    - Home: index.md
    - Sobre: about.md

theme:
  name: material # Adicione esta linha para usar o tema Material
```

## 3. Inicie el servidor de desarrollo

## Para ver su documentación localmente mientras la edita, navegue hasta el directorio de su proyecto e inicie el servidor de desarrollo:

```bash
cd void-Artigos
```

```bash
mkdocs serve
```

## El servidor se iniciará y podrá acceder a la documentación en su navegador, generalmente en http://127.0.0.1:8000.. MkDocs monitoreará automáticamente los cambios en sus archivos y recargará la página.

## Para dar servicio a la red interna, proporcione la IP y el puerto del Servidor

```bash
mkdocs serve 192.168.70.100:8000
```

## Ser accesible desde cualquier navegador de la red interna

```bash
http://192.168.70.100:8000
```

## 4. Cree documentación estática

## Cuando su documentación esté lista para ser publicada, cree los archivos estáticos:

```bash
mkdocs build
```

## Esto creará un directorio llamado sitio/ que contiene todos los archivos HTML, CSS y JavaScript necesarios para alojar su documentación en cualquier servidor web. En resumen, estar en Void Linux no cambia el flujo de trabajo de MkDocs, gracias al uso de pipx que aísla efectivamente la aplicación.

---

🎯 ¡ESO ES TODO AMIGOS!

👉 Contacto: zerolies@disroot.org
👉https://t.me/z3r0l135
