# Installation von Mkdocs Server unter Void Linux

## 🎯 Ziel – Laden Sie den MkDocs-Server hoch, einen schnellen, einfachen und projektorientierten Website-Generator für statische Dokumentation. Es verwandelt einfache Markdown-Dateien in eine professionelle, vollständig navigierbare Dokumentations-Website. Die Konfiguration erfolgt über eine einzige YAML-Datei (mkdocs.yml) und der Inhalt wird im Standard-Markdown geschrieben. Es ist ideal für die Erstellung technischer Dokumentationen, Benutzerhandbücher oder Wissensdatenbanken und bietet einen integrierten Entwicklungsserver für die Echtzeitanzeige.

---

## Installieren Sie Systemabhängigkeiten (Python und Pipx) über XBPS

```bash
sudo xbps-install -S python3 python3-pipx
```

## 🏠 Installieren Sie das mkdocs-Paket in der virtuellen Python-Umgebung

```bash
pipx install mkdocs
```

## Fügen Sie den neuen Pfad zum Betriebssystem hinzu, lokal oder global

## Lokal

```bash
pipx ensurepath
```

## Global

```bash
sudo pipx ensurepath --global
```

## Der Standort wird in der .bashrc-Datei des Benutzers angezeigt

```bash
# Created by `pipx` on 2025-11-27 14:07:54
export PATH="$PATH:/home/suporte/.local/bin"
```

## Validieren Sie den neuen Pfad des Benutzers zum Betriebssystem

```bash
source ~/.bashrc
```

## Validieren Sie die Paketinstallation

```bash
mkdocs --version
```

## Installieren des Material-Designs in der virtuellen Python-Umgebung

```bash
pipx inject mkdocs mkdocs-material
```

## Durch die Injektion wird das Designpaket in einem versteckten Pfad im Zuhause des Benutzers installiert

```bash
/home/suporte/.local/bin/mkdocs
```

## Reihenfolge der Verwendung des Tools:

## 1. Erstellen Sie ein neues Projekt

## 🔧 Um ein neues Dokumentationsprojekt zu starten, navigieren Sie zu dem Verzeichnis, in dem Sie das Projekt erstellen möchten, und führen Sie Folgendes aus:

```bash
mkdocs new Void_Artigos
```

## Dadurch wird ein neues Verzeichnis namens Void_Artigos mit der grundlegenden MkDocs-Struktur erstellt.

## 2. Materialthema verwenden (optional)

## 🧩 Wenn Sie ein neues Projekt erstellt haben, bearbeiten Sie die Konfigurationsdatei mkdocs.yml im Projektverzeichnis (Void_Artigos/mkdocs.yml) und fügen Sie die Material-Designkonfiguration hinzu:

```bash
site_name: Void Artigos
nav:
    - Home: index.md
    - Sobre: about.md

theme:
  name: material # Adicione esta linha para usar o tema Material
```

## 3. Starten Sie den Entwicklungsserver

## Um Ihre Dokumentation lokal anzuzeigen, während Sie sie bearbeiten, navigieren Sie zu Ihrem Projektverzeichnis und starten Sie den Entwicklungsserver:

```bash
cd void-Artigos
```

```bash
mkdocs serve
```

## Der Server wird gestartet und Sie können in Ihrem Browser auf die Dokumentation zugreifen, normalerweise unter http://127.0.0.1:8000.. MkDocs überwacht automatisch Änderungen an Ihren Dateien und lädt die Seite neu.

## Um das interne Netzwerk zu bedienen, geben Sie die IP und den Port des Servers an

```bash
mkdocs serve 192.168.70.100:8000
```

## Von jedem Browser im internen Netzwerk aus zugänglich

```bash
http://192.168.70.100:8000
```

## 4. Erstellen Sie eine statische Dokumentation

## Wenn Ihre Dokumentation zur Veröffentlichung bereit ist, erstellen Sie die statischen Dateien:

```bash
mkdocs build
```

## Dadurch wird ein Verzeichnis namens site/ erstellt, das alle HTML-, CSS- und JavaScript-Dateien enthält, die zum Hosten Ihrer Dokumentation auf einem beliebigen Webserver erforderlich sind. Kurz gesagt, die Verwendung von Void Linux ändert den MkDocs-Workflow nicht, dank der Verwendung von pipx, das die Anwendung effektiv isoliert.

---

🎯 DAS IST ALLES, LEUTE!

👉 Kontakt: zerolies@disroot.org
👉 https://t.me/z3r0l135
