# 🔥   Tutorial de instalação do Void Linux base

## Iniciar a Instalação
Inicie pelo ISO do Void Linux (x86_64 glibc ou musl).

1. Entre como root
```
login    : root
password : voidlinux
```
2. Troque o shell de sh para o bash. O dash/sh NÃO suporta várias coisas que muitos scripts usam.
```
bash
```
3. Troque o layout de teclado para ABNT2
```bash
loadkeys br-abnt2
```

```
echo 'PermitRootLogin Yes' >> /etc/ssh/sshd_config
sv restart sshd
```

4. Cole no terminal (opcional, mas recomendável) — Prompt com cores, usuário@host:caminho e status do último comando (✔/✘). Útil e bonito.
```
export PS1='\[\e[1;32m\]\u\[\e[1;33m\]@\[\e[1;36m\]\h\[\e[1;31m\]:\w \
$([[ $? -eq 0 ]] && echo -e "\e[1;32m✔" || echo -e "\e[1;31m✘$?") \
\[\e[0m\]\$ '
```
# Conectar à Internet
- Para Wi-Fi *(se estiver no cabo, pule esta etapa)*:
```
wpa_passphrase "SSID" "SENHA" > wifi.conf
wpa_supplicant -B -i wlan0 -c wifi.conf
dhcpcd wlan0
```

1. Testar a conexão:
```
ping -c3 8.8.8.8
ping -c3 repo-default.voidlinux.org
```

2. Instale alguns necessários pacotes:
```
xbps-install -Sy xbps parted nano zstd xz bash-completion
```

## Particionar o disco
1. Identificar o disco
```bash
fdisk -l | grep -E '^(Disk|Disco) '
```
> Assumiremos para o tutorial `/dev/sda`

2. Altere abaixo, conforme o disco que será usado (IMPORTANTE):
```
DEVICE=/dev/sda
DEV_UFI=/dev/sda1
DEV_RAIZ=/dev/sda2
```

3. Usando o parted (automatico)
```
parted --script ${DEVICE} -- \
    mklabel gpt \
    mkpart ESP fat32 1MiB 512MiB set 1 esp on name 1 EFI \
    mkpart ROOT btrfs 512MiB 100% name 2 ROOT \
    align-check optimal 1
parted --script ${DEVICE} -- print
```

## Formatar partições

```bash
# Formatar como ext4:
mkfs.ext4 ${DEV_RAIZ}

# Formatar ESP:
mkfs.fat -F32 ${DEV_EFI}
```

## Montar os volumes em /mnt:

```bash
# O volume raiz
mount ${DEV_RAIZ} /mnt/

# Crie os pontos de montagem no chroot
mkdir -p /mnt/{home,boot/efi,var/log,var/cache,dev,proc,sys,run}

# monte EFI:
mount ${DEV_EFI} /mnt/boot/efi
```

## Instalar o sistema base
```
xbps-install -Sy -R https://repo-default.voidlinux.org/current \
   -r /mnt \
   base-system e2fsprogs grub-x86_64-efi dracut linux \
   linux-headers linux-firmware linux-firmware-network glibc-locales \
   xtools dhcpcd openssh vim nano grc zstd xz bash-completion vpm vsv \
   socklog-void wget net-tools tmate ncurses
```

## Isso garante:

- grub-x86_64-efi → bootloader UEFI
- linux → kernel
- linux-firmware-network → drivers de rede
- xtools → obrigatório para usar xgenfstab sem falhas

## Criar fstab
```
xgenfstab -U /mnt > /mnt/etc/fstab
```

## Entrar no sistema (chroot)
1. Entrar no chroot:
```
xchroot /mnt /bin/bash
```
## Configurar GRUB
```bash
# Pegar a UUID da partição sda2:
UUID=$(blkid -s UUID -o value ${DEV_RAIZ})
echo ${UUID}

# Crie o path para suportar o grub
mkdir -p /boot/grub

# Gerar o novo grub.cfg
grub-mkconfig -o /boot/grub/grub.cfg
```

## Instalação do Boot Manager GRUB em UEFI
```bash
# Instale o GRUB
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=void --recheck

# Criar fallback UEFI (boot universal) - Esse arquivo garante boot mesmo quando a NVRAM for apagada.
mkdir -p /boot/efi/EFI/BOOT
cp -vf /boot/efi/EFI/void/grubx64.efi /boot/efi/EFI/BOOT/BOOTX64.EFI
```

## Gerando o INITRAMFS
```
mods=(/usr/lib/modules/*)
KVER=$(basename "${mods[0]}")
echo ${KVER}
dracut --force --kver ${KVER}
```

## Configurações básicas
```bash
# Setar Hostname
echo void > /etc/hostname

# Setar Localtime
ln -sf /usr/share/zoneinfo/America/Sao_Paulo /etc/localtime

# Setar Locales
sed -i 's/#en_US.UTF-8/en_US.UTF-8/' /etc/default/libc-locales
sed -i 's/#pt_BR.UTF-8/pt_BR.UTF-8/' /etc/default/libc-locales

# Gerar locales:
xbps-reconfigure -f glibc-locales

# Corrigir possivel erro no symlink do /var/service (importante):
rm -f /var/service
ln -sf /etc/runit/runsvdir/default /var/service

# Ativar alguns serviços:
ln -sf /etc/sv/dhcpcd /var/service/
ln -sf /etc/sv/sshd /var/service/
ln -sf /etc/sv/nanoklogd /var/service/
ln -sf /etc/sv/socklog-unix /var/service/

# baixar svlogtail customizado (opcional, mas recomendável):
wget --quiet --no-check-certificate -O /usr/bin/svlogtail \
   "https://raw.githubusercontent.com/voidlinux-br/void-installer/refs/heads/main/svlogtail" && \
   chmod +x /usr/bin/svlogtail

# Criar um resolv.conf
printf 'nameserver 1.1.1.1\nnameserver 8.8.8.8\n' > /etc/resolv.conf

#Configurar sudo - grupo wheel (opcional, mas recomendável)
cat << 'EOF' > /etc/sudoers.d/g_wheel
%wheel ALL=(ALL:ALL) NOPASSWD: ALL
EOF
#Permissões obrigatórias
chmod 440 /etc/sudoers.d/g_wheel

# Criar o usuário
NEWUSER=seunomeaqui
useradd -m -G audio,video,wheel,tty -s /bin/bash ${NEWUSER}
passwd ${NEWUSER}
```

## Trocar senha de root (importante):
```bash
passwd root
```

## Personalizar o /etc/xbps.d/00-repository-main.conf (opcional, mas recomendável):
- Cria o diretório de configurações do XBPS (se ainda não existir) e adiciona uma lista de repositórios oficiais e alternativos.
Os repositórios repo-fastly costumam ter melhor latência.
```
mkdir -pv /etc/xbps.d
cat << 'EOF' >> /etc/xbps.d/00-repository-main.conf
repository=https://repo-fastly.voidlinux.org/current
#repository=https://repo-fastly.voidlinux.org/current/nonfree
#repository=https://repo-fastly.voidlinux.org/current/multilib
#repository=https://repo-fastly.voidlinux.org/current/multilib/nonfree

repository=https://void.chililinux.com/voidlinux/current
#repository=https://void.chililinux.com/voidlinux/current/extras
#repository=https://void.chililinux.com/voidlinux/current/nonfree
#repository=https://void.chililinux.com/voidlinux/current/multilib
#repository=https://void.chililinux.com/voidlinux/current/multilib/nonfree
EOF
```

## Personalizar o /etc/rc.conf (opcional, mas recomendável):
- Define o fuso horário, layout do teclado e fonte padrão do console. Altere conforme necessidade.
```
cat << 'EOF' >> /etc/rc.conf
TIMEZONE=America/Sao_Paulo
KEYMAP=br-abnt2
FONT=Lat2-Terminus16
EOF
```

## Personalizar o .bashrc do usuario (opcional, mas recomendável):
Cria um .bash_profile para o usuário e garante que o .bashrc seja carregado automaticamente no login.
> confira se criou o usuário no passo anterior
```
wget --quiet --no-check-certificate \
   -O /etc//skel/.bashrc \
   "https://raw.githubusercontent.com/voidlinux-br/void-installer/refs/heads/main/.bashrc"
chown root:root /etc/skel/.bashrc
chmod 644 /etc/skel/.bashrc
```

```
cat << 'EOF' > /etc/skel/.bash_profile
# ~/.bash_profile — carrega o .bashrc no Void

# Se o .bashrc existir, carregue
if [ -f ~/.bashrc ]; then
  source ~/.bashrc
fi
EOF
```

```
# copia para o root e usuario
for d in /root "/home/${NEWUSER}"; do
   cp -f /etc/skel/.bash_profile "$d/"
   cp -f /etc/skel/.bashrc "$d/"
done

chown "${NEWUSER}:${NEWUSER}" "/home/${NEWUSER}/.bash_profile" "/home/${NEWUSER}/.bashrc"
chmod 644 "/home/${NEWUSER}/.bash_profile" "/home/${NEWUSER}/.bashrc"
```

## configurar ssh (opcional, mas recomendável):
```
mkdir -pv /etc/ssh/sshd_config.d/
cat << 'EOF' > /etc/ssh/sshd_config.d/10-custom.conf
PermitTTY yes
PrintMotd yes
PrintLastLog yes
Banner /etc/issue.net

PermitRootLogin yes
KbdInteractiveAuthentication yes
X11Forwarding yes
PubkeyAuthentication yes
PubkeyAcceptedKeyTypes=+ssh-rsa
AuthorizedKeysFile .ssh/authorized_keys
PasswordAuthentication yes
ChallengeResponseAuthentication yes
UsePAM yes

Subsystem sftp internal-sftp
EOF
```

## Sair do chroot
```
exit
```

```
# Desmonta todas as partições montadas em /mnt (subvolumes e /boot/efi)
umount -R /mnt

# Desativa qualquer swapfile ou swap partition que tenha sido ativada dentro do chroot
swapoff -a
```

```
# Reinicia a máquina física ou a VM para testar o boot real
reboot
```
---

# 🎉   Enjoy!
O Void Linux agora está instalado.

# DISCLAIMER

```
Este tutorial é livre: você pode usar, copiar, modificar e redistribuir como quiser.  
O conteúdo é disponibilizado sob a **Licença MIT**, e pode incluir trechos ou comandos derivados de softwares de código aberto sujeitos às suas próprias licenças.

Nenhuma garantia é fornecida — tudo aqui é entregue “no estado em que se encontra”.  
Use por sua conta e risco. Nem o autor, nem colaboradores, nem o Void Linux são responsáveis por perdas, danos, falhas de sistema ou qualquer consequência do uso deste material.

Se desejar, você pode obter o código-fonte, revisar, adaptar e gerar sua própria versão deste tutorial.
```

