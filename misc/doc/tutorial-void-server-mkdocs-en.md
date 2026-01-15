# Installing Mkdocs Server on Void Linux

## 🎯 Objective - Upload the MkDocs Server, a fast, simple and project-focused static documentation website generator. It turns simple Markdown files into a professional, fully navigable documentation website. Configuration is done through a single YAML file (mkdocs.yml), and content is written in standard Markdown. It's ideal for creating technical documentation, user manuals or knowledge bases, offering a built-in development server for real-time viewing.

---

## Install system dependencies (Python and pipx) via XBPS

```bash
sudo xbps-install -S python3 python3-pipx
```

## 🏠 Install the mkdocs package in the Python virtual environment

```bash
pipx install mkdocs
```

## Add the new path to the Operating System, locally or globally

## Local

```bash
pipx ensurepath
```

## Global

```bash
sudo pipx ensurepath --global
```

## Location will appear in the user's .bashrc

```bash
# Created by `pipx` on 2025-11-27 14:07:54
export PATH="$PATH:/home/suporte/.local/bin"
```

## Validate the user's new path to the Operating System

```bash
source ~/.bashrc
```

## Validate the package installation

```bash
mkdocs --version
```

## Installing the Material theme in the Python virtual environment

```bash
pipx inject mkdocs mkdocs-material
```

## The injection will install the theme package in a hidden path, in the user's home

```bash
/home/suporte/.local/bin/mkdocs
```

## Sequence of using the tool:

## 1. Create a New Project

## 🔧 To start a new documentation project, navigate to the directory where you want to create the project and run:

```bash
mkdocs new Void_Artigos
```

## This will create a new directory called Void_Artigos with the basic MkDocs structure.

## 2. Use Material Theme (Optional)

## 🧩 If you created a new project, edit the mkdocs.yml configuration file inside the project directory (Void_Artigos/mkdocs.yml) and add the Material theme configuration:

```bash
site_name: Void Artigos
nav:
    - Home: index.md
    - Sobre: about.md

theme:
  name: material # Adicione esta linha para usar o tema Material
```

## 3. Start the Development Server

## To view your documentation locally while editing it, navigate to your project directory and start the development server:

```bash
cd void-Artigos
```

```bash
mkdocs serve
```

## The server will start and you will be able to access the documentation in your browser, usually at http://127.0.0.1:8000.. MkDocs will automatically monitor changes to your files and reload the page.

## To serve the internal network, provide the Server's IP and port

```bash
mkdocs serve 192.168.70.100:8000
```

## Being accessible from any browser on the internal network

```bash
http://192.168.70.100:8000
```

## 4. Build Static Documentation

## When your documentation is ready to be published, build the static files:

```bash
mkdocs build
```

## This will create a directory called site/ containing all the HTML, CSS and JavaScript files needed to host your documentation on any web server. In short, being on Void Linux does not change the MkDocs workflow, thanks to the use of pipx which effectively isolates the application.

---

🎯 THAT'S ALL FOLKS!

👉 Contact: zerolies@disroot.org
👉 https://t.me/z3r0l135
