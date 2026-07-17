# refit-blog Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Public Hugo blog for homelab experience reports, built on push via GitHub Actions, deployed to dmoench-dell, reverse-proxied through kant21 at `/refit-blog/`.

**Architecture:** Markdown posts in a public GitHub repo (`dmoench76/refit-blog`) → GitHub Actions builds with Hugo + PaperMod theme on push to `main` → rsync over SSH deploys the built `public/` directory to dmoench-dell → dmoench-dell's existing per-project nginx snippet pattern (port 8080) serves the static files → kant21's existing nextcloud.conf (the public kant21.spdns.eu server block) gets a new `location ^~ /refit-blog/` proxying to dmoench-dell.

**Tech Stack:** Hugo (extended, latest), PaperMod theme (git submodule), Giscus (GitHub Discussions comments), GitHub Actions, rsync/ssh, nginx.

## Global Constraints

- Public URL: `https://kant21.spdns.eu/refit-blog/` (existing cert, no new cert needed)
- GitHub account: `dmoench76` (confirmed via `gh api user`)
- dmoench-dell LAN IP: `192.168.178.108`, existing project-serving nginx listens on port `8080`
- dmoench-dell nginx snippets live in `/etc/nginx/snippets/*.conf`, auto-included by `/etc/nginx/sites-enabled/dell.conf`
- kant21's public HTTPS server block (kant21.spdns.eu) lives in `/etc/nginx/sites-enabled/nextcloud.conf`, NOT kant21-ssl.conf (that file is the LAN-only `kant21.fritz.box` admin vhost — do not touch it for this project)
- Existing proxy pattern to follow exactly (from nextcloud.conf's `/webmon/` and `/battery/` entries):
  ```nginx
  location ^~ /refit-blog/ {
      proxy_pass http://192.168.178.108:8080/refit-blog/;
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-Proto $scheme;
  }
  ```
- dmoench-dell snippet pattern to follow exactly (from `/etc/nginx/snippets/battery.conf`):
  ```nginx
  location ^~ /refit-blog/ {
      alias /srv/refit-blog/public/;
      index index.html;
  }
  ```

---

### Task 1: Scaffold Hugo site with PaperMod theme

**Files:**
- Create: `~/refit-blog/hugo.toml`
- Create: `~/refit-blog/.gitmodules` (via `git submodule add`)
- Create: `~/refit-blog/themes/PaperMod/` (submodule)
- Create: `~/refit-blog/content/posts/willkommen.md`
- Create: `~/refit-blog/.gitignore`

**Interfaces:**
- Produces: a working local Hugo site buildable with `hugo --minify`, output in `public/`

- [ ] **Step 1: Install Hugo if not present**

Run: `command -v hugo || (curl -sL https://github.com/gohugoio/hugo/releases/latest/download/hugo_extended_$(curl -s https://api.github.com/repos/gohugoio/hugo/releases/latest | grep -oP '"tag_name": "v\K[0-9.]+')_linux-amd64.deb -o /tmp/hugo.deb && sudo dpkg -i /tmp/hugo.deb)`

Expected: `hugo version` prints a version string containing `extended`

- [ ] **Step 2: Initialize Hugo site structure**

```bash
cd ~/refit-blog
hugo new site . --force
```

Expected: creates `archetypes/`, `content/`, `layouts/`, `static/`, `data/`, `hugo.toml` (or `config.toml`, rename to `hugo.toml` if needed)

- [ ] **Step 3: Add PaperMod as a git submodule**

```bash
cd ~/refit-blog
git submodule add --depth=1 https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
git submodule update --init --recursive
```

Expected: `themes/PaperMod/theme.toml` exists

- [ ] **Step 4: Configure hugo.toml**

Replace the contents of `~/refit-blog/hugo.toml` with:

```toml
baseURL = "https://kant21.spdns.eu/refit-blog/"
languageCode = "de-de"
title = "refit-blog"
theme = "PaperMod"
canonifyURLs = true

[params]
  env = "production"
  description = "Erfahrungsberichte aus dem Homelab: alte Hardware reaktivieren, NAS-Recovery und mehr."
  ShowReadingTime = true
  ShowPostNavLinks = true
  ShowShareButtons = false
  ShowBreadCrumbs = true
  ShowCodeCopyButtons = true

[[menu.main]]
  name = "Archiv"
  url = "/posts/"
  weight = 10
```

- [ ] **Step 5: Write a first post**

Create `~/refit-blog/content/posts/willkommen.md`:

```markdown
---
title: "Willkommen"
date: 2026-07-17T12:00:00+02:00
draft: false
---

Erster Beitrag auf refit-blog – Erfahrungsberichte aus Homelab-Projekten:
alte Hardware reaktivieren, NAS-Recovery, und was dabei schiefgehen kann.
```

- [ ] **Step 6: Write .gitignore**

Create `~/refit-blog/.gitignore`:

```
public/
resources/_gen/
.hugo_build.lock
```

- [ ] **Step 7: Build and verify locally**

```bash
cd ~/refit-blog
hugo --minify
```

Expected: `Total in N ms` output, no errors, `public/index.html` exists and contains `refit-blog`

Run: `grep -q "refit-blog" public/index.html && echo OK`
Expected: `OK`

- [ ] **Step 8: Commit**

```bash
cd ~/refit-blog
git add hugo.toml .gitmodules content/ .gitignore
git commit -m "Scaffold Hugo site with PaperMod theme"
```

---

### Task 2: Create GitHub repo and push

**Files:**
- None created locally; remote GitHub repo `dmoench76/refit-blog` created

**Interfaces:**
- Consumes: local `~/refit-blog` git repo from Task 1
- Produces: public GitHub repo with `main` branch pushed, Discussions enabled (required by Task 3's Giscus setup)

- [ ] **Step 1: Create the public GitHub repository**

```bash
cd ~/refit-blog
gh repo create dmoench76/refit-blog --public --source=. --remote=origin --description "Erfahrungsberichte aus dem Homelab"
```

Expected: prints `https://github.com/dmoench76/refit-blog`

- [ ] **Step 2: Enable Discussions on the repo**

```bash
gh api -X PATCH repos/dmoench76/refit-blog -f has_discussions=true
```

Expected: no error output (empty JSON response body is fine)

- [ ] **Step 3: Push**

```bash
cd ~/refit-blog
git push -u origin main
```

Expected: `main -> main` with `[new branch]`, no errors

- [ ] **Step 4: Verify**

```bash
gh repo view dmoench76/refit-blog --json hasDiscussionsEnabled --jq .hasDiscussionsEnabled
```

Expected: `true`

---

### Task 3: Configure Giscus comments

**Files:**
- Create: `~/refit-blog/layouts/partials/extend_head.html` (or theme-supported comment hook)
- Modify: `~/refit-blog/hugo.toml`

**Interfaces:**
- Consumes: public repo with Discussions from Task 2
- Produces: comment widget rendered under each post

- [ ] **Step 1: Get the Giscus config values**

Run: `gh api repos/dmoench76/refit-blog --jq '.id'` → note as `REPO_ID`
Run: `gh api repos/dmoench76/refit-blog/discussions --jq '.[0]' 2>&1 || echo "no discussions yet, category will be created via giscus.app UI"`

Visit `https://giscus.app`, enter `dmoench76/refit-blog`, select mapping "pathname", pick or create category "General" (or "Announcements"), copy the generated `data-repo-id` and `data-category-id` values.

- [ ] **Step 2: Add giscus params to hugo.toml**

Add to the `[params]` section of `~/refit-blog/hugo.toml`:

```toml
  giscusRepo = "dmoench76/refit-blog"
  giscusRepoID = "PASTE_REPO_ID_HERE"
  giscusCategory = "General"
  giscusCategoryID = "PASTE_CATEGORY_ID_HERE"
```

(PaperMod does not ship giscus support natively as of this writing — Step 3 wires it in manually via a partial override.)

- [ ] **Step 3: Create the giscus partial override**

Create `~/refit-blog/layouts/partials/comments.html`:

```html
<script src="https://giscus.app/client.js"
    data-repo="{{ .Site.Params.giscusRepo }}"
    data-repo-id="{{ .Site.Params.giscusRepoID }}"
    data-category="{{ .Site.Params.giscusCategory }}"
    data-category-id="{{ .Site.Params.giscusCategoryID }}"
    data-mapping="pathname"
    data-strict="0"
    data-reactions-enabled="1"
    data-emit-metadata="0"
    data-input-position="bottom"
    data-theme="preferred_color_scheme"
    data-lang="de"
    crossorigin="anonymous"
    async>
</script>
```

PaperMod's `single.html` layout already calls `partial "comments.html" .` when `.Site.Params.comments` is `true` — enable it by adding this to `hugo.toml`'s `[params]` section:

```toml
  comments = true
```

- [ ] **Step 4: Build and verify the widget markup is present**

```bash
cd ~/refit-blog
hugo --minify
grep -q "giscus.app/client.js" public/posts/willkommen/index.html && echo OK
```

Expected: `OK`

- [ ] **Step 5: Commit**

```bash
cd ~/refit-blog
git add hugo.toml layouts/
git commit -m "Wire up giscus comments"
git push
```

---

### Task 4: Deploy target on dmoench-dell

**Files (on dmoench-dell):**
- Create: `/srv/refit-blog/public/` (deploy target directory)
- Create: `/etc/nginx/snippets/refit-blog.conf`
- Modify: `~deploy-user/.ssh/authorized_keys` (or dedicated deploy user)

**Interfaces:**
- Produces: `http://192.168.178.108:8080/refit-blog/` serving whatever is rsynced into `/srv/refit-blog/public/`, and an SSH key GitHub Actions can use to write there

- [ ] **Step 1: Create the deploy directory**

```bash
ssh dmoench-dell "sudo mkdir -p /srv/refit-blog/public && sudo chown dmoench:dmoench /srv/refit-blog/public"
```

Expected: no errors

- [ ] **Step 2: Generate a dedicated deploy SSH keypair (no passphrase, scoped to this one task)**

```bash
ssh-keygen -t ed25519 -f /tmp/refit-blog-deploy-key -N "" -C "github-actions-refit-blog-deploy"
```

Expected: creates `/tmp/refit-blog-deploy-key` and `/tmp/refit-blog-deploy-key.pub`

- [ ] **Step 3: Install the public key on dmoench-dell, restricted via rrsync**

Hardcoding the exact `rsync --server` flag string in a forced command is fragile — it's generated by the rsync client and differs across rsync versions, so a version mismatch between the GitHub Actions runner and dmoench-dell would break deploys. Use `rrsync` (restricted rsync, ships with the rsync package) instead — it validates and constrains the path without depending on exact flags.

```bash
ssh dmoench-dell "dpkg -L rsync | grep -q rrsync || sudo apt-get install -y rsync"
RRSYNC_PATH=$(ssh dmoench-dell "dpkg -L rsync | grep 'bin/rrsync$' || find /usr -name rrsync 2>/dev/null | head -1")
echo "rrsync found at: $RRSYNC_PATH"
PUBKEY=$(cat /tmp/refit-blog-deploy-key.pub)
ssh dmoench-dell "mkdir -p ~/.ssh && echo 'command=\"$RRSYNC_PATH -wo /srv/refit-blog/public/\",no-port-forwarding,no-X11-forwarding,no-agent-forwarding,no-pty $PUBKEY' >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

Expected: `$RRSYNC_PATH` prints a path such as `/usr/bin/rrsync`; the ssh command produces no errors. `-wo` restricts rrsync to write-only access inside `/srv/refit-blog/public/` — it cannot read arbitrary paths or open a shell.

- [ ] **Step 4: Verify the key works and is restricted**

```bash
ssh -i /tmp/refit-blog-deploy-key dmoench-dell "whoami" 2>&1
```

Expected: an error/rejection (NOT `dmoench`) since the `command=` restriction overrides any command the client requests — this confirms the restriction is active. Then verify rsync itself works. Use a bare `dmoench-dell:` destination with **no path after the colon** — `rrsync -wo /srv/refit-blog/public/` already anchors the session there and re-prepends that directory to any absolute path the client sends, so repeating the full path doubles it up and fails with a nested-mkdir error:

```bash
echo "test" > /tmp/rsync-test.txt
rsync -avz -e "ssh -i /tmp/refit-blog-deploy-key" /tmp/rsync-test.txt dmoench-dell:
ssh dmoench-dell "cat /srv/refit-blog/public/rsync-test.txt"
```

Expected: `test`

- [ ] **Step 5: Create the nginx snippet on dmoench-dell**

```bash
ssh dmoench-dell "sudo tee /etc/nginx/snippets/refit-blog.conf > /dev/null" <<'EOF'
# refit-blog - Homelab-Erfahrungsberichte, statisch via Hugo gebaut

location ^~ /refit-blog/ {
    alias /srv/refit-blog/public/;
    index index.html;
}
EOF
```

- [ ] **Step 6: Test and reload nginx on dmoench-dell**

```bash
ssh dmoench-dell "sudo nginx -t && sudo systemctl reload nginx"
```

Expected: `syntax is ok` / `test is successful`, no reload errors

- [ ] **Step 7: Verify locally on dmoench-dell's network**

```bash
curl -s http://192.168.178.108:8080/refit-blog/rsync-test.txt
```

Expected: `test`

- [ ] **Step 8: Clean up the test file**

```bash
ssh dmoench-dell "rm /srv/refit-blog/public/rsync-test.txt"
rm /tmp/rsync-test.txt
```

---

### Task 5: GitHub Actions deploy workflow

**Files:**
- Create: `~/refit-blog/.github/workflows/deploy.yml`

**Interfaces:**
- Consumes: deploy SSH key from Task 4 Step 2, deploy target `/srv/refit-blog/public/` on dmoench-dell
- Produces: automatic build+deploy on every push to `main`

- [ ] **Step 1: Register the deploy key and host info as GitHub Actions secrets**

```bash
cd ~/refit-blog
gh secret set DEPLOY_SSH_KEY < /tmp/refit-blog-deploy-key
gh secret set DEPLOY_HOST --body "dmoench-dell"
gh secret set DEPLOY_USER --body "dmoench"
```

Expected: `✓ Set Actions secret DEPLOY_SSH_KEY for dmoench76/refit-blog` (x3)

- [ ] **Step 2: Shred the local copy of the private key (it's now only in GitHub Secrets + dmoench-dell's authorized_keys)**

```bash
shred -u /tmp/refit-blog-deploy-key
```

Expected: file is removed. Keep `/tmp/refit-blog-deploy-key.pub` for reference or delete it too — the public key is already installed.

- [ ] **Step 3: Write the workflow file**

The deploy key installed in Task 4 is restricted via `rrsync -wo /srv/refit-blog/public/`, which anchors and re-prepends that directory itself. Task 4's verification found that passing any path after the host (e.g. `dmoench-dell:/srv/refit-blog/public/`) doubles the path and fails with a nested-mkdir error — the correct destination is a **bare `host:` with no path**, relying on the key's built-in restriction to anchor the write. Third-party rsync-deploy GitHub Actions (e.g. `burnett01/rsync-deployments`) require a non-empty `remote_path` input, which would reintroduce this bug — so this step invokes `rsync` directly instead, giving full control over the destination argument.

Create `~/refit-blog/.github/workflows/deploy.yml`:

```yaml
name: Build and deploy

on:
  push:
    branches: [main]

jobs:
  build-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: recursive
          fetch-depth: 0

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v3
        with:
          hugo-version: 'latest'
          extended: true

      - name: Build
        run: hugo --minify

      - name: Deploy via rsync
        env:
          DEPLOY_SSH_KEY: ${{ secrets.DEPLOY_SSH_KEY }}
          DEPLOY_HOST: ${{ secrets.DEPLOY_HOST }}
          DEPLOY_USER: ${{ secrets.DEPLOY_USER }}
        run: |
          install -m 600 -D /dev/null ~/.ssh/deploy_key
          echo "$DEPLOY_SSH_KEY" > ~/.ssh/deploy_key
          ssh-keyscan -H "$DEPLOY_HOST" >> ~/.ssh/known_hosts 2>/dev/null
          rsync -avz --delete -e "ssh -i ~/.ssh/deploy_key -o StrictHostKeyChecking=yes" \
            public/ "$DEPLOY_USER@$DEPLOY_HOST:"
```

Note the trailing colon with nothing after it on the destination (`"$DEPLOY_USER@$DEPLOY_HOST:"`) — this is required by the rrsync restriction, not an omission. The trailing slash on the source (`public/`) means rsync copies the *contents* of `public/` rather than the directory itself, matching the plain-file layout `/srv/refit-blog/public/` already has.

- [ ] **Step 4: Commit and push to trigger the first automated deploy**

```bash
cd ~/refit-blog
git add .github/
git commit -m "Add GitHub Actions build+deploy workflow"
git push
```

- [ ] **Step 5: Watch the run and verify success**

```bash
gh run watch --exit-status
```

Expected: exits 0, log shows `Deploy via rsync` step succeeded

- [ ] **Step 6: Verify the deployed content on dmoench-dell**

```bash
curl -s http://192.168.178.108:8080/refit-blog/ | grep -q "refit-blog" && echo OK
```

Expected: `OK`

---

### Task 6: kant21 reverse proxy

**Files:**
- Modify: `/etc/nginx/sites-enabled/nextcloud.conf` (the public `kant21.spdns.eu` HTTPS server block — NOT kant21-ssl.conf, which is the LAN-only admin vhost)

**Interfaces:**
- Consumes: `http://192.168.178.108:8080/refit-blog/` from Task 4/5
- Produces: `https://kant21.spdns.eu/refit-blog/` publicly reachable

- [ ] **Step 1: Locate the insertion point**

```bash
grep -n "location \^~ /webmon/" /etc/nginx/sites-enabled/nextcloud.conf
```

Expected: a line number (insert the new block right after this location's closing `}`)

- [ ] **Step 2: Add the proxy location block**

Using the line number from Step 1, insert immediately after the `/webmon/` block's closing brace:

```nginx
    location ^~ /refit-blog/ {
        proxy_pass http://192.168.178.108:8080/refit-blog/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
```

(Use the Edit tool to insert this after the identified line, matching the file's existing indentation.)

- [ ] **Step 3: Test and reload kant21's nginx**

```bash
sudo nginx -t && sudo systemctl reload nginx
```

Expected: `syntax is ok` / `test is successful`, no reload errors

- [ ] **Step 4: Verify end-to-end from the public URL**

```bash
curl -s https://kant21.spdns.eu/refit-blog/ | grep -q "refit-blog" && echo OK
```

Expected: `OK`

- [ ] **Step 5: Verify the comment widget loads on a real post**

```bash
curl -s https://kant21.spdns.eu/refit-blog/posts/willkommen/ | grep -q "giscus.app/client.js" && echo OK
```

Expected: `OK`

- [ ] **Step 6: Commit the nginx config change**

nginx configs on kant21 are not git-tracked in this project's repo (they live in `/etc/nginx/`, outside `~/refit-blog`) — no commit needed here. Note the change in the plan's completion summary instead.

---

## Post-implementation note

Local testing workflow going forward: `cd ~/refit-blog && hugo server -D`, browse `http://localhost:1313/`. New posts: `hugo new posts/<slug>.md`, edit, commit, push — GitHub Actions handles the rest.
