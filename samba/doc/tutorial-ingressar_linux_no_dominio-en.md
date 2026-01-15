# LightDM configuration for joining Linux Mint to the domain

## 🎯 Objective Join Linux to the Samba4 Domain with Winbind through NSS/PAM integration

---

## Prerequisites

- Samba4 as domain controller (PDC)
- Linux with DNS and time aligned with PDC (192.168.70.253)
- Connectivity to the server

---

## 🛠️ Steps

## 1. Install required packages

```bash
sudo apt update && sudo apt install samba winbind libpam-winbind libnss-winbind krb5-user
```

## 2. Configure /etc/krb5.conf

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

## 3. Configure /etc/samba/smb.conf

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

## 4. Configure /etc/nsswitch.conf

```bash
passwd:         compat winbind
group:          compat winbind
shadow:         compat
```

## 5. Join the domain

```bash
sudo net ads join -U Administrator
```

## 6. System Reboot

```bash
sudo reboot
```

## 7. Checks

```bash
sudo net ads testjoin
wbinfo -u
wbinfo -g
getent passwd usuario
```

## 8. Automatically create HOME directories


## Edit /etc/pam.d/common-session and add:

```bash
session required pam_mkhomedir.so skel=/etc/skel umask=0022
```

## 9. Restart services

```bash
sudo systemctl restart smbd nmbd winbind
sudo systemctl enable winbind
```

## 10. Time Synchronization

```bash
sudo timedatectl set-ntp true
```

## IF you are a Lightdm user, as is the case with Mint, set it to log in with a network user, instead of just a local user.

## 🛠️ Step by Step: Configure LightDM to accept domain users

## 1. Edit the LightDM configuration file

```bash
sudo vim /etc/lightdm/lightdm.conf
```

## Edit the following lines in the file:

```bash
[Seat:*]
greeter-show-manual-login=true
greeter-hide-users=true
allow-guest=false
```

## Explanations:

- greeter-show-manual-login=true: Allows you to enter the username manually.
- greeter-hide-users=true: Hides the local list of users (useful for corporate environments).
- allow-guest=false: Prevents guest login (for security).

## 2. Make sure PAM is allowing domain users

## If you used SSSD or Winbind, PAM should already be integrated correctly. But make sure the home module is present:

```bash
sudo vim /etc/pam.d/common-session
```

## Confirm that this line exists OR add it:

```bash
session required pam_mkhomedir.so skel=/etc/skel umask=0022
```

## 3. Restart LightDM

```bash
sudo timedatectl set-ntp true
```

## ⚠️ For Ubuntu-based Linux MInt, you can disable systemd-resolved to control DNS manually.

```bash
systemctl status systemd-resolved
```

```bash
systemctl stop systemd-resolved
systemctl disable systemd-resolved.service
sudo systemctl mask systemd-resolved
```

## REMOVING THE FILE WITHOUT EDIT PERMISSION CREATED BY SYSTEMD-RESOLVED:

```bash
rm -f /etc/resolv.conf
```

## Creating a new file with edit permission:

```bash
vim /etc/resolv.conf
```

```bash
domain educatux.edu
search educatux.edu.
nameserver 192.168.70.253
```

## Locking the file against automatic editing

```bash
sudo chattr +i /etc/resolv.conf
```

## Service restart

```bash
sudo systemctl restart NetworkManager
```

---

🎯 THAT'S ALL FOLKS!

👉 Contact: zerolies@disroot.org
👉 https://t.me/z3r0l135

