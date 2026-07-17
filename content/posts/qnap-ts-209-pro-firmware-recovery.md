---
title: "Ein neues Leben für ein QNAP TS-209 Pro"
date: 2026-07-17T13:00:00+02:00
draft: false
---

Nachbarn haben mir ein QNAP TS-209 Pro überlassen, das nach einem
fehlgeschlagenen Firmware-Update nicht mehr funktionierte. Es hatte schon
eine ganze Weile nur noch in der Ecke gestanden – ein 2-Bay-NAS aus der
Generation 2008, für das sich sonst niemand mehr zuständig fühlte. Die
Teile sind aber ganz brauchbar und viel zu schade zum Wegwerfen, also habe
ich es mitgenommen mit dem Ziel: wieder lauffähig machen. Was auf dem
Papier nach "Firmware neu aufspielen" klang, wurde zu einer Tour durch
tote FTP-Zugänge, falsche Firmware-Formate und eine Emulationssackgasse.
Hier die Kurzfassung für alle, die vor demselben Gerät stehen.

## Ausgangslage

Ein fehlgeschlagenes Firmware-Update hatte das System-Image zerschossen –
das Gerät bootete zwar noch bis zu einem gewissen Punkt, aber ohne
funktionierende Standard-Initialisierung über die Weboberfläche kam man
nicht weiter. Über die Shell ließ sich das NAS manuell mit den
Bordmitteln des Herstellers (`init_single_disk`, `config_util`,
`storage_boot_init`) im Single-Disk-Betrieb initialisieren – ohne RAID,
mit nur einer Platte bestückt.

Damit war das Gerät zwar am Leben, aber ohne aktuelle, funktionierende
QTS-Firmware nicht sinnvoll nutzbar. Also: Firmware besorgen.

## Sackgasse 1: TFTP-Recovery-Modus

Ältere QNAP-Modelle bieten einen TFTP-basierten System-Recovery-Modus
(`192.168.0.10`/`192.168.0.11`, `qnapimg.bin`) für den Fall, dass das
System gar nicht mehr bootet. Für unseren Fall – Gerät läuft, nur die
Firmware fehlt – war das der falsche Weg: TFTP-Recovery erwartet ein
rohes Flash-Image, kein reguläres Firmware-Update-Paket. Genau dieser
Unterschied hat später auch beim eigentlichen Firmware-Tool für Verwirrung
gesorgt (siehe unten).

## Sackgasse 2: Qfinder Pro unter Wine/Box64

QNAPs eigenes Tool zum Aufspielen der Firmware, Qfinder Pro, gibt es nur
für Windows/macOS/x86-Linux – nicht für ARM. Der Versuch, die alte Windows-
Version unter Wine mit Box64/Box32-Emulation auf einem ARM64-System zum
Laufen zu bringen, endete an einer harten Wand: `wineboot.exe` scheiterte
mit `STATUS_INVALID_IMAGE_FORMAT` (`c000007b`), verursacht durch
ungelöste Box32-Symbole (u. a. `arc4random`, `strtold`,
`FT_Get_Transform`, `FcFontSetList`) – ein bekanntes, offenes Problem im
Box64-Projekt, kein Konfigurationsfehler unsererseits.

Bemerkenswerte Randnotiz aus diesem Versuch: Ein direkter
`apt-get install wine64:amd64` auf dem produktiv laufenden Host wurde vom
Paketmanager selbst verweigert (Abhängigkeitskonflikte), bevor irgendetwas
verändert wurde – der anschließende Versuch lief stattdessen isoliert in
einem Docker-Container, um das Live-System (Nginx, Datenbank, weitere
Dienste) nicht zu gefährden. Am Ende halfen weder das noch weitere
Debugging-Versuche – die Entscheidung fiel, stattdessen eine echte
Windows-Maschine zu organisieren.

## Der eigentliche Engpass: eine funktionierende Firmware-Quelle finden

Mit Windows und Qfinder Pro lief das Tool zwar, aber die Firmware selbst
zu bekommen war die eigentliche Herausforderung:

- Die alten, offiziell dokumentierten QNAP-FTP-Zugangsdaten für den
  Recovery-Bereich (`csdread` / `csdread`) sind tot: `530 Login incorrect`.
- Mehrere in Foren verlinkte Mirror-URLs für das reguläre Firmware-Paket
  lieferten nur noch 404 oder 403.
- Die erste gefundene Datei war ein TFTP-Recovery-Image (rohes
  Flash-Image) – Qfinder Pro quittierte das mit: *"Das Format der
  ausgewählten Firmware-Datei stimmt nicht. Bitte überprüfen."* Genau der
  Unterschied aus Sackgasse 1: TFTP-Image ≠ reguläres Firmware-Paket.
- Community-Foren (u. a. qnapclub.de) waren der entscheidende Hinweis,
  brachten aber selbst nicht direkt die Datei.

Am Ende funktionierte ein Community-Mirror mit der QTS-Version **3.3.3**
vom **2014-10-03** – die letzte für das TS-209 Pro veröffentlichte
Firmware-Version.

## Vor dem Aufspielen: Datei-Integrität prüfen

Eine Firmware-Datei aus einer nicht-offiziellen Quelle sollte man nicht
blind vertrauen. Vor der Installation:

- **Hash-Verifikation** – MD5/SHA256 der heruntergeladenen Datei bilden
  und mit dem unten dokumentierten Wert abgleichen, bevor man sie aufs
  NAS spielt.
- **Entropie-Analyse** als Heuristik gegen nachträgliche Manipulation:
  eine unauffällig hohe, gleichmäßige Entropie über die komplette
  Firmware-Datei (typisch für Kompression) spricht gegen nachträglich
  eingefügte, unkomprimierte Schadcode-Blöcke – kein Beweis, aber ein
  sinnvoller erster Check.

## Ergebnis

Mit der richtigen Firmware-Datei (regulärem Paket, nicht dem TFTP-Image)
lief die Initialisierung über Qfinder Pro durch – das TS-209 Pro ist
wieder mit aktueller QTS-Firmware in Betrieb.

## Firmware selbst besorgen – Download

Die Firmware-Datei selbst wird hier bewusst **nicht** zum Download
angeboten – QNAP-Firmware ist proprietäre Herstellersoftware, eine eigene
öffentliche Weiterverbreitung ist rechtlich nicht unser Territorium. Wer
vor demselben Problem steht: über die üblichen QNAP-Community-Foren (z. B.
qnapclub.de) nach *"TS-209 Pro Firmware 3.3.3"* suchen – dort finden sich
i. d. R. aktuelle, funktionierende Mirror-Links.

Falls die Mirror-Suche bei euch ins Leere läuft: Wir haben die Datei noch
und helfen im Zweifel gerne weiter – einfach unten in den Kommentaren
melden.

Zur Verifikation der eigenen heruntergeladenen Datei – das ist exakt die
Version, die bei uns funktioniert hat:

| | |
|---|---|
| **Datei** | `TS-209_20141003-3.3.3.img` |
| **Größe** | 95.424.952 Bytes |
| **MD5** | `47708d56e83056d98fb170b491c4b3cf` |
| **SHA256** | `a916bccb5a04ee2ff07aa7213cf4df0240ffef1dfd2eb0c5ee5e0c9861866509` |

```bash
md5sum TS-209_20141003-3.3.3.img
sha256sum TS-209_20141003-3.3.3.img
```

Stimmen die Hashes überein, ist die Datei identisch zu der, die bei uns
erfolgreich getestet wurde.

## Fazit

Das eigentliche Problem bei so alter Hardware ist selten die Hardware
selbst – es sind die toten Download-Links, die verwaisten FTP-Zugänge und
die Verwechslung zwischen verschiedenen Firmware-Formaten für denselben
Recovery-Zweck. Ein bisschen Foren-Archäologie und ein echtes
Windows-System für die herstellereigenen Tools waren am Ende
zielführender als jeder Emulationsversuch.
