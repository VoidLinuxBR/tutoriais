# 🔥   Tutorial de instalação do Void Linux base

# Antes de começar

Este tutorial descreve uma **instalação manual do Void Linux**, utilizando particionamento direto de disco, `chroot` e configuração explícita do sistema.  
Ele **não é um instalador automático**.

## ⚠️ Leia com atenção

- Este guia **pressupõe familiaridade com Linux**, terminal e conceitos básicos de sistemas (discos, partições, boot, serviços).
- Vários comandos **apagam dados permanentemente** (`parted`, `mkfs`, `umount -R`).
- Um erro ao definir o disco (`/dev/sdX`, `/dev/nvmeX`) pode resultar em **perda total de dados**.
- Leia **todo o tutorial antes de executar qualquer comando**.

## 🖥️ Ambiente recomendado

- **VM (VirtualBox, QEMU, KVM, etc.)** para testes e aprendizado.
- Hardware dedicado **sem dados importantes**.
- Ambiente de laboratório ou instalação consciente.

❌ **Não recomendado** para uso direto em produção sem adaptações.

## 🔐 Sobre segurança

Durante o processo de instalação, algumas configurações **priorizam praticidade**, não segurança:
- Login do usuário `root` via SSH pode ser habilitado temporariamente.
- Autenticação por senha pode estar ativa.
- Compatibilidade legada (ex: `ssh-rsa`) pode ser permitida.

👉 **Essas configurações devem ser revisadas após a instalação**, especialmente em sistemas expostos à rede.

## 🧠 Importante saber

- Execute os comandos **um a um**, conferindo a saída.
- Ajuste nomes de discos, interfaces de rede e usuários conforme o seu sistema.
- **Não copie e cole cegamente**.
- Em caso de dúvida, **pare** e revise o passo atual.

## 🛠️ Em caso de erro

Se algo der errado:
- Não reinicie às cegas.
- Refaça a montagem das partições.
- Entre novamente no sistema com `chroot`.
- Verifique GRUB, EFI e `initramfs`.

Errar faz parte. Entender o erro é o que separa usuário de operador.

---

> Este guia é voltado a usuários que preferem **controle total** sobre a instalação, seguindo a abordagem clássica Unix:  
> **entender → configurar → validar → continuar**.

## Iniciar a Instalação
Inicie pelo ISO do Void Linux (x86_64 glibc ou musl).

1. Entre como root
```bash
login    : root
password : voidlinux
```
2. Troque o shell de *sh* para o *bash*.  
O *dash/sh* **NÃO implementa** vários recursos que muitos scripts usam.
```bash
bash
```
3. Troque o layout de teclado para **ABNT2**, garantindo o mapeamento correto de acentos e símbolos:
```bash
loadkeys br-abnt2
```

4. Ative o login do usuário **root** via SSH.  
Isso é necessário para acessar a **VM a partir do host** e continuar a instalação remotamente; depois disso, os comandos poderão ser colados/executados diretamente no terminal via SSH.
```bash
echo 'PermitRootLogin yes' >> /etc/ssh/sshd_config
```
5. Reinicie o serviço ssh
```
sv restart sshd
```
6. Exiba o IP da interface de rede
```
ip -4 route get 1.1.1.1 | awk '{print $7}'
```
>Anote o IP da interface de rede e utilize-o para conectar-se à VM via SSH.

7. (Opcional) Acesse a VM via SSH a partir do host.  
- Este passo aplica-se apenas quando o sistema está sendo executado em uma VM; em caso de boot local (sem VM), a instalação pode prosseguir normalmente pelo terminal local.

- Acesse a VM via SSH a partir do host para continuar a instalação remotamente.  
```bash
sudo ssh <ip-da-vm>
```
> Senha padrão: `voidlinux`

8. Configure um prompt colorido no terminal, exibindo usuário, host, caminho atual e o status do último comando:
```bash
export PS1='\[\e[1;32m\]\u\[\e[1;33m\]@\[\e[1;36m\]\h\[\e[1;31m\]:\w \
$([[ $? -eq 0 ]] && echo -e "\e[1;32m✔" || echo -e "\e[1;31m✘$?") \
\[\e[0m\]\$ '
```
> 📌 Este prompt vale apenas para a sessão atual; para torná-lo permanente, adicione ao `.bashrc`.

## Conectar à Internet
- Para **Wi-Fi** *(se estiver no cabo, pule esta etapa)*:
```bash
wpa_passphrase "NOME_DA_REDE_WIFI" "SENHA_DA_REDE" > wifi.conf
wpa_supplicant -B -i wlan0 -c wifi.conf
dhcpcd wlan0
```
> 📌 **Nota:** `wlan0` pode variar (`wlp2s0`, `wlp0s3`, etc.).  
> Use o comando abaixo para identificar a interface correta:
>
> ```bash
> ip -br a
> ```

Testar a conexão:
```
ping -c3 8.8.8.8
ping -c3 repo-default.voidlinux.org
```
## Instalar pacotes necessários
```bash
xbps-install -Sy xbps parted nano vim zstd xz bash-completion
```

## Particionar o disco
1. Identificar o disco
```bash
fdisk -l | grep -E '^(Disk|Disco) '
```
> Assumiremos para o tutorial `/dev/sda`

2. Ajuste as variáveis abaixo conforme o disco que será utilizado (**IMPORTANTE**):
```bash
# Discos SATA/SCSI (sdX)
export DEVICE=/dev/sda
export DEV_EFI=${DEVICE}2
export DEV_RAIZ=${DEVICE}3
```

> 📌 **Nota:**  
> Para discos **NVMe**, o sufixo da partição muda (`p`):
> ```bash
> export DEVICE=/dev/nvme0n1
> export DEV_EFI=${DEVICE}p2
> export DEV_RAIZ=${DEVICE}p3
> ```

3. Particione o disco usando o **parted** (modo automático).  
Este esquema cria:
- Partição BIOS (bios_grub)
- Partição EFI (ESP)
- Partição raiz (ROOT)
```bash
parted --script "${DEVICE}" -- \
  mklabel gpt \
  mkpart primary 1MiB 2MiB name 1 BIOS set 1 bios_grub on \
  mkpart primary fat32 2MiB 514MiB name 2 EFI set 2 esp on \
  mkpart primary 514MiB 100% name 3 ROOT \
  align-check optimal 1

parted --script "${DEVICE}" -- print
```

## Formatar partições
```bash
# Formata a partição raiz (ext4)
mkfs.ext4 -F ${DEV_RAIZ}

# Formata a partição EFI (FAT32)
mkfs.fat -F32 -I ${DEV_EFI}
```

## Montar os volumes em `/mnt`
```bash
# Monte a partição raiz
mount ${DEV_RAIZ} /mnt

# Crie os pontos de montagem necessários
mkdir -p /mnt/{home,boot/efi,var/log,var/cache,dev,proc,sys,run}

# Monte a partição EFI
mount ${DEV_EFI} /mnt/boot/efi
```

## Instalar o sistema base
Instala o sistema base do Void Linux no ambiente montado em `/mnt`, incluindo kernel, firmware, bootloader, rede e ferramentas essenciais.
```bash
xbps-install -Sy -R https://repo-default.voidlinux.org/current \
  -r /mnt \
  base-system e2fsprogs grub-x86_64-efi dracut linux \
  linux-headers linux-firmware linux-firmware-network glibc-locales \
  xtools dhcpcd openssh vim nano grc zstd xz bash-completion vpm vsv \
  socklog-void wget net-tools tmate ncurses
```

> 📌 **Nota:**  
> - `grub-x86_64-efi` → bootloader UEFI  
> - `linux` → kernel  
> - `linux-firmware-network` → drivers de rede  
> - `xtools` → necessário para usar `xgenfstab` sem falhas

## Criar o `fstab`
Gera automaticamente o arquivo de montagem permanente do sistema.
```bash
xgenfstab -U /mnt > /mnt/etc/fstab
```

## Entrar no sistema (chroot)
Acesse o sistema instalado em `/mnt` para continuar a configuração.
```bash
xchroot /mnt /bin/bash
```

## Gerar o INITRAMFS
Detecta automaticamente a versão do kernel instalada e gera o `initramfs` correspondente usando o **dracut**.
```bash
mods=(/usr/lib/modules/*)
KVER=$(basename "${mods[0]}")
echo ${KVER}
dracut --force --kver ${KVER}
```

## Configurar o GRUB

> 📌 Ambos os métodos (BIOS e UEFI) são instalados propositalmente.  
> Isso permite que o mesmo disco inicialize tanto em sistemas **Legacy BIOS** quanto **UEFI**, aumentando a portabilidade entre máquinas.

1. Crie o diretório de suporte ao GRUB:
```bash
mkdir -p /boot/grub
```

2. Instale o GRUB para **BIOS (Legacy)**:
```bash
grub-install --target=i386-pc ${DEVICE}
```

3. Instale o GRUB para **UEFI**:
```bash
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=void
```

4. Crie o fallback UEFI (boot universal).  
Este arquivo garante o boot mesmo se a NVRAM for apagada:
```bash
mkdir -p /boot/efi/EFI/BOOT
cp -f /boot/efi/EFI/void/grubx64.efi /boot/efi/EFI/BOOT/BOOTX64.EFI
```

5. Gere o arquivo final de configuração do GRUB:
```bash
grub-mkconfig -o /boot/grub/grub.cfg
```

## Criar usuário

⚠️ **IMPORTANTE:** defina abaixo o nome do usuário real.
```bash
export NEWUSER=seu_usuario_aqui
```

Crie o usuário com diretório home, grupos básicos e shell Bash:
```bash
useradd -m -G audio,video,wheel,tty -s /bin/bash ${NEWUSER}
passwd ${NEWUSER}
```

## Definir senha do usuário root (IMPORTANTE)
```bash
passwd root
```

## Alterar o shell padrão do usuário root para Bash
```bash
chsh -s /bin/bash root
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
```

## Personalizar `/etc/xbps.d/00-repository-main.conf`  
*(Opcional, mas recomendável)*

Cria o diretório de configuração do **XBPS** (caso ainda não exista) e define uma lista de repositórios oficiais e alternativos.  
Os repositórios **repo-fastly** costumam apresentar melhor latência.

```bash
mkdir -pv /etc/xbps.d

cat << 'EOF' > /etc/xbps.d/00-repository-main.conf
# Repositório oficial (Fastly – melhor latência)
repository=https://repo-fastly.voidlinux.org/current
#repository=https://repo-fastly.voidlinux.org/current/nonfree
#repository=https://repo-fastly.voidlinux.org/current/multilib
#repository=https://repo-fastly.voidlinux.org/current/multilib/nonfree

# Repositório alternativo (Chili Linux)
repository=https://void.chililinux.com/voidlinux/current
#repository=https://void.chililinux.com/voidlinux/current/extras
#repository=https://void.chililinux.com/voidlinux/current/nonfree
#repository=https://void.chililinux.com/voidlinux/current/multilib
#repository=https://void.chililinux.com/voidlinux/current/multilib/nonfree
EOF
```

## Personalizar `/etc/rc.conf`  
Define o fuso horário, o layout do teclado e a fonte padrão do console.  
Altere conforme a necessidade.
```bash
cat << 'EOF' > /etc/rc.conf
TIMEZONE=America/Sao_Paulo
KEYMAP=br-abnt2
FONT=Lat2-Terminus16
EOF
```

## Personalizar o `.bashrc` do usuário
Cria um `.bash_profile` padrão e garante que o `.bashrc` seja carregado automaticamente no login.  
> ⚠️ Certifique-se de que o usuário foi criado no passo anterior.

```bash
# Baixar .bashrc padrão para o /etc/skel
wget --quiet --no-check-certificate \
  -O /etc/skel/.bashrc \
  "https://raw.githubusercontent.com/voidlinux-br/void-installer/refs/heads/main/.bashrc"

chown root:root /etc/skel/.bashrc
chmod 644 /etc/skel/.bashrc

# Criar .bash_profile padrão
cat << 'EOF' > /etc/skel/.bash_profile
# ~/.bash_profile — carrega o .bashrc no Void

# Se o .bashrc existir, carregue
if [ -f ~/.bashrc ]; then
  source ~/.bashrc
fi
EOF

# Copiar para root e usuário
for d in /root "/home/${NEWUSER}"; do
  cp -f /etc/skel/.bash_profile "$d/"
  cp -f /etc/skel/.bashrc "$d/"
done

# Ajustar permissões do usuário
chown "${NEWUSER}:${NEWUSER}" \
  "/home/${NEWUSER}/.bash_profile" \
  "/home/${NEWUSER}/.bashrc"

chmod 644 \
  "/home/${NEWUSER}/.bash_profile" \
  "/home/${NEWUSER}/.bashrc"
```

## Configurar SSH  
*(Opcional, mas recomendável)*

Cria um arquivo de configuração complementar para o **sshd**, mantendo o arquivo principal intacto.
```bash
mkdir -pv /etc/ssh/sshd_config.d

cat << 'EOF' > /etc/ssh/sshd_config.d/10-custom.conf
# Configurações gerais
PermitTTY yes
PrintMotd yes
PrintLastLog yes
Banner /etc/issue.net

# Autenticação
PermitRootLogin yes
PasswordAuthentication yes
KbdInteractiveAuthentication yes
ChallengeResponseAuthentication yes
PubkeyAuthentication yes
PubkeyAcceptedKeyTypes=+ssh-rsa
AuthorizedKeysFile .ssh/authorized_keys
UsePAM yes

# Recursos
X11Forwarding yes
Subsystem sftp internal-sftp
EOF
```
> ⚠️ Recomenda-se revisar e endurecer estas configurações de SSH após o primeiro boot, especialmente em sistemas expostos à Internet.

## Sair do chroot
```bash
exit
```

Desmonte todas as partições montadas em `/mnt` (incluindo subvolumes e `/boot/efi`):
```bash
umount -R /mnt
```

Reinicie a máquina física ou a VM para testar o boot real:
```bash
reboot
```

# 🎉 Enjoy!
O **Void Linux** agora está instalado e pronto para uso.

# DISCLAIMER

> Este tutorial é livre: você pode usar, copiar, modificar e redistribuir como quiser.  
> O conteúdo é disponibilizado sob a **Licença MIT** e pode incluir trechos ou comandos derivados de softwares de código aberto, sujeitos às suas próprias licenças.
>
> Nenhuma garantia é fornecida — tudo aqui é entregue **“no estado em que se encontra”**.  
> Use por sua conta e risco. Nem o autor, nem colaboradores, nem o Void Linux são responsáveis por perdas, danos, falhas de sistema ou qualquer consequência do uso deste material.
>
> Você é livre para revisar, adaptar e gerar sua própria versão deste tutorial.

