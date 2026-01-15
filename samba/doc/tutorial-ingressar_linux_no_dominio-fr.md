# Configuration LightDM pour joindre Linux Mint au domaine

## 🎯 Objectif Rejoindre Linux au domaine Samba4 avec Winbind via l'intégration NSS/PAM

---

## Conditions préalables

- Samba4 comme contrôleur de domaine (PDC)
- Linux avec DNS et heure alignée sur PDC (192.168.70.253)
- Connectivité au serveur

---

## 🛠️ Étapes

## 1. Installez les packages requis

```bash
sudo apt update && sudo apt install samba winbind libpam-winbind libnss-winbind krb5-user
```

## 2. Configurez /etc/krb5.conf

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

## 3. Configurez /etc/samba/smb.conf

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

## 4. Configurez /etc/nsswitch.conf

```bash
passwd:         compat winbind
group:          compat winbind
shadow:         compat
```

## 5. Rejoignez le domaine

```bash
sudo net ads join -U Administrator
```

## 6. Redémarrage du système

```bash
sudo reboot
```

## 7. Chèques

```bash
sudo net ads testjoin
wbinfo -u
wbinfo -g
getent passwd usuario
```

## 8. Créez automatiquement des répertoires HOME


## Modifiez /etc/pam.d/common-session et ajoutez :

```bash
session required pam_mkhomedir.so skel=/etc/skel umask=0022
```

## 9. Redémarrez les services

```bash
sudo systemctl restart smbd nmbd winbind
sudo systemctl enable winbind
```

## 10. Synchronisation du temps

```bash
sudo timedatectl set-ntp true
```

## SI vous êtes un utilisateur Lightdm, comme c'est le cas avec Mint, configurez-le pour qu'il se connecte avec un utilisateur réseau, au lieu d'un simple utilisateur local.

## 🛠️ Étape par étape : configurez LightDM pour accepter les utilisateurs de domaine

## 1. Modifiez le fichier de configuration LightDM

```bash
sudo vim /etc/lightdm/lightdm.conf
```

## Modifiez les lignes suivantes dans le fichier :

```bash
[Seat:*]
greeter-show-manual-login=true
greeter-hide-users=true
allow-guest=false
```

## Explications :

- greeter-show-manual-login=true : Vous permet de saisir le nom d'utilisateur manuellement.
- greeter-hide-users=true : masque la liste locale des utilisateurs (utile pour les environnements d'entreprise).
- allow-guest=false : empêche la connexion des invités (pour des raisons de sécurité).

## 2. Assurez-vous que PAM autorise les utilisateurs du domaine

## Si vous avez utilisé SSSD ou Winbind, PAM devrait déjà être correctement intégré. Mais assurez-vous que le module home est présent :

```bash
sudo vim /etc/pam.d/common-session
```

## Confirmez que cette ligne existe OU ajoutez-la :

```bash
session required pam_mkhomedir.so skel=/etc/skel umask=0022
```

## 3. Redémarrez LightDM

```bash
sudo timedatectl set-ntp true
```

## ⚠️ Pour Linux MInt basé sur Ubuntu, vous pouvez désactiver la résolution système pour contrôler manuellement le DNS.

```bash
systemctl status systemd-resolved
```

```bash
systemctl stop systemd-resolved
systemctl disable systemd-resolved.service
sudo systemctl mask systemd-resolved
```

## SUPPRESSION DU FICHIER SANS AUTORISATION DE MODIFICATION CRÉÉ PAR SYSTEMD-RESOLVED :

```bash
rm -f /etc/resolv.conf
```

## Création d'un nouveau fichier avec autorisation de modification :

```bash
vim /etc/resolv.conf
```

```bash
domain educatux.edu
search educatux.edu.
nameserver 192.168.70.253
```

## Verrouillage du fichier contre l'édition automatique

```bash
sudo chattr +i /etc/resolv.conf
```

## Redémarrage du service

```bash
sudo systemctl restart NetworkManager
```

---

🎯 C'EST TOUS LES GENS !

👉Contact : zerolies@disroot.org
👉 https://t.me/z3r0l135

