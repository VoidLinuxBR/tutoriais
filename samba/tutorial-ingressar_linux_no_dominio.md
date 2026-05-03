# Configuração do LightDM para inserção do Linux Mint no domínio

## 🎯 Objetivo Ingressar Linux no Domínio Samba4 com Winbind por integração NSS/PAM

---

## Pré-requisitos

- Samba4 como controlador de domínio (PDC)
- Linux com DNS e horário alinhados com o PDC (192.168.70.253)
- Conectividade com o servidor

---

## 🛠️ Etapas

## 1. Instalar pacotes necessários

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

## 5. Ingressar no domínio

```bash
kinit administrator
sudo net ads join -U Administrator
```

## 6. Reboot do Sistema

```bash
sudo reboot
```

## 7. Verificações

```bash
sudo net ads testjoin
wbinfo -u
wbinfo -g
getent passwd usuario
```

## 8. Criar diretórios HOME automaticamente


## Edite /etc/pam.d/common-session e adicione:

```bash
session required pam_mkhomedir.so skel=/etc/skel umask=0022
```

## 9. Reiniciar serviços

```bash
sudo systemctl restart smbd nmbd winbind
sudo systemctl enable winbind
```

## 10. Sincronização de Hora

```bash
sudo timedatectl set-ntp true
```

## SE você for usuário de Lightdm, como é o caso do Mint, ajuste pra logar com usuário de rede, ao invés de usuário local apenas.

## 🛠️ Passo a Passo: Configurar LightDM para aceitar usuários do domínio

## 1. Editar o arquivo de configuração do LightDM

```bash
sudo vim /etc/lightdm/lightdm.conf
```

## Edite as seguintes linhas do arquivo:

```bash
[Seat:*]
greeter-show-manual-login=true
greeter-hide-users=true
allow-guest=false
```

## Explicações:

- greeter-show-manual-login=true: Permite digitar o nome de usuário manualmente.
- greeter-hide-users=true: Esconde a lista local de usuários (útil para ambientes corporativos).
- allow-guest=false: Impede login de convidados (por segurança).

## 2. Certifique-se de que PAM está permitindo usuários do domínio

## Se você usou SSSD ou Winbind, o PAM já deve estar integrado corretamente. Mas valide que o módulo home esteja presente:

```bash
sudo vim /etc/pam.d/common-session
```

## Confirme que esta linha existe OU adicione-a:

```bash
session required pam_mkhomedir.so skel=/etc/skel umask=0022
```

## 3. Reiniciar o LightDM

```bash
sudo timedatectl set-ntp true
```

## ⚠️ Para Linux MInt baseado em Ubuntu, pode-se desativar o systemd-resolved para controlar o DNS manualmente.

```bash
systemctl status systemd-resolved
```

```bash
systemctl stop systemd-resolved
systemctl disable systemd-resolved.service
sudo systemctl mask systemd-resolved
```

## REMOVENDO O ARQUIVO SEM PERMISSÃO DE EDIÇÃO CRIADO PELO SYSTEMD-RESOLVED:

```bash
rm -f /etc/resolv.conf
```

## Criando um arquivo novo com permissão de edição:

```bash
vim /etc/resolv.conf
```

```bash
domain educatux.edu
search educatux.edu.
nameserver 192.168.70.253
```

## Bloqueando o arquivo contra edição automática

```bash
sudo chattr +i /etc/resolv.conf
```

## Restart do serviço

```bash
sudo systemctl restart NetworkManager
```

---

🎯 THAT'S ALL FOLKS!

👉 Contato: zerolies@disroot.org
👉 https://t.me/z3r0l135

