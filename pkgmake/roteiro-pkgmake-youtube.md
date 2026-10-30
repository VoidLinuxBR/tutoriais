# Roteiro — pkgmake: crie seus próprios pacotes no VoidBR (do zero)

> **Público:** iniciantes no Void/VoidBR
> **Duração estimada:** 20 a 30 minutos
> **Formato:** cada cena tem a **🎙️ FALA** (o que você diz) e o **💻 TELA** (o que aparece/digita no terminal).

---

## CENA 0 — Abertura (30 s)

🎙️ **FALA**
> Fala, pessoal! Hoje no canal VoidBR vou mostrar o **pkgmake**, a ferramenta do VoidBR para criar pacotes `.xbps` de um jeito simples, parecido com o `makepkg` do Arch.
> Com ele você pega um programa — do GitHub, de um tarball, ou até de outra distro — e transforma num pacote que instala e remove pelo `xbps`, como qualquer outro.
> Vamos ver: instalação, configuração, exemplos feitos à mão e exemplos importados do Arch, Void, Venom, Alpine, Fedora e Debian.

---

## CENA 1 — O que é o pkgmake (1 min)

🎙️ **FALA**
> A ideia é simples: você escreve (ou importa) um arquivo chamado **PKGFILE**, que é uma receita.
> Nele vão: o nome do pacote, a versão, de onde baixar o código e como compilar e instalar.
> O pkgmake lê essa receita, baixa, compila, empacota, assina e coloca o pacote no seu **repositório local**.

💻 **TELA** (pode mostrar como slide)
```
PKGFILE  ──►  pkgmake  ──►  pacote.xbps  ──►  repositório local  ──►  xbps-install
```

---

## CENA 1.5 — E o xbps-src? (2 min)

🎙️ **FALA**
> Quem já usa Void deve estar pensando: "mas o Void já tem o **xbps-src**!" Tem sim, e ele é a ferramenta oficial usada para montar os pacotes do repositório do Void.
> Só que, para o dia a dia de quem quer empacotar um programa para si, ele é bem mais pesado. Olha como é o caminho com o xbps-src:

💻 **TELA**
```sh
git clone https://github.com/void-linux/void-packages.git   # baixa o repositório inteiro, com milhares de templates
cd void-packages
./xbps-src binary-bootstrap                                  # monta um ambiente de build (masterdir) só para ele
# cria/edita srcpkgs/<pacote>/template
./xbps-src pkg <pacote>                                      # o pacote sai em hostdir/binpkgs
```

🎙️ **FALA**
> E com o pkgmake:

💻 **TELA**
```sh
pkgmake new <pacote>      # cria a pasta com o PKGFILE, em qualquer lugar
cd local-<pacote>
pkgmake -s                # pacote pronto, já no seu repositório local
```

🎙️ **FALA**
> As diferenças principais:

💻 **TELA** (pode mostrar como slide)
| | xbps-src | pkgmake |
|---|---|---|
| Instalação | clonar o void-packages + bootstrap | um pacote: `voidbr-pkgmake` |
| Onde trabalha | dentro da árvore do void-packages | em qualquer pasta |
| Receita | `template` (formato próprio do Void) | `PKGFILE` (parecido com o PKGBUILD do Arch) |
| Importar de outras distros | não | Arch/AUR, Void, Venom, Alpine, Fedora, Debian |
| Repositório local e assinatura | manual | automático (`PKGDEST` + `SIGNPKG`) |
| Build isolado | sim (chroot do masterdir) | opcional, com o `pkgmake-docker` |

🎙️ **FALA**
> Resumindo: para mandar um pacote para o repositório oficial do Void, o caminho é o xbps-src.
> Para empacotar seus programas, testar, ou montar o seu próprio repositório, o pkgmake é muito mais simples e limpo — sem clonar nada gigante e sem bootstrap.

---

## CENA 2 — Instalação (3 min)

### Configurar para utilizar os repositórios da VoidBR (recomendado)

🎙️ **FALA**
> Antes de instalar, um passo importante para quem usa o **Void Linux oficial**: adicionar os repositórios da VoidBR.
> É neles que estão o `voidbr-pkgmake`, o `vinstall` e **centenas de pacotes que o Void oficial não tem**. Por isso a gente recomenda usar os dois juntos: o oficial do Void e o da VoidBR. Quem já usa o VoidBR pode pular esta parte, porque já vem configurado.

💻 **TELA**
```sh
{
  echo 'repository=https://void.voidbr.org/voidlinux/current'
  echo 'repository=https://void.voidbr.org/voidlinux/extra'
  echo 'repository=https://repo-fastly.voidlinux.org/current'
} | sudo tee /etc/xbps.d/00-repository-main.conf

sudo xbps-install -Syu xbps
```

🎙️ **FALA**
> Explicando o que cada coisa faz:
> - As duas primeiras linhas são os repositórios da VoidBR: o **current**, com os pacotes principais, e o **extra**, com os pacotes adicionais da comunidade.
> - A terceira é o repositório **oficial do Void**, então você continua recebendo tudo do Void normalmente.
> - O arquivo se chama `00-repository-main.conf` de propósito: é o mesmo nome do arquivo padrão do Void em `/usr/share/xbps.d/`. Um arquivo com o mesmo nome em `/etc/xbps.d/` **substitui** o padrão, e assim a lista de repositórios fica toda num lugar só.
> - Por fim, o `xbps-install -Syu xbps` sincroniza os repositórios novos e atualiza o próprio xbps primeiro. O xbps sempre pede para ser atualizado antes do resto do sistema.

### Instalando o pkgmake

🎙️ **FALA**
> Agora sim. O jeito recomendado é pelo repositório da VoidBR, porque assim ele atualiza junto com o sistema.

💻 **TELA**
```sh
vinstall -S voidbr-pkgmake
```

> ℹ️ **Info:** também dá para instalar com o `xbps-install`, se preferir:
> `sudo xbps-install -S voidbr-pkgmake`

> ⚠️ **Usando o Void oficial?** Recomendamos adicionar também os repositórios da VoidBR (como mostrado acima): eles têm **centenas de pacotes que o Void oficial não tem**.
> O `vinstall` é uma ferramenta do VoidBR e não vem no Void oficial. Se não for usá-lo, em todo o vídeo, onde aparecer `vinstall`, use o `xbps-install` (e o `xbps-remove` para remover):
>
> | VoidBR | Void oficial |
> |---|---|
> | `vinstall -S pacote` | `sudo xbps-install -S pacote` |
> | `vinstall -R ~/repodir pacote` | `sudo xbps-install -R ~/repodir pacote` |
> | `vinstall -X pacote` | `sudo xbps-remove pacote` |

🎙️ **FALA**
> Se você não usa o repositório do VoidBR, dá para instalar direto do GitHub:

💻 **TELA**
```sh
curl -fsSL https://raw.githubusercontent.com/voidlinuxbr/voidbr-pkgmake/main/install.sh | sudo bash
```

🎙️ **FALA**
> Ou clonando o repositório e usando o Makefile:

💻 **TELA**
```sh
git clone --depth=1 https://github.com/voidlinuxbr/voidbr-pkgmake.git
cd voidbr-pkgmake
sudo make install
```

🎙️ **FALA**
> Os três jeitos instalam a mesma coisa. O instalador e o `make install` fazem o mesmo que o pacote:
> - instalam as dependências que faltarem (docker, shfmt, cmake...);
> - **não apagam** o seu `/etc/pkgmake.conf` se ele já existir (o modelo novo vai para `/etc/pkgmake.conf.new`);
> - colocam o seu usuário no grupo `docker` e perguntam se podem ativar o serviço do docker.
>
> Depois de entrar no grupo `docker`, saia e entre de novo na sessão para valer.

🎙️ **FALA**
> Pronto. Vamos conferir:

💻 **TELA**
```sh
pkgmake --help
```

> 💡 **Dica de gravação:** mostre o help colorido rapidinho e diga que vamos usar as opções principais ao longo do vídeo.

---

## CENA 3 — Configurando o /etc/pkgmake.conf (3 min)

🎙️ **FALA**
> O pkgmake tem um arquivo de configuração: **/etc/pkgmake.conf**. Ele é obrigatório. Vamos abrir:

💻 **TELA**
```sh
sudo nano /etc/pkgmake.conf
```

🎙️ **FALA**
> São poucas opções. Vou explicar uma por uma:

💻 **TELA** (exemplo preenchido)
```sh
# Onde os pacotes .xbps e o índice do repositório vão ficar
PKGDEST=$HOME/repodir

# Seu nome e e-mail (aparece nos pacotes)
PACKAGER="Seu Nome <seu@email.com>"

# Instalar automaticamente as dependências de build? (0 = não, 1 = sim)
SYNCDEPS=0

# Apagar src/ e pkg/ depois do build? (0 = não, 1 = sim)
CLEANUP=0

# Assinar os pacotes? (0 = não, 1 = sim)
SIGNPKG=1

# Chave privada usada para assinar
PRIVKEY=$HOME/.ssh/privkey.pem

# Recriar o índice do repositório do zero? (0 = não, 1 = sim)
REINDEX_FORCE=0

# Mostrar os comandos executados? (0 = não, 1 = sim)
VERBOSE=0
```

🎙️ **FALA**
> O mais importante para começar: **PKGDEST** (onde os pacotes vão parar) e **PACKAGER** (seu nome e e-mail, no formato `Nome <email>`).
> Atenção: se deixar o PACKAGER com o exemplo "John Doe", seus pacotes vão sair com esse nome!

### Gerando a chave de assinatura

🎙️ **FALA**
> Com `SIGNPKG=1`, o pkgmake assina os pacotes. Se a chave ainda não existe, ele mesmo pergunta se pode criar. Mas dá para criar antes:

💻 **TELA**
```sh
pkgmake --gen-key
```

🎙️ **FALA**
> Ele cria a chave no caminho do `PRIVKEY`. Guarde bem essa chave: se trocar, as máquinas que usam seu repositório vão precisar aceitar a chave nova.

### Conferindo a configuração

💻 **TELA**
```sh
pkgmake -p
```

🎙️ **FALA**
> O `-p` mostra a configuração que está valendo. Ótimo para conferir se está tudo certo.

---

## CENA 4 — Exemplo local 1: o primeiro pacote (GNU Hello) (4 min)

🎙️ **FALA**
> Vamos criar nosso primeiro pacote do zero. O clássico: **GNU Hello**, um programa que só imprime "Olá, mundo".
> O `pkgmake new` cria um esqueleto de PKGFILE para a gente:

💻 **TELA**
```sh
pkgmake new hello
cd local-hello
```

🎙️ **FALA**
> Ele cria a pasta `local-hello` com um PKGFILE de **exemplo**. Atenção: esse esqueleto é só um modelo — ele aponta para um repositório que não existe, então **não funciona sem editar**.
> Vou trocar o conteúdo todo pela receita do GNU Hello. Pode copiar e colar este bloco inteiro no terminal:

💻 **TELA**
```bash
cat > PKGFILE <<'EOF'
pkgname=hello
pkgver=2.12.3
pkgrel=1
pkgdesc="Programa GNU que imprime Olá, mundo"
license=('GPL-3.0-or-later')
url="https://www.gnu.org/software/hello/"
source=("https://ftp.gnu.org/gnu/hello/hello-${pkgver}.tar.gz")
md5sums=('SKIP')
makedepends=('gcc' 'make')
depends=()

build() {
  cd "$srcdir/hello-$pkgver"
  ./configure --prefix=/usr
  make
}

package() {
  cd "$srcdir/hello-$pkgver"
  make DESTDIR="$pkgdir" install
}
EOF
```

🎙️ **FALA**
> Explicando:
> - `pkgname`, `pkgver` e `pkgrel` → nome, versão e revisão do pacote.
> - `source` → de onde baixar o código.
> - `makedepends` → o que precisa para **compilar**. `depends` → o que precisa para **rodar**.
> - `build()` → como compilar.
> - `package()` → instala tudo dentro de `$pkgdir`, que vira o conteúdo do pacote.

💻 **TELA**
```sh
pkgmake -s
```

🎙️ **FALA**
> O `-s` instala as dependências de build que faltarem. O pkgmake baixa, extrai, compila, empacota e coloca o pacote no repositório local.
> No final ele mostra como instalar:

💻 **TELA** (saída)
```
==> Install with #1: sudo /usr/bin/xbps-install -R /home/voce/repodir hello
==> Install with #2: vinstall -R /home/voce/repodir hello
```

💻 **TELA**
```sh
vinstall -R ~/repodir hello
hello
```

🎙️ **FALA**
> Olha aí: um pacote de verdade, instalado como qualquer outro. Para remover:

💻 **TELA**
```sh
vinstall -X hello
```

> ℹ️ No Void oficial: `sudo xbps-install -R ~/repodir hello` para instalar e `sudo xbps-remove hello` para remover.

---

## CENA 5 — Exemplo local 2: pacote direto do GitHub (git) (3 min)

🎙️ **FALA**
> Agora um programa que vem direto de um repositório git. Vou usar o **neofetch**, que é um script em bash.

💻 **TELA**
```sh
pkgmake new neofetch
cd local-neofetch
```

🎙️ **FALA**
> De novo, trocando o esqueleto pela receita de verdade:

💻 **TELA**
```bash
cat > PKGFILE <<'EOF'
pkgname=neofetch
pkgver=7.1.0
pkgrel=1
pkgdesc="Mostra informações do sistema no terminal"
license=('MIT')
url="https://github.com/dylanaraps/neofetch"
source=("git+${url}.git#tag=${pkgver}")
md5sums=('SKIP')
depends=('bash')

package() {
  cd "$srcdir/neofetch"
  make PREFIX=/usr DESTDIR="$pkgdir" install
}
EOF
```

🎙️ **FALA**
> O `git+` na frente do endereço diz para o pkgmake **clonar** o repositório.
> Como é só um script, nem precisa de `build()`: só o `package()`.

💻 **TELA**
```sh
pkgmake -i
neofetch
```

🎙️ **FALA**
> O `-i` já **instala** o pacote logo depois do build. Prático!

> 💡 **Dica:** o `#tag=${pkgver}` no final fixa a versão do git. Também dá para usar `#branch=` ou `#commit=`.

---

## CENA 5.5 — Exemplo local 3: empacotando arquivos seus (3 min)

🎙️ **FALA**
> E se o programa for **seu**? Um script, um arquivo de configuração, um tema... Também dá para empacotar direto da pasta, sem baixar nada.
> Vou criar um script `ola` que lê o nome de um arquivo de configuração.

💻 **TELA**
```sh
pkgmake new ola
cd local-ola
```

🎙️ **FALA**
> Primeiro os nossos arquivos: o script e a configuração.

💻 **TELA**
```bash
cat > ola.sh <<'EOF'
#!/bin/bash
source /etc/ola.conf
echo "Olá, $NOME! Bem-vindo ao VoidBR."
EOF

cat > ola.conf <<'EOF'
NOME="pessoal"
EOF
```

🎙️ **FALA**
> Agora o PKGFILE. Repare no `source`: são só os nomes dos arquivos, sem endereço. O pkgmake entende que estão aqui na pasta.

💻 **TELA**
```bash
cat > PKGFILE <<'EOF'
pkgname=ola
pkgver=1.0
pkgrel=1
pkgdesc="Meu primeiro pacote com arquivos locais"
license=('MIT')
source=('ola.sh' 'ola.conf')
md5sums=('SKIP')
depends=('bash')
backup=('/etc/ola.conf')

package() {
  install -Dm755 "$srcdir/ola.sh"   "$pkgdir/usr/bin/ola"
  install -Dm644 "$srcdir/ola.conf" "$pkgdir/etc/ola.conf"
}
EOF
```

🎙️ **FALA**
> O `install -D` copia o arquivo e já cria as pastas. O `755` deixa o script executável.
> E o `backup` é um detalhe legal: ele marca o `/etc/ola.conf` como configuração. Se você editar esse arquivo e depois atualizar o pacote, sua edição **não é perdida**.

💻 **TELA**
```sh
pkgmake -i
ola
```

🎙️ **FALA**
> Agora edite o `/etc/ola.conf`, troque o nome, e rode `ola` de novo. Pronto: seu próprio pacote, com seus arquivos!

---

## CENA 6 — Opções do dia a dia (2 min)

🎙️ **FALA**
> Algumas opções que você vai usar bastante:

💻 **TELA**
| Comando | O que faz |
|---|---|
| `pkgmake` | build simples |
| `pkgmake -s` | instala as dependências de build antes |
| `pkgmake -i` | instala o pacote depois do build |
| `pkgmake -f -i` | força a reinstalação (mesma versão) |
| `pkgmake -v` | mostra a saída completa dos comandos |
| `pkgmake --sign` | assina o pacote |
| `pkgmake -k ~/outro-repo` | gera o pacote em outro diretório |
| `pkgmake -c` | limpa as pastas de trabalho (src/ e pkg/) |
| `pkgmake --reindex-only` | recria o índice do repositório local |

---

## CENA 7 — Importando de outras distros (intro) (1 min)

🎙️ **FALA**
> Agora a parte mais legal: você **não precisa escrever tudo do zero**.
> O pkgmake importa receitas prontas de outras distros e já converte para PKGFILE.
> E mais: ele **traduz os nomes das dependências** para os nomes do Void. Por exemplo, `gtk3` do Arch vira `gtk+3`, e `libx11` vira `libX11`.

💻 **TELA**
```
pkgmake new --from-void      <pacote>    # Void Linux
pkgmake new --from-voidlinuxbr <pacote>  # comunidade VoidLinuxBR
pkgmake new --from-arch      <pacote>    # Arch Linux / AUR
pkgmake new --from-venom     <pacote>    # Venom Linux
pkgmake new --from-alpine    <pacote>    # Alpine Linux
pkgmake new --from-fedora    <pacote>    # Fedora
pkgmake new --from-debian    <pacote>    # Debian
```

🎙️ **FALA**
> ⚠️ Regra de ouro: **todo PKGFILE importado precisa ser revisado** antes do build. O pkgmake avisa isso no final de cada importação.

---

## CENA 8 — Importando do Arch / AUR (3 min)

🎙️ **FALA**
> Vamos começar pelo Arch, que tem a maior coleção de receitas, incluindo o AUR.

💻 **TELA**
```sh
pkgmake new --from-arch figlet
cd arch-figlet
nano PKGFILE
```

🎙️ **FALA**
> Repare nas linhas `# NOTE:` em cima do `depends` e do `makedepends`: ali fica a lista **original** do Arch, e logo abaixo a lista **traduzida** para o Void.
> Se algum nome não existir no Void, o pkgmake mostra um aviso no final, para você corrigir.

💻 **TELA**
```sh
pkgmake -s -i
figlet VoidBR
```

🎙️ **FALA**
> O figlet é pequeno e compila em segundos — ótimo para testar. Ele escreve qualquer texto com letras gigantes no terminal.
> Se o pacote não estiver nos repositórios oficiais do Arch, ele procura no **AUR** automaticamente. Nesse caso a pasta se chama `aur-<pacote>`.

---

## CENA 9 — Importando do Void e do Venom (2 min)

🎙️ **FALA**
> Do próprio Void, você pega o template oficial do `void-packages`:

💻 **TELA**
```sh
pkgmake new --from-void figlet
cd void-figlet
```

🎙️ **FALA**
> E do Venom Linux:

💻 **TELA**
```sh
pkgmake new --from-venom figlet
cd venom-figlet
```

🎙️ **FALA**
> O Venom não separa as dependências de build das de execução. O pkgmake faz isso por você: ferramentas como `cmake` e `ninja` vão para o `makedepends`.

---

## CENA 10 — Importando do Alpine (2 min)

🎙️ **FALA**
> O Alpine é ótimo porque as receitas dele, os APKBUILDs, são bem parecidas com o PKGFILE.

💻 **TELA**
```sh
pkgmake new --from-alpine figlet
cd alpine-figlet
nano PKGFILE
```

🎙️ **FALA**
> Ele procura nos repositórios main, community e testing, baixa os patches e converte os nomes: `ncurses-dev` vira `ncurses-devel`, `py3-algo` vira `python3-algo`.
> No caso do figlet, repare no `source`: o patch do Alpine veio junto e é aplicado no `prepare()`.

---

## CENA 11 — Importando do Fedora (2 min)

🎙️ **FALA**
> Do Fedora o pkgmake lê o arquivo `.spec` e converte as seções de build para shell.
> Um bom exemplo é o **nyancat**, o gatinho do arco-íris no terminal:

💻 **TELA**
```sh
pkgmake new --from-fedora nyancat
cd fedora-nyancat
nano PKGFILE
pkgmake -s -i
nyancat
```

🎙️ **FALA**
> Repare que os patches do Fedora já vieram junto e são aplicados no `prepare()`.
> O que o pkgmake não conseguiu converter fica comentado com `NOTE(revisar)` — é só olhar essas linhas.

> 💡 **Dica:** com o `xlocate` instalado, o pkgmake consegue descobrir pacotes pedidos no formato `pkgconfig(...)`, comuns no Fedora.

---

## CENA 12 — Importando do Debian (2 min)

🎙️ **FALA**
> Do Debian ele pega o pacote fonte mais novo, os patches e as dependências de build:

💻 **TELA**
```sh
pkgmake new --from-debian nano
cd debian-nano
nano PKGFILE
```

🎙️ **FALA**
> Um detalhe: o Debian calcula as dependências de execução na hora do build. Por isso o `depends` costuma vir **vazio** — confira e preencha se precisar.
> Os arquivos `debian/control` e `debian/rules` ficam na pasta `debian/` só para consulta.

---

## CENA 13 — Bônus: build limpo com Docker (2 min)

🎙️ **FALA**
> Quer compilar sem instalar um monte de dependências no seu sistema? Use o **pkgmake-docker**.
> Ele roda o pkgmake dentro de um container Void limpo, usando o mesmo `/etc/pkgmake.conf`, e o pacote sai no seu repositório normal.

💻 **TELA**
```sh
pkgmake-docker --help
pkgmake-docker -s
```

🎙️ **FALA**
> Na primeira vez ele cria a imagem, então demora um pouco. Seu usuário precisa estar no grupo `docker` e o serviço do docker precisa estar rodando — a instalação do pkgmake (pacote, instalador ou `make install`) já cuida disso.

---

## CENA 14 — Encerramento (30 s)

🎙️ **FALA**
> Resumindo:
> 1. Instala o `voidbr-pkgmake`.
> 2. Configura o `/etc/pkgmake.conf` (PKGDEST e PACKAGER!).
> 3. Cria do zero com `pkgmake new`, ou importa com `--from-arch`, `--from-fedora` e companhia.
> 4. Revisa o PKGFILE e roda `pkgmake -s -i`.
>
> Se gostou, deixa o like, se inscreve no canal e entra no grupo do VoidLinuxBR no Telegram. Até a próxima!

---

## Checklist antes de gravar

- [ ] `/etc/pkgmake.conf` com seu **PACKAGER** (não deixar "John Doe")
- [ ] Chave criada (`pkgmake --gen-key`) ou `SIGNPKG=0` para simplificar
- [ ] Nunca rodar o `pkgmake` no esqueleto do `pkgmake new` sem trocar o PKGFILE (ele aponta para um repositório que não existe)
- [ ] Testar antes cada exemplo (hello, neofetch, ola, figlet no Arch/Void/Venom/Alpine, nyancat no Fedora, nano no Debian)
- [ ] Limpar as pastas de teste antes de gravar (`rm -rf local-* arch-* void-* venom-* alpine-* fedora-* debian-*`)
- [ ] Fonte do terminal grande o suficiente para leitura no vídeo