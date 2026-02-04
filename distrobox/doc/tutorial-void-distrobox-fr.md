# Tutoriel Distrobox sans Void Linux

Distrobox vous permet d'utiliser d'autres distributions Linux dans votre Void Linux
sans compromettre le système de base (hôte).
La logique est comme d'habitude : **système propre, tests isolés, aucune solution de contournement**.

---

## Quelle est la proposition ici

Installez distrobox sur Void Linux et utilisez des conteneurs pour exécuter d'autres distributions en toute sécurité.

Cela élimine le risque de :
- briser les dépendances du système
- polluer l'hôte avec des packages d'autres distributions
- transformer le Vide en Frankenstein

Remarque : ce guide suppose que Void Linux est déjà installé.

---

## Tout d'abord : le dépôt Chililinux

Le package `distrobox` n'existe pas dans les référentiels officiels de Void.
Mais il a été packagé par la communauté VoidLinuxBR, il est donc nécessaire d'ajouter le dépôt chililinux (miroir officiel de Void au Brésil - <https://xmirror.voidlinux.org/>).

Exécutez **exactement** les commandes ci-dessous :
```bash
sudo sh -c "{
  echo 'repository=https://repo-fastly.voidlinux.org/current'
  echo 'repository=https://void.chililinux.com/voidlinux/current'
} > /etc/xbps.d/00-repository-main.conf"
```

---

## Mise à jour du système de base

Avant d'installer quoi que ce soit, assurez-vous que votre système est à jour :

```bash
sudo xbps-install -Syu xbps
sudo xbps-install -Syu libssh2 xtools
sudo xbps-install -Suy
xcheckrestart
```
Si `xcheckrestart` indique un redémarrage, redémarrez.

---

## Installation de Distrobox et des dépendances

Maintenant, installez les packages nécessaires :

```bash
sudo xbps-install -Syf voidbr-distrobox podman docker crun
```

Important:
Après avoir installé `crun`, il est obligatoire de redémarrer le système :

```bash
sudo reboot
```

---

## À propos de la compatibilité des distributions

Toutes les distributions ne fonctionnent pas bien dans les conteneurs.
Avant de choisir, consultez la liste officielle :

https://distrobox.it/compatibility/#containers-distros

Cela évite les pertes de temps et les maux de tête.

---

## Création du premier conteneur (Debian)

À titre d'exemple, Debian Testing sera utilisé.

```bash
distrobox create -Y --name debian --image docker.io/library/debian:testing
```

Que se passe-t-il ici :
- `distrobox create` crée le conteneur
- `-Y` évite les questions interactives
- `--name` définit le nom du conteneur
- `--image` définit l'image de base

Pour voir toutes les options disponibles :

```bash
distrobox --help
```

---

## Entrer dans le conteneur

Après avoir extrait l'image, entrez dans le conteneur :

```bash
distrobox enter debian
```

Dans Debian, l'utilisation est normale :

```bash
sudo apt update
sudo apt upgrade
sudo apt autoremove
sudo apt install firefox
```

Vous êtes littéralement dans une autre distribution.

---

## Exécuter des commandes sans entrer dans le conteneur

Vous pouvez également exécuter des commandes directement depuis l'hôte.

Exemple : installer Firefox sur Debian sans y entrer :

```bash
distrobox enter debian -- sudo apt install -y firefox-esr-l10n-pt-br
```

Pratique, rapide et traditionnel.

---

## Exportation d'applications vers le système hôte

Distrobox vous permet d'exporter des applications depuis le conteneur
au menu graphique VoidLinuxBR.

Exemple : exporter Firefox depuis le conteneur Debian :

```bash
distrobox enter debian -- distrobox-export --app firefox
```

L'application apparaîtra dans le menu de l'environnement graphique
comme s'il était indigène.

---

## Mettre à jour tous les conteneurs

Pour mettre à jour tous les conteneurs en même temps,
n'exécutez aucun hôte :

```bash
distrobox-upgrade --all -v
```

---

## Liste des conteneurs existants

Pour voir tous les conteneurs créés :

```bash
distrobox list
```

Le nom, le statut et l'image utilisés sont affichés.

---

## Arrêter un conteneur

Si vous avez juste besoin d'arrêter le conteneur :

```bash
distrobox stop debian
```

---

## Supprimer un conteneur

Pour supprimer le conteneur Distrobox :

```bash
distrobox rm debian
```

Si vous souhaitez également supprimer l'image Podman :

```bash
podman rmi -f [IMAGE ID]
```

---

## Observations finales

- Utilisez des conteneurs pour tester, pas pour encombrer l'hôte
- Ajustez les noms et les images selon vos besoins
- Consultez toujours la documentation officielle :
https://distrobox.it
- Testez d'abord sur une VM ou un ordinateur de laboratoire

Distrobox est un outil pour ceux qui aiment le contrôle,
isolation et système bien entretenu.

---

## 📜 Crédits

Créé par : Robson Nakane <theblizzard1983@hotmail.com>
Communauté : Void Linux Brésil <https://github.com/voidlinuxbr>
Distribution : Chili Linux <https://chililinux.com>
Distribution : VoidBR <https://github.com/voidlinuxbr>

---

## ⚖️ Avis de non-responsabilité (Mentions légales)

CE LOGICIEL/TUTORIEL EST FOURNI « TEL QUEL » SANS ABSOLUMENT AUCUNE GARANTIE
DE TOUTE SORTE, EXPRESSE OU IMPLICITE, Y COMPRIS, MAIS SANS LIMITATION,
GARANTIES DE QUALITÉ MARCHANDE OU D’ADAPTATION À UN USAGE PARTICULIER.

L'UTILISATION DE CE LOGICIEL EST L'ENTIÈRE RESPONSABILITÉ DE L'UTILISATEUR.

A AUCUN MOMENT L'AUTEUR OU LES CONTRIBUTEURS NE SERONT RESPONSABLES DE
TOUT DOMMAGE, PERTE DE DONNÉES OU DÉFAILLANCE DU SYSTÈME DÉCOULANT DE L'UTILISATION
DE CE PROGRAMME.

---

Copyright (C) 2026 Robson Nakane
