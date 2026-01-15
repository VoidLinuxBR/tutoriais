# Installation du serveur Mkdocs sur Void Linux

## 🎯 Objectif - Téléchargez le serveur MkDocs, un générateur de site Web de documentation statique rapide, simple et axé sur le projet. Il transforme de simples fichiers Markdown en un site Web de documentation professionnel entièrement navigable. La configuration se fait via un seul fichier YAML (mkdocs.yml) et le contenu est écrit en Markdown standard. Il est idéal pour créer de la documentation technique, des manuels d'utilisation ou des bases de connaissances, offrant un serveur de développement intégré pour une visualisation en temps réel.

---

## Installer les dépendances système (Python et pipx) via XBPS

```bash
sudo xbps-install -S python3 python3-pipx
```

## 🏠 Installez le package mkdocs dans l'environnement virtuel Python

```bash
pipx install mkdocs
```

## Ajoutez le nouveau chemin d'accès au système d'exploitation, localement ou globalement

## Locale

```bash
pipx ensurepath
```

## Mondial

```bash
sudo pipx ensurepath --global
```

## L'emplacement apparaîtra dans le .bashrc de l'utilisateur

```bash
# Created by `pipx` on 2025-11-27 14:07:54
export PATH="$PATH:/home/suporte/.local/bin"
```

## Valider le nouveau chemin de l'utilisateur vers le système d'exploitation

```bash
source ~/.bashrc
```

## Valider l'installation du package

```bash
mkdocs --version
```

## Installation du thème Material dans l'environnement virtuel Python

```bash
pipx inject mkdocs mkdocs-material
```

## L'injection installera le package du thème dans un chemin caché, chez l'utilisateur

```bash
/home/suporte/.local/bin/mkdocs
```

## Séquence d'utilisation de l'outil :

## 1. Créer un nouveau projet

## 🔧 Pour démarrer un nouveau projet de documentation, accédez au répertoire dans lequel vous souhaitez créer le projet et exécutez :

```bash
mkdocs new Void_Artigos
```

## Cela créera un nouveau répertoire appelé Void_Artigos avec la structure de base MkDocs.

## 2. Utiliser le thème matériel (facultatif)

## 🧩 Si vous avez créé un nouveau projet, modifiez le fichier de configuration mkdocs.yml dans le répertoire du projet (Void_Artigos/mkdocs.yml) et ajoutez la configuration du thème Material :

```bash
site_name: Void Artigos
nav:
    - Home: index.md
    - Sobre: about.md

theme:
  name: material # Adicione esta linha para usar o tema Material
```

## 3. Démarrez le serveur de développement

## Pour afficher votre documentation localement tout en la modifiant, accédez au répertoire de votre projet et démarrez le serveur de développement :

```bash
cd void-Artigos
```

```bash
mkdocs serve
```

## Le serveur démarrera et vous pourrez accéder à la documentation dans votre navigateur, généralement à http://127.0.0.1:8000.. MkDocs surveillera automatiquement les modifications apportées à vos fichiers et rechargera la page.

## Pour desservir le réseau interne, fournissez l'adresse IP et le port du serveur

```bash
mkdocs serve 192.168.70.100:8000
```

## Être accessible depuis n'importe quel navigateur du réseau interne

```bash
http://192.168.70.100:8000
```

## 4. Créer une documentation statique

## Lorsque votre documentation est prête à être publiée, créez les fichiers statiques :

```bash
mkdocs build
```

## Cela créera un répertoire appelé site/ contenant tous les fichiers HTML, CSS et JavaScript nécessaires pour héberger votre documentation sur n'importe quel serveur Web. Bref, être sur Void Linux ne change pas le workflow MkDocs, grâce à l'utilisation de pipx qui isole efficacement l'application.

---

🎯 C'EST TOUS LES GENS !

👉Contact : zerolies@disroot.org
👉 https://t.me/z3r0l135
