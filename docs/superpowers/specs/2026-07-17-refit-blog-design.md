# refit-blog: Erfahrungsberichte-Blog für Homelab-Projekte

## Zweck

Öffentlicher Blog für Erfahrungsberichte aus Homelab-/Hardware-Projekten (NAS-Recovery,
alte Hardware reaktivieren etc.), Zielgruppe: andere Homelab-Bastler. Beiträge werden als
Markdown in Git geschrieben, versioniert veröffentlicht, mit Kommentarfunktion über Giscus.

## Architektur

```
[dmoench-dell]                          [kant21]
Hugo-Site (statisch gebaut)    <---     nginx reverse proxy
nginx (liefert /refit-blog/)            location /refit-blog/ {
                                            proxy_pass http://dmoench-dell:PORT/;
                                         }
       ^
       | rsync/ssh (deploy)
       |
[GitHub Actions]  <---  git push  <---  [lokal: Markdown schreiben]
  - Hugo Build
  - rsync --delete zu dmoench-dell
```

**Ablauf:** Markdown-Post schreiben → committen → Push zu GitHub → GitHub Actions baut die
Seite mit Hugo → deployt das Ergebnis per SSH/rsync nach dmoench-dell → kant21 leitet
Anfragen unter `/refit-blog/` an dmoench-dell weiter.

dmoench-dell ist dauerhaft eingeschaltet und im lokalen Netz erreichbar (perspektivisch
Ersatz für kant21 als primärer Server) – daher direkter Live-Reverse-Proxy statt
Datei-Kopie zu kant21.

## URL

`https://kant21.spdns.eu/refit-blog/` – nutzt das bestehende, bereits gültige
Let's-Encrypt-Zertifikat für `kant21.spdns.eu` (kein neues Zertifikat nötig).

## Komponenten

### Static Site Generator: Hugo
Go-Binary, schnelle Builds, große Theme-Auswahl, Standard-Wahl für technische Blogs mit
Markdown-Workflow und CI-Integration.

### Theme: PaperMod
Sauberes, schnelles, aktiv gepflegtes Theme für technische/persönliche Blogs. Eingebunden
als Git-Submodule im Hugo-Projekt.

### Kommentare: Giscus
Benötigt ein öffentliches GitHub-Repo mit aktivierten Discussions (dasselbe Repo wie der
Blog-Quellcode). Einrichtung über giscus.app liefert ein Script-Snippet, das ins Theme
eingebunden wird.

### CI/CD: GitHub Actions
Trigger: Push auf `main`. Schritte:
1. Hugo installieren (offizielle `peaceiris/actions-hugo`-Action o.ä.)
2. `hugo --minify` bauen
3. `rsync -avz --delete public/ user@dmoench-dell:/pfad/zu/refit-blog/` per SSH
   (SSH-Key als GitHub Actions Secret hinterlegt, kein Passwort im Klartext)

### Webserver dmoench-dell
Schlankes nginx (oder Caddy), liefert die gebauten statischen Dateien lokal aus,
erreichbar für kant21 im internen Netz.

### kant21 nginx
Neuer `location /refit-blog/`-Block in der bestehenden `kant21-ssl.conf`, proxied zu
dmoench-dell.

## Fehlerbehandlung

- **Build-Fehler:** Schlägt `hugo build` fehl, bricht die Action vor dem Deploy ab – die
  Live-Seite bleibt unverändert.
- **Deploy-Fehler:** Ist dmoench-dell nicht erreichbar, schlägt der rsync-Schritt fehl,
  GitHub Actions meldet das (rot + E-Mail-Benachrichtigung).
- **Proxy-Ausfallsicherheit:** Ist dmoench-dell zur Anfragezeit nicht erreichbar, liefert
  kant21 für `/refit-blog/` einen 502, ohne andere Dienste auf kant21 zu beeinträchtigen
  (Standard-nginx-Verhalten, kein Sonderfall nötig).

## Testen

Lokale Vorschau vor dem Push: `hugo server -D` (Standard-Hugo-Workflow, kein
zusätzliches Tooling nötig).

## Offene Punkte für die Umsetzung (kein Blocker für dieses Spec)

- GitHub-Account/Repo-Name für den Blog-Quellcode (öffentlich, wegen Giscus)
- Genauer Zielpfad und Port für den nginx auf dmoench-dell
- SSH-Key-Erzeugung/Hinterlegung als GitHub Actions Secret
