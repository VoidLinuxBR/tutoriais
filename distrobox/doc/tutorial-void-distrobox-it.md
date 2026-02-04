# Tutorial Distrobox senza Void Linux

Distrobox ti consente di utilizzare altre distribuzioni Linux all'interno del tuo Void Linux
senza compromettere il sistema base (host).
La logica è la solita: **sistema pulito, test isolati, zero soluzioni alternative**.

---

## Qual è la proposta qui?

Installa distrobox su Void Linux e utilizza i contenitori per eseguire altre distribuzioni in sicurezza.

Ciò elimina il rischio di:
- interrompere le dipendenze del sistema
- inquinare l'host con pacchetti di altre distribuzioni
- trasforma Void in Frankenstein

Nota: questa guida presuppone che Void Linux sia già installato.

---

## Prima di tutto: repository chililinux

Il pacchetto `distrobox` non esiste nei repository ufficiali di Void.
Ma è stato confezionato dalla comunità VoidLinuxBR, quindi è necessario aggiungere il repository chililinux (mirror ufficiale Void in Brasile - <https://xmirror.voidlinux.org/>).

Esegui **esattamente** i comandi seguenti:
```bash
sudo sh -c "{
  echo 'repository=https://repo-fastly.voidlinux.org/current'
  echo 'repository=https://void.chililinux.com/voidlinux/current'
} > /etc/xbps.d/00-repository-main.conf"
```

---

## Aggiornamento del sistema di base

Prima di installare qualsiasi cosa, assicurati che il tuo sistema sia aggiornato:

```bash
sudo xbps-install -Syu xbps
sudo xbps-install -Syu libssh2 xtools
sudo xbps-install -Suy
xcheckrestart
```
Se "xcheckrestart" indica il riavvio, riavvia.

---

## Installazione di Distrobox e dipendenze

Ora installa i pacchetti necessari:

```bash
sudo xbps-install -Syf voidbr-distrobox podman docker crun
```

Importante:
Dopo aver installato `crun`, è obbligatorio riavviare il sistema:

```bash
sudo reboot
```

---

## Informazioni sulla compatibilità della distribuzione

Non tutte le distribuzioni funzionano bene nei contenitori.
Prima di scegliere consulta il listino ufficiale:

https://distrobox.it/compatibility/#containers-distros

In questo modo si evitano perdite di tempo e mal di testa.

---

## Creazione del primo contenitore (Debian)

Ad esempio verrà utilizzato Debian Testing.

```bash
distrobox create -Y --name debian --image docker.io/library/debian:testing
```

Cosa sta succedendo qui:
- `distrobox create` crea il contenitore
- "-Y" evita le domande interattive
- "--name" definisce il nome del contenitore
- "--image" definisce l'immagine di base

Per vedere tutte le opzioni disponibili:

```bash
distrobox --help
```

---

## Entrando nel contenitore

Dopo aver estratto l'immagine, inserisci il contenitore:

```bash
distrobox enter debian
```

All'interno di Debian, l'utilizzo è normale:

```bash
sudo apt update
sudo apt upgrade
sudo apt autoremove
sudo apt install firefox
```

Sei letteralmente all'interno di un'altra distribuzione.

---

## Eseguire comandi senza entrare nel contenitore

Puoi anche eseguire comandi direttamente dall'host.

Esempio: installare Firefox su Debian senza entrarci:

```bash
distrobox enter debian -- sudo apt install -y firefox-esr-l10n-pt-br
```

Pratico, veloce e tradizionale.

---

## Esportazione delle applicazioni nel sistema host

Distrobox ti consente di esportare applicazioni dal contenitore
al menu grafico di VoidLinuxBR.

Esempio: esporta Firefox dal contenitore Debian:

```bash
distrobox enter debian -- distrobox-export --app firefox
```

L'applicazione apparirà nel menu dell'ambiente grafico
come se fosse nativo.

---

## Aggiornamento di tutti i contenitori

Per aggiornare tutti i contenitori contemporaneamente,
non eseguire nessun host:

```bash
distrobox-upgrade --all -v
```

---

## Elenco dei contenitori esistenti

Per vedere tutti i contenitori creati:

```bash
distrobox list
```

Vengono visualizzati il nome, lo stato e l'immagine utilizzata.

---

## Arresto di un contenitore

Se hai solo bisogno di fermare il contenitore:

```bash
distrobox stop debian
```

---

## Rimozione di un contenitore

Per rimuovere il contenitore Distrobox:

```bash
distrobox rm debian
```

Se vuoi rimuovere anche l'immagine Podman:

```bash
podman rmi -f [IMAGE ID]
```

---

## Osservazioni finali

- Utilizzare i contenitori per i test, non per ingombrare l'host
- Modifica i nomi e le immagini secondo necessità
- Consulta sempre la documentazione ufficiale:
https://distrobox.it
- Testare prima su VM o macchina da laboratorio

Distrobox è uno strumento per coloro che amano il controllo,
isolamento e sistema ben mantenuto.

---

## 📜 Crediti

Creato da: Robson Nakane <theblizzard1983@hotmail.com>
Comunità: Void Linux Brasile <https://github.com/voidlinuxbr>
Distribuzione: Chili Linux <https://chililinux.com>
Distribuzione: VoidBR <https://github.com/voidlinuxbr>

---

## ⚖️ Dichiarazione di non responsabilità (Avviso legale)

QUESTO SOFTWARE/TUTORIAL VIENE FORNITO "COSÌ COM'È" SENZA ASSOLUTAMENTE NESSUNA GARANZIA
DI QUALSIASI TIPO, ESPRESSO O IMPLICITO, INCLUSO, MA NON LIMITATO A,
GARANZIE DI COMMERCIABILITÀ O IDONEITÀ PER UNO SCOPO PARTICOLARE.

L'UTILIZZO DI QUESTO SOFTWARE È INTERA RESPONSABILITÀ DELL'UTENTE.

IN NESSUN MOMENTO L'AUTORE O I CONTRIBUTORI SARANNO RESPONSABILI DI
QUALSIASI DANNO, PERDITA DI DATI O GUASTO DEL SISTEMA DERIVANTE DALL'UTILIZZO
DI QUESTO PROGRAMMA.

---

Copyright (C) 2026 Robson Nakane
