# refit-blog

Öffentlicher Hugo-Blog für Homelab-Erfahrungsberichte (NAS-Recovery, alte
Hardware reaktivieren etc.), Zielgruppe: andere Homelab-Bastler.

**Live:** https://kant21.spdns.eu/refit-blog/
**Repo:** `dmoench76/refit-blog` (öffentlich, Discussions aktiviert für Giscus)

## Neuen Post anlegen

```bash
hugo new posts/<slug>.md
# Front Matter prüfen: date darf nicht in der Zukunft liegen (Hugo blendet
# sonst den Post standardmäßig aus), draft: false setzen wenn fertig
hugo server -D    # lokale Vorschau: http://localhost:1313/
git add content/posts/<slug>.md
git commit -m "..."
git push
```

Push nach `main` löst automatisch Build + Deploy aus – kein manueller Schritt
nötig. Nach ca. 15–30s ist der Post live unter
`https://kant21.spdns.eu/refit-blog/posts/<slug>/`.

## Architektur (Kurzfassung)

```
git push (main)
    → self-hosted GitHub-Actions-Runner auf dmoench-dell (systemd-Service)
    → hugo --minify (gepinnte Version, siehe deploy.yml)
    → rsync -a --delete public/ /srv/refit-blog/public/   (lokale Kopie, keine SSH-Keys nötig)
    → nginx auf dmoench-dell:8080
    → kant21 nginx (nextcloud.conf) reverse-proxied /refit-blog/ dorthin
```

Der Runner läuft direkt auf dem Deploy-Ziel (dmoench-dell) – "Deploy" ist
deshalb eine lokale Dateikopie, kein Netzwerk-Hop, keine Secrets im Spiel.
Details/Historie inkl. verworfener Ansätze:
`docs/superpowers/plans/2026-07-17-refit-blog.md`.

## Tech-Stack

- **Hugo** (extended), Theme **PaperMod** (Git-Submodule unter `themes/PaperMod`)
- **Giscus** für Kommentare (GitHub Discussions, Category "General") –
  Konfiguration in `hugo.toml` (`[params]` → `giscus*`) und
  `layouts/partials/comments.html`
- `defaultContentLanguage = "de"` – Posts sind auf Deutsch

## Wichtig beim Bearbeiten

- **Nach `git clone`/`git pull` immer `git submodule update --init --recursive`**
  ausführen, sonst fehlt das Theme und der Build schlägt fehl.
- Der lokale `hugo`-Build muss zur Host-Architektur passen (arm64 vs amd64) –
  sonst SIGSEGV unter Emulation. Bei `hugo --minify`/`hugo server`-Abstürzen
  zuerst `file $(command -v hugo)` gegen `uname -m` prüfen.
- CI-Workflow: `.github/workflows/deploy.yml`, Runner-Label `dmoench-dell`.
  Läuft nur bei `push` auf `main` – kein `pull_request`-Trigger (bewusst,
  wegen Self-hosted-Runner-Sicherheit bei Fork-PRs, hier ohnehin
  Solo-Repo).
