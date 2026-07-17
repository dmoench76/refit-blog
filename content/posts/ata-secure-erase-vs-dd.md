---
title: "ATA Secure Erase auf alten Platten: Warum ich beim nächsten Mal einfach dd nehme"
date: 2026-07-17T18:07:13+02:00
draft: false
---

Eine alte Seagate Barracuda ES.7 mit 750GB sollte ein zweites Leben in
einem NAS bekommen – SMART-Status einwandfrei, keine Reallocated Sectors,
also nichts, was gegen eine Weiterverwendung spricht. Vor dem Umzug sollte
die Platte aber sauber gelöscht werden. Die naheliegende Wahl war ATA
Secure Erase – ein Firmware-Befehl, der eigentlich für genau diesen Zweck
gedacht ist. Am Ende hat mich das mehrere Stunden Warten und zwei Eigenschaften
gelehrt, die ich vorher unterschätzt hatte: keine Fortschrittsanzeige, kein
Abbruch.

## Ausgangslage

Die Platte (Modell `ST3750641NS`) steckte in einem alten Core-2-Duo-Rechner,
der bisher als BOINC-Rechenknecht diente. `smartctl` zeigte einen sauberen
Gesundheitszustand: `SMART overall-health self-assessment test result:
PASSED`, keine Reallocated, Pending oder Uncorrectable Sectors. Für die
Weiterverwendung im NAS sollte die Platte aber komplett und sicher gelöscht
werden, nicht nur die Partitionstabelle.

## Die Wahl: Secure Erase statt dd

ATA Secure Erase hat gegenüber einem simplen `dd if=/dev/zero`-Durchlauf
einen theoretischen Vorteil: Der Befehl läuft direkt in der Firmware der
Platte, ohne den Umweg über den SATA-Bus für jedes einzelne Datenbyte – bei
modernen SSDs mit "Enhanced Erase" ist das oft in Sekunden erledigt. Also:

```bash
sudo hdparm --user-master u --security-set-pass MeinPasswort /dev/sdb
sudo hdparm --user-master u --security-erase MeinPasswort /dev/sdb
```

Ein Blick vorher auf die Fähigkeiten der Platte zeigte allerdings schon das
erste Warnsignal:

```
Security:
        Master password revision code = 65534
                supported
        not     enabled
        not     locked
        not     frozen
        not     expired: security count
        not     supported: enhanced erase
```

`enhanced erase` – die schnelle Variante – wird nicht unterstützt. Bei einer
Platte aus der ATA-7-Generation (Baujahr grob 2008–2012) keine Überraschung.
Es blieb also nur der "normale" Secure Erase, und der ist bei rotierenden
Platten ohne Enhanced-Support im Kern nichts anderes als ein vollständiger
Oberflächen-Durchlauf – nur eben von der Firmware selbst gesteuert statt vom
Host.

## Das eigentliche Problem: keine Fortschrittsanzeige, kein Abbruch

`SECURITY ERASE UNIT` ist ein einzelner, blockierender ATA-Befehl. Sobald der
Kernel ihn abgeschickt hat, übernimmt die Firmware der Platte komplett – der
Host bekommt schlicht keine Rückmeldung mehr, bis der Befehl fertig ist.
Kein Fortschritt in Prozent, keine Restzeit-Schätzung (manche Platten melden
sowas über die IDENTIFY-DEVICE-Daten, unsere nicht), nichts. Der einzige
Anhaltspunkt: der `hdparm`-Prozess hängt im Kernel-Zustand `D`
(uninterruptible sleep) – das bestätigt nur "läuft noch", nicht "wie weit".

Nach der ursprünglich veranschlagten Zeit (grob 1,5–3 Stunden für 750GB bei
altem SATA-Link) lief der Befehl weiter. Und weiter. Nach über fünf Stunden
kam die naheliegende Frage auf: einfach abbrechen?

Die Antwort ist ernüchternd: nein, nicht wirklich. `SECURITY ERASE UNIT` läuft
in der Firmware der Platte, nicht im `hdparm`-Prozess auf dem Host. Den
Prozess zu killen oder die SSH-Verbindung zu trennen stoppt die Platte nicht
– sie arbeitet intern weiter, der Host verliert nur die Rückmeldung. Der
ATA-Standard kennt für diesen Befehl schlicht keinen Abbruch. Der einzige
Weg, ihn wirklich zu unterbrechen, wäre ein harter Stromausfall – mit dem
Risiko, dass die Platte danach im gesperrten Security-Zustand hängen bleibt
(mit dem gesetzten Passwort zwar theoretisch wieder entsperrbar, aber ein
zusätzlicher Rettungsschritt) und der komplette bisherige Fortschritt
ohnehin futsch ist.

Ein Blick ins Kernel-Log (`dmesg`) zeigte zumindest keine ATA-Fehler oder
Timeouts am betroffenen Port – der Befehl lief also nicht in einer
Fehlerschleife fest, sondern machte tatsächlich (langsam) Fortschritt. Das
war zwar beruhigend, half aber nichts gegen die Ungewissheit.

## Fazit: nächstes Mal dd

Für SSDs mit funktionierendem Enhanced Erase bleibt ATA Secure Erase die
richtige Wahl – Sekunden statt Stunden. Aber bei einer alten Enterprise-Platte
ohne Enhanced-Support würde ich es nicht wieder machen. Der Grund ist nicht
mal in erster Linie die Dauer, sondern die Kombination aus **keiner
Fortschrittsanzeige** und **keiner Abbruchmöglichkeit** – ein mehrstündiger
Blackbox-Vorgang, bei dem man nur warten kann.

```bash
sudo dd if=/dev/zero of=/dev/sdb bs=1M status=progress
```

macht am Ende dasselbe – die komplette Oberfläche wird überschrieben –, aber
mit zwei entscheidenden Unterschieden: `status=progress` zeigt laufend
Durchsatz und geschriebene Menge, und ein `Strg+C` oder `kill` beendet den
Vorgang tatsächlich sofort, ohne dass die Platte in einem undefinierten
Zustand zurückbleibt. Als Nebeneffekt liest man dabei implizit die komplette
Oberfläche einmal komplett durch (SMART-Fehler würden aus den entsprechenden
Zählern sofort sichtbar) – ein kostenloser Gesundheitstest obendrauf.

## Und jetzt?

Hat jemand belastbare Erfahrungswerte, wie lange ATA Secure Erase auf alten
7200-RPM-Enterprise-Platten (750GB–1TB, ohne Enhanced Erase) tatsächlich
braucht? Würde mich interessieren, ob fünf-plus Stunden eher normal oder
schon ein Ausreißer sind.
