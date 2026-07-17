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

### Task 5: GitHub Actions deploy workflow (self-hosted runner on dmoench-dell)

**Revision note:** the original design in this task called for a GitHub-hosted
runner (`runs-on: ubuntu-latest`) deploying over SSH/rsync using the
rrsync-restricted key from Task 4. That was implemented and pushed (commit
`356ae3f`), but the deploy step failed: GitHub-hosted runners execute in
GitHub's cloud and have no route to `dmoench-dell` (private LAN hostname,
RFC1918 address `192.168.178.108`, no public DNS record). Confirmed via
`dig @1.1.1.1 dmoench-dell` returning empty and the job's own
`ssh-keyscan` failing before ever reaching rsync. The user chose, when
presented with the options, to run a **self-hosted GitHub Actions runner
directly on dmoench-dell** — the runner registers itself with GitHub via
an outbound-only connection (no inbound firewall changes needed), and
since it executes ON the deploy target, the "deploy" step becomes a local
file copy: no SSH key, no secrets, no network hop at all. This makes
Task 4's rrsync-restricted SSH key and the three `DEPLOY_*` secrets
unnecessary — this task's last step retires them.

Self-hosted runners on a **public** repo are a known risk when a workflow
triggers on `pull_request` from forks (a malicious PR could run arbitrary
code on your runner). This repo's workflow only triggers on `push` to
`main`, and this is a solo repo with no other contributors — that risk
does not apply here, but do not add `pull_request` triggers to this
workflow without reconsidering runner placement.

**Files:**
- Create: `~/refit-blog/.github/workflows/deploy.yml` (rewrite — a prior
  version targeting `ubuntu-latest` was already committed as `356ae3f`;
  this replaces it with a new commit, not an amend)
- Create (on dmoench-dell): `/opt/actions-runner/` (GitHub Actions runner
  install, running as a systemd service under user `dmoench`)

**Interfaces:**
- Consumes: deploy target `/srv/refit-blog/public/` on dmoench-dell
  (already exists from Task 4, owned by `dmoench`)
- Produces: automatic build+deploy on every push to `main`, executed
  entirely on dmoench-dell

- [ ] **Step 1: Get a runner registration token and the latest runner release URL**

```bash
cd ~/refit-blog
REG_TOKEN=$(gh api -X POST repos/dmoench76/refit-blog/actions/runners/registration-token --jq .token)
echo "token acquired: ${REG_TOKEN:0:8}..." # do not print the full token
RUNNER_VERSION=$(gh api repos/actions/runner/releases/latest --jq .tag_name | sed 's/^v//')
echo "latest runner version: $RUNNER_VERSION"
```

Expected: both variables populated (token is short-lived, ~1 hour — use it
promptly in Step 2). Do not log the full `$REG_TOKEN` value anywhere
(report file included) — treat it like a credential, since anyone holding
it can register a runner against this repo until it expires or is used.

- [ ] **Step 2: Install and register the runner on dmoench-dell**

```bash
ssh dmoench-dell "sudo mkdir -p /opt/actions-runner && sudo chown dmoench:dmoench /opt/actions-runner"
ssh dmoench-dell "cd /opt/actions-runner && curl -o actions-runner.tar.gz -L https://github.com/actions/runner/releases/download/v${RUNNER_VERSION}/actions-runner-linux-x64-${RUNNER_VERSION}.tar.gz && tar xzf actions-runner.tar.gz && rm actions-runner.tar.gz"
ssh dmoench-dell "cd /opt/actions-runner && ./config.sh --url https://github.com/dmoench76/refit-blog --token '${REG_TOKEN}' --name dmoench-dell --labels dmoench-dell --work _work --unattended --replace"
```

Expected: `config.sh` prints `√ Connected to GitHub` and
`√ Runner successfully added` / `√ Runner connection is good`. Run as
user `dmoench` (not root) — the runner explicitly refuses to run
`config.sh` as root by default, which is the correct, safer behavior; do
not work around this with `--unattended` root flags.

- [ ] **Step 3: Install the runner as a persistent systemd service**

```bash
ssh dmoench-dell "cd /opt/actions-runner && sudo ./svc.sh install dmoench && sudo ./svc.sh start"
ssh dmoench-dell "sudo ./opt/actions-runner/svc.sh status 2>/dev/null || sudo /opt/actions-runner/svc.sh status"
```

Expected: service status shows `active (running)`. This makes the runner
start automatically on boot and survive this SSH session ending.

- [ ] **Step 4: Verify the runner is registered and online**

```bash
gh api repos/dmoench76/refit-blog/actions/runners --jq '.runners[] | {name, status, labels: [.labels[].name]}'
```

Expected: one entry, `"name": "dmoench-dell"`, `"status": "online"`,
labels including `dmoench-dell`.

- [ ] **Step 5: Rewrite the workflow file to target the self-hosted runner with a local deploy step**

Overwrite `~/refit-blog/.github/workflows/deploy.yml`:

```yaml
name: Build and deploy

on:
  push:
    branches: [main]

jobs:
  build-deploy:
    runs-on: [self-hosted, dmoench-dell]
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

      - name: Deploy (local copy)
        run: rsync -a --delete public/ /srv/refit-blog/public/
```

No secrets, no SSH, no network hop — the runner executes this directly on
dmoench-dell, so "deploy" is just copying files into the directory nginx
already serves.

- [ ] **Step 6: Commit and push**

```bash
cd ~/refit-blog
git add .github/
git commit -m "Switch deploy workflow to self-hosted runner on dmoench-dell"
git push
```

Note: `origin` for this repo was already changed to
`git@github.com:dmoench76/refit-blog.git` (SSH) during the earlier attempt,
because pushing changes under `.github/workflows/` requires `workflow`
scope that the cached `gh` OAuth token over HTTPS did not have. The SSH
remote works and needs no further action — do not switch it back to HTTPS
for this push.

- [ ] **Step 7: Watch the run and verify success**

```bash
gh run watch --exit-status
```

Expected: exits 0, log shows `Deploy (local copy)` step succeeded

- [ ] **Step 8: Verify the deployed content on dmoench-dell**

```bash
curl -s http://192.168.178.108:8080/refit-blog/ | grep -q "refit-blog" && echo OK
```

Expected: `OK`

- [ ] **Step 9: Retire the now-unused SSH deploy key and secrets from the abandoned approach**

The rrsync-restricted key (Task 4) and the three `DEPLOY_*` secrets are no
longer used now that the runner executes locally. Remove them to avoid
leaving unused credentials around:

```bash
gh secret delete DEPLOY_SSH_KEY
gh secret delete DEPLOY_HOST
gh secret delete DEPLOY_USER
ssh dmoench-dell "sed -i '/github-actions-refit-blog-deploy/d' ~/.ssh/authorized_keys"
ssh dmoench-dell "grep -c github-actions-refit-blog-deploy ~/.ssh/authorized_keys || echo 0"
rm -f /tmp/refit-blog-deploy-key.pub
```

Expected: `gh secret list` no longer shows the three `DEPLOY_*` secrets;
the grep count is `0`; the leftover public key file is removed.

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
