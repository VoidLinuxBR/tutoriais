# Tutorial Distrobox no Void Linux

O Distrobox permite usar outras distribuições Linux dentro do teu Void Linux
sem comprometer o sistema base (host).  
A lógica é a de sempre: **sistema limpo, testes isolados, zero gambiarra**.

---

## O que é a proposta aqui

Instalar o distrobox no Void Linux, e usar containers para rodar outras distribuições com segurança.

Isso elimina o risco de:
- quebrar dependências do sistema
- poluir o host com pacotes de outras distros
- transformar o Void em Frankenstein

Obs: este guia parte do pressuposto que o Void Linux já está instalado.

---

## Antes de tudo: repositório chililinux

O pacote `distrobox` não existe nos repositórios oficiais do Void.
Mas foi empacotado pelo Comunidade VoidLinuxBR, por isso, é necessário adicionar o repositório chililinux (um mirror oficial do Void no Brasil).

Execute **exatamente** os comandos abaixo:
```bash
sudo sh -c "{
  echo 'repository=https://repo-fastly.voidlinux.org/current'
  echo 'repository=https://void.chililinux.com/voidlinux/current'
} > /etc/xbps.d/00-repository-main.conf"
```

---

## Atualizando o sistema base

Antes de instalar qualquer coisa, deixe o sistema em dia:

```bash
sudo xbps-install -Syu xbps
sudo xbps-install -Syu libssh2 xtools
sudo xbps-install -Suy
xcheckrestart
```
Se o `xcheckrestart` indicar reinício, reinicie.

---

## Instalando o Distrobox e dependências

Agora sim, instale os pacotes necessários:

```bash
sudo xbps-install -Syf voidbr-distrobox podman docker crun
```

Importante:
após instalar o `crun`, é obrigatório reiniciar o sistema:

```bash
sudo reboot
```

---

## Sobre compatibilidade de distribuições

Nem toda distro funciona bem em container.
Antes de escolher, consulte a lista oficial:

https://distrobox.it/compatibility/#containers-distros

Isso evita perda de tempo e dor de cabeça.

---

## Criando o primeiro container (Debian)

Como exemplo, será usado Debian Testing.

```bash
distrobox create -Y --name debian --image docker.io/library/debian:testing
```

O que está acontecendo aqui:
- `distrobox create` cria o container
- `-Y` evita perguntas interativas
- `--name` define o nome do container
- `--image` define a imagem base

Para ver todas as opções disponíveis:

```bash
distrobox --help
```

---

## Entrando no container

Após o pull da imagem, entre no container:

```bash
distrobox enter debian
```

Dentro do Debian, o uso é normal:

```bash
sudo apt update
sudo apt upgrade
sudo apt autoremove
sudo apt install firefox
```

Você está literalmente dentro de outra distro.

---

## Executando comandos sem entrar no container

Também é possível executar comandos diretamente a partir do host.

Exemplo: instalar Firefox no Debian sem entrar nele:

```bash
distrobox enter debian -- sudo apt install -y firefox-esr-l10n-pt-br
```

Prático, rápido e tradicional.

---

## Exportando aplicações para o sistema host

O Distrobox permite exportar aplicações do container
para o menu gráfico do VoidLinuxBR.

Exemplo: exportar o Firefox do container Debian:

```bash
distrobox enter debian -- distrobox-export --app firefox
```

O aplicativo aparecerá no menu do ambiente gráfico
como se fosse nativo.

---

## Atualizando todos os containers

Para atualizar todos os containers de uma vez,
execute no host:

```bash
distrobox-upgrade --all -v
```

---

## Listando containers existentes

Para ver todos os containers criados:

```bash
distrobox list
```

São exibidos nome, status e imagem utilizada.

---

## Parando um container

Se precisar apenas parar o container:

```bash
distrobox stop debian
```

---

## Removendo um container

Para remover o container do Distrobox:

```bash
distrobox rm debian
```

Se quiser remover também a imagem do Podman:

```bash
podman rmi -f [IMAGE ID]
```

---

## Observações finais

- Use containers para testes, não para bagunçar o host
- Ajuste nomes e imagens conforme sua necessidade
- Consulte sempre a documentação oficial:
  https://distrobox.it
- Teste primeiro em VM ou máquina de laboratório

Distrobox é ferramenta de quem gosta de controle,
isolamento e sistema bem cuidado.

---

## 📜 Créditos

Criado por: Robson Nakane <theblizzard1983@hotmail.com>

Comunidade: Void Linux Brasil  
https://github.com/voidlinuxbr

Distribuição: Chili Linux  
https://chililinux.com

Distribuição: VoidBR  
https://github.com/voidlinuxbr

---

## ⚖️ Disclaimer (Aviso Legal)

ESTE SOFTWARE/TUTORIAL É FORNECIDO "COMO ESTÁ", SEM ABSOLUTAMENTE NENHUMA GARANTIA
DE QUALQUER TIPO, EXPRESSA OU IMPLÍCITA, INCLUINDO, MAS NÃO SE LIMITANDO A,
GARANTIAS DE COMERCIALIZAÇÃO OU ADEQUAÇÃO A UM PROPÓSITO ESPECÍFICO.

O USO DESTE SOFTWARE É DE TOTAL RESPONSABILIDADE DO USUÁRIO.

EM NENHUM MOMENTO O AUTOR OU OS CONTRIBUIDORES SERÃO RESPONSÁVEIS POR
QUALQUER DANO, PERDA DE DADOS OU FALHAS NO SISTEMA DECORRENTES DO USO
DESTE PROGRAMA.

---

Copyright (C) 2026 Robson Nakane
