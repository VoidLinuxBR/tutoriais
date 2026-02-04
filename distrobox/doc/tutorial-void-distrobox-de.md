# Tutorial Distrobox no Void Linux

Mit Distrobox können Sie andere Linux-Distributionen in Ihrem Void Linux verwenden
ohne das Basissystem (Host) zu gefährden.
Die Logik ist wie immer: **sauberes System, isolierte Tests, keine Problemumgehungen**.

---

## Was ist hier der Vorschlag?

Installieren Sie distrobox auf Void Linux und verwenden Sie Container, um andere Distributionen sicher auszuführen.

Dadurch entfällt das Risiko von:
- Systemabhängigkeiten aufheben
- Verunreinigen Sie den Host mit Paketen aus anderen Distributionen
- Verwandle Void in Frankenstein

Hinweis: In dieser Anleitung wird davon ausgegangen, dass Void Linux bereits installiert ist.

---

## Zunächst einmal: Chililinux-Repository

Das Paket „distrobox“ existiert nicht in den offiziellen Void-Repositories.
Da es jedoch von der VoidLinuxBR-Community gepackt wurde, ist es notwendig, das Chililinux-Repository (offizieller Void-Spiegel in Brasilien – <https://xmirror.voidlinux.org/>).) hinzuzufügen

Führen Sie **genau** die folgenden Befehle aus:
```bash
sudo sh -c "{
  echo 'repository=https://repo-fastly.voidlinux.org/current'
  echo 'repository=https://void.chililinux.com/voidlinux/current'
} > /etc/xbps.d/00-repository-main.conf"
```

---

## Aktualisierung des Basissystems

Stellen Sie vor der Installation sicher, dass Ihr System auf dem neuesten Stand ist:

```bash
sudo xbps-install -Syu xbps
sudo xbps-install -Syu libssh2 xtools
sudo xbps-install -Suy
xcheckrestart
```
Wenn „xcheckrestart“ einen Neustart anzeigt, starten Sie neu.

---

## Distrobox und Abhängigkeiten installieren

Installieren Sie nun die erforderlichen Pakete:

```bash
sudo xbps-install -Syf voidbr-distrobox podman docker crun
```

Wichtig:
Nach der Installation von „crun“ ist ein Neustart des Systems zwingend erforderlich:

```bash
sudo reboot
```

---

## Informationen zur Verteilungskompatibilität

Nicht jede Distribution funktioniert gut in Containern.
Konsultieren Sie vor der Auswahl die offizielle Liste:

https://distrobox.it/compatibility/#containers-distros

Dies vermeidet Zeitverschwendung und Kopfschmerzen.

---

## Erstellen des ersten Containers (Debian)

Als Beispiel wird Debian Testing verwendet.

```bash
distrobox create -Y --name debian --image docker.io/library/debian:testing
```

Was passiert hier:
- „distrobox create“ erstellt den Container
- „-Y“ vermeidet interaktive Fragen
- `--name` definiert den Namen des Containers
- „--image“ definiert das Basisbild

So sehen Sie alle verfügbaren Optionen:

```bash
distrobox --help
```

---

## Betreten des Containers

Geben Sie nach dem Ziehen des Bildes den Container ein:

```bash
distrobox enter debian
```

Innerhalb von Debian ist die Verwendung normal:

```bash
sudo apt update
sudo apt upgrade
sudo apt autoremove
sudo apt install firefox
```

Sie befinden sich buchstäblich in einer anderen Distribution.

---

## Befehle ausführen, ohne den Container zu betreten

Sie können Befehle auch direkt vom Host ausführen.

Beispiel: Firefox unter Debian installieren, ohne darauf einzugehen:

```bash
distrobox enter debian -- sudo apt install -y firefox-esr-l10n-pt-br
```

Praktisch, schnell und traditionell.

---

## Exportieren von Anwendungen auf das Hostsystem

Mit Distrobox können Sie Anwendungen aus dem Container exportieren
zum grafischen Menü von VoidLinuxBR.

Beispiel: Firefox aus dem Debian-Container exportieren:

```bash
distrobox enter debian -- distrobox-export --app firefox
```

Die Anwendung wird im grafischen Umgebungsmenü angezeigt
als wäre er ein Einheimischer.

---

## Alle Container werden aktualisiert

Um alle Container gleichzeitig zu aktualisieren,
keinen Host ausführen:

```bash
distrobox-upgrade --all -v
```

---

## Auflistung vorhandener Container

So sehen Sie alle erstellten Container:

```bash
distrobox list
```

Name, Status und verwendetes Bild werden angezeigt.

---

## Stoppen eines Containers

Wenn Sie nur den Container stoppen müssen:

```bash
distrobox stop debian
```

---

## Entfernen eines Containers

So entfernen Sie den Distrobox-Container:

```bash
distrobox rm debian
```

Wenn Sie auch das Podman-Image entfernen möchten:

```bash
podman rmi -f [IMAGE ID]
```

---

## Abschließende Bemerkungen

- Verwenden Sie Container zum Testen, nicht um den Host zu überladen
- Passen Sie Namen und Bilder nach Bedarf an
- Konsultieren Sie immer die offizielle Dokumentation:
https://distrobox.it
- Testen Sie zuerst auf einer VM oder einem Laborcomputer

Distrobox ist ein Tool für diejenigen, die Kontrolle mögen,
Isolierung und gepflegtes System.

---

## 📜 Credits

Erstellt von: Robson Nakane <theblizzard1983@hotmail.com>
Community: Void Linux Brazil <https://github.com/voidlinuxbr>
Distribution: Chili Linux <https://chililinux.com>
Verteilung: VoidBR <https://github.com/voidlinuxbr>

---

## ⚖️ Haftungsausschluss (Rechtliche Hinweise)

DIESE SOFTWARE/LEHRTUTORIAL WIRD „WIE BESEHEN“ UND ABSOLUT KEINE GEWÄHRLEISTUNG ZUR VERFÜGUNG GESTELLT
JEGLICHER ART, OB AUSDRÜCKLICH ODER STILLSCHWEIGEND, EINSCHLIESSLICH, ABER NICHT BESCHRÄNKT AUF,
GARANTIEN DER MARKTGÄNGIGKEIT ODER EIGNUNG FÜR EINEN BESTIMMTEN ZWECK.

DIE NUTZUNG DIESER SOFTWARE LIEGT IN DER VOLLSTÄNDIGEN VERANTWORTUNG DES BENUTZERS.

ZU KEINEM ZEITPUNKT SIND DER AUTOR ODER MITARBEITER VERANTWORTLICH FÜR
Jegliche Schäden, Datenverluste oder Systemausfälle, die sich aus der Nutzung ergeben
DIESES PROGRAMMS.

---

Copyright (C) 2026 Robson Nakane
