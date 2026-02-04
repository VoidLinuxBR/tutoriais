# Distrobox no VoidLinuxBR

Container para utilizar todas as distribuições Linux no VoidLinuxBR

## Objetivo
Instalar o pacote `voidbr-distrobox`, que não está no repositório oficial do Void Linux, e usar o Distrobox no VoidLinuxBR sem a preocupação de quebrar dependências dos seus pacotes favoritos.

ADAPTE este tutorial ao SEU gosto pessoal!

> Observação: A instalação padrão do Void Linux não será coberta aqui.

---

## Importante — repositório chililinux
Como o pacote `distrobox` não consta no repositório oficial do Void Linux, é necessário adicionar o repositório chililinux. Execute os comandos abaixo no terminal:

```bash
sudo sh -c "{
  echo 'repository=https://repo-fastly.voidlinux.org/current'
  echo 'repository=https://void.chililinux.com/voidlinux/current'
} > /etc/xbps.d/00-repository-main.conf"
sudo xbps-install -Syu xbps
sudo xbps-install -Syu libssh2
```

---

## Atualize o sistema após instalar o VoidLinuxBR
Sempre mantenha o sistema atualizado:

```bash
sudo xbps -Suy
sudo xbps -Sy xtools
xcheckrestart
```

---

## Pacotes necessários para o voidbr-distrobox
Para que o `voidbr-distrobox` funcione, instale os pacotes abaixo:

```bash
sudo xbps-install -Sy voidbr-distrobox podman docker crun
```

> Observação: após instalar `crun`, é necessário reiniciar o sistema:
>
 ```bash
 sudo reboot
 ```

---

## Compatibilidade das distribuições
Todas as distribuições disponíveis estão listadas no site oficial do Distrobox. Verifique se a distro que você deseja usar é compatível:
- https://distrobox.it/compatibility/#containers-distros

---

## Criando um container Debian (exemplo)
Escolhemos Debian como exemplo por ser uma das maiores distribuições ativas.

```bash
distrobox create -Y --name debian --image docker.io/library/debian:testing
```

Legenda:
- `distrobox create` — comando base para criar o container;
- `-Y` — executa sem pedir interação ao usuário;
- `--name` — nome do container (usado em outros comandos, ex.: remoção);
- `--image` — imagem a ser usada (Docker Hub, Quay, etc).

Outras flags estão disponíveis. Consulte `distrobox --help` ou a documentação oficial.

---

## Atualizar todos os containers no Distrobox (executar no terminal do VoidLinuxBR)

```bash
distrobox-upgrade --all -v
```

---

## Acessando o container Debian (após o pull)
Para entrar no container:

```bash
distrobox enter debian
```

Dentro do container (exemplo Debian):

```bash
sudo apt update
sudo apt upgrade
sudo apt autoremove
sudo apt install firefox
```

---

## Lista de containers criados no Distrobox

```bash
distrobox list
```

Este comando mostra informações como ID, NAME, STATUS e IMAGE.

---

## Gerenciamento das distribuições no Distrobox
Existem duas formas de gerenciar uma distro no Distrobox:

Opção 1: Entrando no próprio container:
```bash
distrobox enter <nome-do-container>
# e então rodar comandos normalmente dentro do container
```

Opção 2: Executando comandos do host que sejam encaminhados ao container:
Exemplo — instalar o Firefox no container `debian` a partir do VoidLinuxBR:

```bash
distrobox enter debian -- sudo apt install -y firefox-esr-l10n-pt-br
```

---

## Exportação de aplicações para o menu do VoidLinuxBR
O Distrobox permite exportar aplicações do container para que sejam executadas como se fossem nativas do sistema.

Exemplo — exportar o Firefox instalado no container `debian`:

```bash
distrobox enter debian -- distrobox-export --app firefox
```

Após a exportação, você deve encontrar o atalho/aplicação no menu do seu ambiente gráfico (depende do desktop).

---

## Exclusão do container (Distrobox / Podman)
O Distrobox usa Podman (ou Docker) como motor de contêineres. Ao criar um container com Distrobox, também é criado um container no Podman. Para excluir totalmente:

```bash
distrobox rm debian
# em seguida, remover a imagem no podman (caso queira)
podman rmi -f [IMAGE ID]
```

---

## Parar um container no Distrobox
Se precisar parar o container antes de removê-lo:

```bash
distrobox stop debian
```

---

## Observações finais
- Adapte este tutorial conforme suas preferências (nomes de containers, imagens, flags).
- Sempre verifique a documentação oficial do Distrobox: https://distrobox.it
- Teste em uma máquina ou VM antes de aplicar em sistemas de produção.

Bom uso do Distrobox no VoidLinuxBR!
