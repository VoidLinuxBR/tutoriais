# Installazione di Mkdocs Server su Void Linux

## 🎯 Obiettivo: caricare il server MkDocs, un generatore di siti Web di documentazione statica veloce, semplice e incentrato sul progetto. Trasforma semplici file Markdown in un sito Web di documentazione professionale e completamente navigabile. La configurazione viene eseguita tramite un singolo file YAML (mkdocs.yml) e il contenuto è scritto in Markdown standard. È ideale per creare documentazione tecnica, manuali utente o basi di conoscenza, offrendo un server di sviluppo integrato per la visualizzazione in tempo reale.

---

## Installa le dipendenze di sistema (Python e pipx) tramite XBPS

```bash
sudo xbps-install -S python3 python3-pipx
```

## 🏠 Installa il pacchetto mkdocs nell'ambiente virtuale Python

```bash
pipx install mkdocs
```

## Aggiungi il nuovo percorso al sistema operativo, localmente o globalmente

## Locale

```bash
pipx ensurepath
```

## Globale

```bash
sudo pipx ensurepath --global
```

## La posizione verrà visualizzata nel file .bashrc dell'utente

```bash
# Created by `pipx` on 2025-11-27 14:07:54
export PATH="$PATH:/home/suporte/.local/bin"
```

## Convalidare il nuovo percorso dell'utente al sistema operativo

```bash
source ~/.bashrc
```

## Convalidare l'installazione del pacchetto

```bash
mkdocs --version
```

## Installazione del tema Material nell'ambiente virtuale Python

```bash
pipx inject mkdocs mkdocs-material
```

## L'iniezione installerà il pacchetto del tema in un percorso nascosto, nella casa dell'utente

```bash
/home/suporte/.local/bin/mkdocs
```

## Sequenza di utilizzo dello strumento:

## 1. Crea un nuovo progetto

## 🔧 Per avviare un nuovo progetto di documentazione, vai alla directory in cui desideri creare il progetto ed esegui:

```bash
mkdocs new Void_Artigos
```

## Questo creerà una nuova directory chiamata Void_Artigos con la struttura di base di MkDocs.

## 2. Utilizza il tema del materiale (facoltativo)

## 🧩 Se hai creato un nuovo progetto, modifica il file di configurazione mkdocs.yml all'interno della directory del progetto (Void_Artigos/mkdocs.yml) e aggiungi la configurazione del tema Material:

```bash
site_name: Void Artigos
nav:
    - Home: index.md
    - Sobre: about.md

theme:
  name: material # Adicione esta linha para usar o tema Material
```

## 3. Avviare il server di sviluppo

## Per visualizzare la documentazione localmente mentre la modifichi, vai alla directory del tuo progetto e avvia il server di sviluppo:

```bash
cd void-Artigos
```

```bash
mkdocs serve
```

## Il server si avvierà e potrai accedere alla documentazione nel tuo browser, solitamente all'indirizzo http://127.0.0.1:8000.. MkDocs monitorerà automaticamente le modifiche ai tuoi file e ricaricherà la pagina.

## Per servire la rete interna, fornire l'IP e la porta del server

```bash
mkdocs serve 192.168.70.100:8000
```

## Essere accessibile da qualsiasi browser sulla rete interna

```bash
http://192.168.70.100:8000
```

## 4. Costruisci documentazione statica

## Quando la documentazione è pronta per essere pubblicata, crea i file statici:

```bash
mkdocs build
```

## Questo creerà una directory chiamata site/ contenente tutti i file HTML, CSS e JavaScript necessari per ospitare la tua documentazione su qualsiasi server web. Insomma, essere su Void Linux non cambia il flusso di lavoro di MkDocs, grazie all'utilizzo di pipx che isola di fatto l'applicazione.

---

🎯 E' TUTTO RAGAZZE!

👉 Contatto: zerolies@disroot.org
👉 https://t.me/z3r0l135
