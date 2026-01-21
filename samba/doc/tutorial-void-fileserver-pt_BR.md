
# Servidor de arquivos rodando Samba4 no Void Linux Server ;D

## 🎯 Objetivo – Implantar um servidor de arquivos no Void Linux (glibc) compilando Samba4 a partir da fonte, integração AD, ACLs, serviços e toda a pilha necessária para um servidor de arquivos atender clientes de rede.

## 🔧 Laboratório de networking com QEMU/Virtmanager e Proxmox. Ajuste o tutorial para corresponder ao seu próprio ambiente.

---

## 📡 Layout de rede local

- Domínio: EDUCATUX.EDU

- Nome do host: servidor de arquivos

- Firewall 192.168.70.254 (DNS/GW)

- IP: 192.168.70.251

## Instale o Void Linux

## Altere o shell padrão no Void

```bash
chsh -s /bin/bash
```

## 🧩 Instale pacotes de dependência para compilar o Samba4 no Void

```bash
xbps-install -S \
 net-tools rsync acl attr attr-devel autoconf automake libtool \
 binutils bison gcc make ccache chrpath curl \
 docbook-xml docbook-xsl flex gdb git htop \
 mit-krb5 mit-krb5-client mit-krb5-devel \
 libarchive-devel avahi avahi-libs libblkid-devel \
 libbsd-devel libcap-devel cups-devel dbus-devel glib-devel \
 gnutls-devel gpgme-devel icu-devel jansson-devel \
 lmdb lmdb-devel libldap-devel ncurses-devel pam-devel perl \
 perl-Text-ParseWords perl-JSON perl-Parse-Yapp \
 libpcap-devel popt-devel readline-devel \
 libtasn1 libtasn1-devel libunwind-devel python3 python3-devel \
 python3-dnspython python3-cryptography \
 python3-matplotlib python3-pexpect python3-pyasn1 \
 tree libuuid-devel wget xfsprogs-devel zlib-devel \
 bind ldns pkg-config vim
```

## 🖥️ Definir nome do host

```bash
echo "fileserver" > /etc/hostname
```

## 🏠 /etc/hosts

```bash
vim /etc/hosts
```

## Contente:

```bash
127.0.0.1      localhost
127.0.1.1      fileserver.educatux.edu fileserver
192.168.70.251 fileserver.educatux.edu fileserver
```

## 🌐 Configurar IP estático

## 👉 Usaremos o método Void padrão, /etc/dhcpcd.conf

```bash
vim /etc/dhcpcd.conf
```

## Adicione IP, gateway (Roteador) e DNS (AD):

```bash
interface eth0
static ip_address=192.168.70.251/24
static routers=192.168.70.254
static domain_name_servers=192.168.70.253
```

## Reinicie a interface de rede:

```bash
sv restart dhcpcd
```

## 🧭 Definir endereço DNS - Aponte para o PDC

```bash
vim /etc/resolv.conf
```

## Contente:

```bash
domain educatux.edu
search educatux.edu
nameserver 192.168.70.253
```

## Bloquear resolv.conf

```bash
chattr +i /etc/resolv.conf
```

## 🔍 Valide o endereço da interface de rede

```bash
ip -c addr
ip -br link
```

## 📥 Baixe e extraia o código-fonte do Samba4

```bash
wget https://download.samba.org/pub/samba/samba-4.23.4.tar.gz
```

```bash
tar -xvzf samba-4.23.4.tar.gz
```

## Compilar e instalar a partir do código-fonte

```bash
cd samba-4.23.4
```

```bash
./configure --prefix=/opt/samba
```

```bash
make -j$(nproc) && make install
```

## Notas:

- Void não interfere porque o Samba é compilado em /opt/samba.

- make -j acelera bastante a compilação - ainda assim, vá tomar um café.

- Após a instalação, o Samba4 compilado não possui nenhum serviço runit.

- Criaremos os serviços manualmente.

## 📁 Adicione Samba4 ao PATH do sistema e recarregue o ambiente

```bash
echo 'export PATH=/opt/samba/bin:/opt/samba/sbin:$PATH' >> /etc/profile
```

```bash
source /etc/profile
```

## Teste a inserção do Samba4 PATH no sistema operacional

```bash
samba-tool -V
```

## Saída:

```bash
4.23.4
```

## ⚠️ Aviso: NÃO use o comando de provisionamento no Servidor de Arquivos!

## 📝 Crie o arquivo smb.conf

```bash
vim /opt/samba/etc/smb.conf
```

```bash
[global]
   workgroup = EDUCATUX
   security = ads
   realm = EDUCATUX.EDU
   netbios name = fileserver
   encrypt passwords = yes
   # point to the services, the active interfaces
   interfaces = eth0
   bind interfaces only = yes

   log file = /opt/samba/var/log.%m
   max log size = 50

   winbind use default domain = yes
   winbind enum users = yes
   winbind enum groups = yes
   winbind refresh tickets = yes

   idmap config * : backend = tdb
   idmap config * : range = 3000-7999
   idmap config EDUCATUX : backend = rid
   idmap config EDUCATUX : range = 10000-999999

   template shell = /bin/bash
   template homedir = /home/%U

[Public]
   path = /srv/samba/public
   browsable = yes
   writable = yes
``` 

## Crie o arquivo de log

```bash
mkdir /opt/samba/var
```

## 📂 Crie o caminho de compartilhamento

```bash
sudo mkdir -p /srv/samba/arquivos
sudo chown -R root:"Domain Admins" /srv/samba/arquivos
sudo chmod -R 0770 /srv/samba/arquivos
```

## Recarregar configuração do Samba4

```bash
smbcontrol all reload-config
```

## 🕒 Servidor NTP / Chrony

## O controlador de domínio deve ser o servidor de horário local, pois com um desvio de 5 minutos o Kerberos não autenticará mais os clientes.

## Instale o Chrony

```bash
xbps-install -Syu chrony
```

## Edite a configuração e permita a rede interna

```bash
vim /etc/chrony.conf
```

## Defina o controle de domínio nos servidores de horário:

```bash
# Comment the external line
#pool pool.ntp.org iburst

# PDC Time Servers
server 192.168.70.253 iburst
```

## Habilite o chronyd no runit

```bash
ln -sf /etc/sv/chronyd/ /var/service/
```

## Reinicie o serviço:

```bash
sv restart chronyd
```

## Validar servidores:

```bash
chronyc sources -v
```

## 🔐 Crie o arquivo Kerberos - Aponte para o PDC

```bash
vim /etc/krb5.conf
```

## contendo

```bash
[libdefaults]
    default_realm = EDUCATUX.EDU
    dns_lookup_realm = true
    dns_lookup_kdc = true
    rdns = false
    forwardable = true
    proxiable = true

[realms]
    EDUCATUX.EDU = {
        kdc = 192.168.70.253
        admin_server = 192.168.70.253
        default_domain = educatux.edu
    }

[domain_realm]
    .educatux.edu = EDUCATUX.EDU
    educatux.edu = EDUCATUX.EDU
```

## Teste Kerberos

```bash
kinit Administrator
```

```bash
klist
```

## Resultado obtido

```bash
Ticket cache: FILE:/tmp/krb5cc_0
Default principal: administrator@EDUCATUX.EDU

Valid starting       Expires              Service principal
11/12/2025 09:43:40  11/12/2025 19:43:40  krbtgt/EDUCATUX.EDU@EDUCATUX.EDU
	renew until 12/12/2025 09:43:36
```

## 🔗 Vincule bibliotecas Winbind ao sistema

## Valide o caminho do libdir:

```bash
/opt/samba/sbin/smbd -b | grep LIBDIR
```

## Esperado:

```bash
LIBDIR: /opt/samba/lib
```

## Crie links de biblioteca (prefira digitar manualmente):

```bash
ln -sf /opt/samba/lib/libnss_winbind.so.2 /usr/lib/
```

```bash
ln -sf /usr/lib/libnss_winbind.so.2 /usr/lib/libnss_winbind.so
```

## Recarregue o cache da biblioteca:

```bash
ldconfig
```

## Atualize o nsswitch para troca de tickets Kerberos (adicione winbind):

```bash
vim /etc/nsswitch.conf
```

```bash
passwd: files winbind
group:  files winbind
```

## Deixe o resto intocado

## Junte-se ao domínio

```bash
net ads join -U Administrator
```

## Resultado obtido

```bash
Password for [EDUCATUX\Administrator]:
Using short domain name -- EDUCATUX
Joined 'VOIDFILES' to dns domain 'educatux.edu'
```

## 📦 Crie os serviços RUNIT para smbd, winbindd e opcionalmente nmbd

### Void Linux usa runit como sistema init, e seu logger nativo é svlogd, incluído no pacote base runit. Nenhum pacote adicional é necessário.

## SMBD — Serviço e registro

## Crie diretórios de serviço e log

```bash
mkdir -p /etc/sv/smbd/log
mkdir -p /var/log/smbd
```

## Crie /etc/sv/smbd/run

```bash
vim /etc/sv/smbd/run
```

## Contente

```bash
#!/bin/sh
exec 2>&1
exec /opt/samba/sbin/smbd --foreground --no-process-group
```

## Permissão

```bash
chmod +x /etc/sv/smbd/run
```

## Crie /etc/sv/smbd/log/run

```bash
vim /etc/sv/smbd/log/run
```

## Contente

```bash
#!/bin/sh
exec svlogd -tt /var/log/smbd
```

## Permissão

```bash
chmod +x /etc/sv/smbd/log/run
```

## Depurar (opcional)

```bash
/opt/samba/sbin/smbd -i
```

## Resultado obtido

```bash
smbd version 4.23.4 started.
Copyright Andrew Tridgell and the Samba Team 1992-2025
daemon 'smbd' : Starting process ...
```

## WINBINDD — Serviço e registro

## Crie diretórios de serviço e log

```bash
mkdir -p /etc/sv/winbindd/log
mkdir -p /var/log/winbindd
```

## Crie /etc/sv/winbindd/run

```bash
vim /etc/sv/winbindd/run
```

## Contente

```bash
#!/bin/sh
exec 2>&1
exec /opt/samba/sbin/winbindd --foreground --no-process-group
```

## Permissão

```bash
chmod +x /etc/sv/winbindd/run
```

## Crie /etc/sv/winbindd/log/run

```bash
vim /etc/sv/winbindd/log/run
```

## Contente

```bash
#!/bin/sh
exec 2>&1
exec /opt/samba/sbin/winbindd --foreground --no-process-group
```

## Permissão

```bash
chmod +x /etc/sv/winbindd/log/run
```

## Depurar (opcional)

```bash
/opt/samba/sbin/winbindd -i
```

## Resultado obtido

```bash
winbindd version 4.23.4 started.
Copyright Andrew Tridgell and the Samba Team 1992-2025
daemon 'winbindd' : Starting process ...
```

## NMBD — Serviço e registro (opcional)

### Ative apenas se o seu ambiente exigir navegação NetBIOS/SMB1.

## Crie diretórios de serviço e log

```bash
mkdir -p /etc/sv/nmbd/log
mkdir -p /var/log/nmbd
```

## Crie /etc/sv/nmbd/run

```bash
vim /etc/sv/nmbd/run
```

## Contente

```bash
#!/bin/sh
exec 2>&1
exec /opt/samba/sbin/nmbd --foreground --no-process-group
```

## Permissão

```bash
chmod +x /etc/sv/nmbd/run
```

## Crie /etc/sv/nmbd/log/run

```bash
vim /etc/sv/nmbd/log/run
```
 
## Contente

```bash
#!/bin/sh
exec svlogd -tt /var/log/nmbd
```

## Permissão

```bash
chmod +x /etc/sv/nmbd/log/run
```

## Habilitar serviços

```bash
ln -sf /etc/sv/smbd /var/service/
ln -sf /etc/sv/winbindd /var/service/
```

## Opcional - habilite somente se estiver usando NetBIOS:

```bash
ln -sf /etc/sv/nmbd /var/service/
```

## Validar serviços

```bash
sv status smbd winbindd nmbd
```

## 🧪 Validar integração

```bash
net ads testjoin
```

## Resultado obtido

```bash
Join is OK
```

```bash
wbinfo -u
```

## Resultado obtido

```bash
guest
krbtgt
administrator```

```bash
wbinfo -g
```

## Result obtained

```bash
controladores de domínio corporativos somente leitura
usuários protegidos
controladores de domínio
convidados do domínio
controladores de domínio somente leitura
administradores de esquema
dnsupdateproxy
administradores de domínio
proprietários do criador da política de grupo
servidores ras e ias
administradores de DNS
grupo de replicação de senha rodc permitido
administradores corporativos
editores certificados
usuários de domínio
grupo de replicação de senha rodc negada
computadores de domínio
```

```bash
wbinfo --ping-dc
```

## Result obtained

```bash
verificação do NETLOGON para domínio [EDUCATUX] conexão dc com "voiddc.educatux.edu" bem-sucedida
```

## ✅ FINAL SUMMARY

## 🎉 Congratulations — you have successfully deployed a fully functional File Server on Void Linux!

## 👉 REMEMBER: While Samba4 can be managed via CLI, it was designed to be managed via RSAT Remote Server Administration Tools, which can be installed on a Windows 10 machine without issues!

---

🎯 THAT'S ALL FOLKS!

👉 Contact: zerolies@disroot.org

👉 https://t.me/z3r0l135
