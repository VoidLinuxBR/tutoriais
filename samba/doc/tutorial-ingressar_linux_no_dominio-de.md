# LightDM-Konfiguration für den Beitritt von Linux Mint zur Domäne

## 🎯 Ziel Verbinden Sie Linux mit Winbind durch NSS/PAM-Integration mit der Samba4-Domäne

---

## Voraussetzungen

- Samba4 als Domänencontroller (PDC)
- Linux mit DNS und Zeitabgleich mit PDC (192.168.70.253)
- Konnektivität zum Server

---

## 🛠️ Schritte

## 1. Installieren Sie die erforderlichen Pakete

```bash
sudo apt update && sudo apt install samba winbind libpam-winbind libnss-winbind krb5-user
```

## 2. Konfigurieren Sie /etc/krb5.conf

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

## 3. Konfigurieren Sie /etc/samba/smb.conf

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

## 4. Konfigurieren Sie /etc/nsswitch.conf

```bash
passwd:         compat winbind
group:          compat winbind
shadow:         compat
```

## 5. Treten Sie der Domäne bei

```bash
sudo net ads join -U Administrator
```

## 6. Systemneustart

```bash
sudo reboot
```

## 7. Schecks

```bash
sudo net ads testjoin
wbinfo -u
wbinfo -g
getent passwd usuario
```

## 8. Erstellen Sie automatisch HOME-Verzeichnisse


## Bearbeiten Sie /etc/pam.d/common-session und fügen Sie hinzu:

```bash
session required pam_mkhomedir.so skel=/etc/skel umask=0022
```

## 9. Starten Sie die Dienste neu

```bash
sudo systemctl restart smbd nmbd winbind
sudo systemctl enable winbind
```

## 10. Zeitsynchronisation

```bash
sudo timedatectl set-ntp true
```

## Wenn Sie ein Lightdm-Benutzer sind, wie es bei Mint der Fall ist, stellen Sie die Anmeldung so ein, dass Sie sich mit einem Netzwerkbenutzer und nicht nur mit einem lokalen Benutzer anmelden.

## 🛠️ Schritt für Schritt: Konfigurieren Sie LightDM so, dass es Domänenbenutzer akzeptiert

## 1. Bearbeiten Sie die LightDM-Konfigurationsdatei

```bash
sudo vim /etc/lightdm/lightdm.conf
```

## Bearbeiten Sie die folgenden Zeilen in der Datei:

```bash
[Seat:*]
greeter-show-manual-login=true
greeter-hide-users=true
allow-guest=false
```

## Erläuterungen:

- Greeter-show-manual-login=true: Ermöglicht die manuelle Eingabe des Benutzernamens.
- Greeter-hide-users=true: Versteckt die lokale Liste der Benutzer (nützlich für Unternehmensumgebungen).
- allow-guest=false: Verhindert die Gastanmeldung (aus Sicherheitsgründen).

## 2. Stellen Sie sicher, dass PAM Domänenbenutzer zulässt

## Wenn Sie SSSD oder Winbind verwendet haben, sollte PAM bereits korrekt integriert sein. Stellen Sie jedoch sicher, dass das Home-Modul vorhanden ist:

```bash
sudo vim /etc/pam.d/common-session
```

## Bestätigen Sie, dass diese Zeile existiert ODER fügen Sie sie hinzu:

```bash
session required pam_mkhomedir.so skel=/etc/skel umask=0022
```

## 3. Starten Sie LightDM neu

```bash
sudo timedatectl set-ntp true
```

## ⚠️ Für Ubuntu-basiertes Linux MInt können Sie systemd-resolved deaktivieren, um DNS manuell zu steuern.

```bash
systemctl status systemd-resolved
```

```bash
systemctl stop systemd-resolved
systemctl disable systemd-resolved.service
sudo systemctl mask systemd-resolved
```

## ENTFERNEN DER DATEI OHNE BEARBEITUNGSBERECHTIGUNG, ERSTELLT VON SYSTEMD-RESOLVED:

```bash
rm -f /etc/resolv.conf
```

## Erstellen einer neuen Datei mit Bearbeitungsberechtigung:

```bash
vim /etc/resolv.conf
```

```bash
domain educatux.edu
search educatux.edu.
nameserver 192.168.70.253
```

## Sperren der Datei gegen automatische Bearbeitung

```bash
sudo chattr +i /etc/resolv.conf
```

## Neustart des Dienstes

```bash
sudo systemctl restart NetworkManager
```

---

🎯 DAS IST ALLES, LEUTE!

👉 Kontakt: zerolies@disroot.org
👉 https://t.me/z3r0l135

