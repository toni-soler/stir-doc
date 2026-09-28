# STIR dev/staging VM setup — from bare VM to ready-for-Claude Code/Codex

Purpose-built for a **dedicated development/staging VM** (working name `dev.stir.es` — used
throughout this document as a placeholder; **the final hostname may differ**, and nothing below
depends on it being exactly this string — substitute your real choice everywhere it appears),
distinct from the `stir.es` production host. It takes you from *VM recién creada* to *STIR completo
desplegado, tests ejecutables y entorno listo para Claude Code/Codex*.

This is not a generic Ubuntu/Docker tutorial. Every command, version and path below was read
directly out of the real repos as of this writing (`pom.xml`, `package.json`, `Dockerfile`s,
`compose.yml`/`compose.production.yml`, `Caddyfile`/`Caddyfile.production`, `application.yml`,
`upstream.lock.json`, `scripts/*.py`, `HOST_PROVISIONING.md`, `DEPLOYMENT.md`, `README.md`,
workspace `CLAUDE.md`). Where a step genuinely depends on which cloud/hypervisor you use, it is
marked `PROVIDER-SPECIFIC STEP` instead of guessed.

**This document does not create the VM, touch DNS, deploy production, change WebAuthn, or modify
any code.** It is documentation only. Sections 0–16 are a dev/staging adaptation of the existing
`stir-main/HOST_PROVISIONING.md` + `stir-main/DEPLOYMENT.md` (which target production); sections
17–21 are new, specific to resuming the in-progress WebAuthn/hardware-custody MVP and to a
multi-agent (Claude Code + Codex) daily workflow.

---

## 0. Architecture objective

```
developer machine (this PC)                    STIR-DEV VM (new, e.g. dev.stir.es)
──────────────────────────────                 ──────────────────────────────────
 SSH client, browser                            sshd
 git (push/pull to GitHub)          SSH ──────▶ 5 STIR repos (git clone, siblings)
                                                 Docker Engine + Compose plugin
                                                   ├─ postgres:17-alpine
                                                   ├─ core/module migrations (one-shot)
                                                   ├─ runtime-provision (one-shot)
                                                   ├─ shell (IDAX Shell)      ─┐
                                                   ├─ stir (STIR backend)     │ edge+data
                                                   ├─ stir-ui (nginx static)  │ networks
                                                   ├─ ostris / ostris-ui      │
                                                   ├─ ledger / ledger-ui      │
                                                   ├─ minio (object storage) ─┘
                                                   └─ proxy (Caddy, reverse proxy + HTTPS)
                                                 Java 21 + Maven, Node 22, Python 3.10+
                                                 (for `mvn test`/`npm test` outside Docker)

external browser (your laptop/phone)
        │ HTTPS (443)
        ▼
  dev.stir.es  ──────────────────────────────▶  Caddy (proxy container, ACME cert)
                                                   ├─ /extensions/stir/*    → stir-ui:80
                                                   ├─ /extensions/ostris/*  → ostris-ui:80
                                                   ├─ /extensions/ledger/*  → ledger-ui:80
                                                   ├─ /api/stir/*           → stir:8096
                                                   ├─ /api/ostris/*         → ostris:8095
                                                   ├─ /api/ledger/*         → ledger:8094
                                                   └─ /  (everything else)  → shell:8080
```

**What runs where:** your PC only ever holds an SSH client, a browser, and (optionally) a git
clone used to push branches to GitHub — no Docker build, no Testcontainers, no Playwright run
happens on your PC anymore. The VM holds the actual git worktrees, runs `mvn`/`npm`/Docker builds,
Testcontainers-backed Postgres tests, and headless-browser E2E. A real, external browser (on your
laptop, not the VM) is the one that visits `https://dev.stir.es` for manual verification —
including a real WebAuthn authenticator, which cannot be driven from inside a headless VM (see
§14).

`postgres`, `minio`, `ostris`, `ledger`, `shell` and `stir` publish **no public port** — everything
except the `proxy` container binds to `127.0.0.1` only (confirmed in `compose.yml`/
`compose.production.yml`). The only two ports ever exposed to the internet are 80/tcp and 443/tcp
(+443/udp for HTTP/3), both owned by Caddy.

---

## 1. Create the VM

Do not assume a specific provider/hypervisor yet — pick sizing based on what STIR's own build
actually costs, not a generic guess.

| | Minimum viable | Recommended | Why |
|---|---|---|---|
| vCPU | 4 | **8** | `docker compose build` compiles 4 Spring Boot/Maven modules (idax-shell, stir, ostris, idax-ledger) plus the frontend/esbuild step concurrently; `HOST_PROVISIONING.md` already documents 4 vCPU as its production floor. 8 gives headroom for a second agent (Claude + Codex) building/testing at the same time, and for Testcontainers-backed `mvn test` runs (each spins up its own disposable `postgres:17-alpine` container). |
| RAM | 8 GB | **32 GB** | Runtime alone fits in ~4 GB (Shell 512 MB heap + STIR/osTRIS/Ledger 384–512 MB heap each + Postgres + MinIO + Caddy + 3 static nginx UIs). The *build* is the real constraint — `HOST_PROVISIONING.md` calls this out explicitly — and this session's own experience on an 8 GB local machine repeatedly hit **OOM kills mid-Docker-build** doing exactly this work, which is the direct reason this VM is being provisioned. 32 GB leaves comfortable room for two agents' `mvn`/Testcontainers/Docker builds running close together without repeating that failure. |
| Disk | 40 GB SSD | **200–250 GB SSD/NVMe** | 40 GB is HOST_PROVISIONING.md's bare production floor. A dev/staging box additionally accumulates: multiple Docker image layers per rebuild, Maven's `~/.m2` cache, `node_modules`, Testcontainers' own Postgres images, Playwright's downloaded Edge binary, possibly two parallel git worktrees per repo (Claude + Codex) × 5 repos, plus `.local/backups/` snapshots. NVMe matters more than raw capacity for Maven/npm/Docker I/O patterns. |
| Swap | none required | **8–16 GB** | Cheap insurance against a build-time memory spike (§3) — never a substitute for the RAM figure above. |
| Timezone | — | set explicitly (e.g. `Europe/Madrid`) | Affects log timestamps, cert renewal scheduling, cron backups (§19). |
| Hostname | — | `stir-dev` | Matches this document's own working assumption; rename freely. |
| Admin user | — | a real non-root sudo user | See §2 — do not operate as `root` day to day. |

`PROVIDER-SPECIFIC STEP`: choosing the actual cloud/hypervisor, image (Ubuntu Server 24.04 LTS —
confirmed the right target: `HOST_PROVISIONING.md` documents Ubuntu 22.04/24.04 as a first-class
alternative to its Rocky 9 primary path, and this document follows the Ubuntu branch throughout),
region, and attaching/allocating the disk are all specific to whichever provider you pick — resolve
these when you actually provision.

---

## 2. First access and basic hardening

Nothing exotic — this mirrors `HOST_PROVISIONING.md` step 1 plus standard first-login hygiene.

```sh
# As whatever initial user the image gives you (often root on a fresh cloud VM):
apt-get update && apt-get -y upgrade
apt-get -y install ca-certificates curl gnupg git ufw

# Create a real admin user if the image dropped you in as root
adduser stir-admin
usermod -aG sudo stir-admin
```

**SSH keys before disabling password auth** — do these in order, verifying each before the next:

```sh
# On YOUR machine, not the VM:
ssh-keygen -t ed25519 -C "stir-dev access"
ssh-copy-id stir-admin@<vm-ip>      # or manually append to ~/.ssh/authorized_keys on the VM

# From your machine, confirm key-based login actually works BEFORE touching sshd_config:
ssh stir-admin@<vm-ip> "echo key login OK"
```

**If the VM sits behind a firewall/router that already forwards port 22 to a different host**
(common when several SSH-capable machines share one public IP), you need a distinct **external**
port forwarded to this VM's **internal** port 22 — this is a `PROVIDER-SPECIFIC STEP` (configured
on the router/firewall, not on the VM). `sshd` itself keeps listening on its normal internal port
22; only the external mapping differs, so §2's `ufw allow OpenSSH` and §21's checklist are
unaffected — nothing on the VM side changes. Pass the external port explicitly with `-p` on the
commands above:

```sh
ssh-keygen -t ed25519 -C "stir-dev access"
ssh-copy-id -p <ext-port> stir-admin@<public-host-or-ip>
ssh -p <ext-port> stir-admin@<public-host-or-ip> "echo key login OK"
```

Every other section in this guide (§3 onward) assumes you are already inside an SSH session opened
this same way — none of them re-invoke `ssh stir-admin@<vm-ip>` from scratch. To avoid retyping
`-p <ext-port>` for this and any future session (including reconnecting later), add an alias in
your own `~/.ssh/config` once the key-based login above is confirmed working, then use
`ssh stir-dev` from here on instead:

```
Host stir-dev
  HostName <public-host-or-ip>
  Port <ext-port>
  User stir-admin
```

Only **after** key-based login is confirmed (via `-p`/the alias, whichever you set up), on the VM:

```sh
sudo sed -i 's/^#\?PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
sudo systemctl restart ssh
```

Firewall — `HOST_PROVISIONING.md` §3 documents that STIR's production overlay publishes only
80/tcp, 443/tcp and 443/udp; everything else is loopback-only. For a dev VM you also need SSH:

```sh
sudo ufw allow OpenSSH
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 443/udp
sudo ufw enable   # only if not already active; confirm OpenSSH rule is in first
sudo ufw status verbose
```

If the VM sits behind a cloud provider's own security group (most managed VMs do —
`PROVIDER-SPECIFIC STEP`), open the same four ports there too; the host firewall alone is not
sufficient.

Timezone and clock:

```sh
sudo timedatectl set-timezone Europe/Madrid   # or your real timezone
timedatectl status                             # confirm "NTP service: active"
```

Do not go further than this — no exotic AppArmor/SELinux profiles, no custom kernel hardening.
Ubuntu ships AppArmor by default and needs no STIR-specific action (unlike Rocky's SELinux case
documented in `HOST_PROVISIONING.md` §6, which does not apply here).

---

## 3. Memory and swap

```sh
free -h                 # confirm total RAM matches what you provisioned
swapon --show            # empty until you add swap below
```

Add swap (identical to `HOST_PROVISIONING.md` §4, sized up for this VM's larger builds):

```sh
sudo fallocate -l 12G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
free -h                 # confirm the Swap line now shows ~12G
swapon --show
```

**Swap is a shock absorber for a transient build-time spike, not a substitute for the RAM figure
in §1.** A VM that relies on swap to stay up during normal `docker compose build` runs is
undersized — go back and resize it rather than tuning swappiness to compensate.

---

## 4. Docker

Install Docker's own packages (not `docker.io`/`podman-docker`), exactly as `HOST_PROVISIONING.md`
§2 documents for Ubuntu — the distro packages lack the Compose v2 plugin STIR's scripts always
invoke as `docker compose` (never the legacy standalone `docker-compose`):

```sh
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get -y install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker

# Run Docker as your non-root admin user, not root:
sudo usermod -aG docker "$(whoami)"
```

**The group change above does not apply to your current shell/SSH session** — `usermod` only
updates `/etc/group`; a session that was already open keeps the group list it started with. Do one
of the two before verifying:

```sh
newgrp docker    # applies the new group to a subshell of THIS session, no reconnect needed
# — or, if you'd rather just reconnect —
exit             # then ssh back in (§2's alias, or -p <ext-port> ... again)
```

Verify:

```sh
docker version
docker compose version
docker run --rm hello-world
```

If you skip the step above, `docker version` still succeeds (it only talks to the client binary),
but `docker compose version` and `docker run` fail with `permission denied while trying to connect
to the docker API at unix:///var/run/docker.sock` — that exact error means "the group change hasn't
taken effect in this session yet," not a real permissions problem; `newgrp docker` or a reconnect
fixes it every time.

BuildKit: Docker Compose v2 + a current `docker-ce` already default to BuildKit for `docker build`/
`docker compose build` — no separate `DOCKER_BUILDKIT=1` export or `buildx` bootstrap is required
for anything STIR's own `compose.yml`/`Dockerfile`s do (they use plain single/multi-stage builds,
no BuildKit-only syntax). No extra configuration needed here.

---

## 5. Development toolchain

Install **only** what the five repos actually declare, verified against real files, not assumed:

| Tool | Real requirement (source) | Verify |
|---|---|---|
| Java | **21** — `stir-backend/Dockerfile` (`maven:3.9.9-eclipse-temurin-21` build stage, `eclipse-temurin:21-jre-alpine` runtime); `pom.xml` `<java.version>21</java.version>` | `java -version` |
| Maven | **3.8+** (the `Dockerfile`'s build stage bundles 3.9.9, but no `mvnw` wrapper exists in `stir-backend` — a real system Maven install is required, and there is no known 3.8/3.9 incompatibility for this build; Ubuntu 24.04's own `apt` package is 3.8.7, confirmed working — only chase an exact 3.9.x install if a real plugin-resolution error actually appears) | `mvn -version` |
| Node | **22** — `stir-frontend/Dockerfile` (`node:22-alpine` build stage) | `node --version` |
| npm | whatever ships with Node 22 | `npm --version` |
| Python | **3.10+** — `stir-main/README.md`'s own stated quick-start requirement; `HOST_PROVISIONING.md` confirms Ubuntu 22.04 ships 3.10+, 24.04 ships 3.12+ by default | `python3 --version` |
| Git | any recent version | `git --version` |
| OpenSSL | required by `scripts/initialize.py` to generate the local JWT RSA keypair (3072-bit) | `openssl version` |

```sh
sudo apt-get -y install openjdk-21-jdk maven git openssl python3 python3-venv python3-pip
```

Node 22 is not in Ubuntu 24.04's default apt repos at a matching version — use NodeSource's setup
script (the standard, documented way to get a current Node major on Debian/Ubuntu):

```sh
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt-get -y install nodejs
node --version   # expect v22.x
```

Verify Java/Maven/Node all resolve to the versions above before continuing — a mismatched Java
minor (e.g. 21 present but not the JDK, or a stray Java 17 earlier on `PATH`) is the most common
"works on my machine" surprise `mvn test` will hit.

There is no Maven or Node wrapper committed in any of the five repos (`mvnw`, `.nvmrc`) — prefer
the system installs above; there is nothing project-local to prefer instead.

---

## 6. Playwright / browser E2E

Verified directly from the real scripts, not assumed:

- `stir-main/requirements-browser.txt` pins exactly `playwright==1.62.0`.
- **Every** existing browser script (`scripts/*_browser*.py`, `scripts/browser-smoke.py`) launches
  with `playwright.chromium.launch(channel='msedge', headless=True)` — real **Microsoft Edge**, not
  Playwright's bundled Chromium. `README.md` states this plainly: *"With Microsoft Edge installed,
  create a local Python virtual environment, install `requirements-browser.txt`..."*.
- The `webauthn_hardware_custody_e2e.py` HTTP script (already written, unverified — see §17) and
  the not-yet-written `webauthn_hardware_custody_browser.py` are the only scripts that will need a
  **virtual WebAuthn authenticator** (distinct A/B below).
- The `cryptography` Python package is imported by several E2E scripts (`market_integrity_e2e.py`,
  `multi_source_value_evidence_e2e.py`, `webauthn_hardware_custody_e2e.py` — Ed25519/EC key
  generation and signing) but **is not listed in any `requirements*.txt` in this repo** — it must be
  installed separately. This is a real, verified gap in the repo's own dependency declarations, not
  an invented requirement; install it explicitly below.

**Part 1 — system-level Edge install. Order-independent; do this whenever, including before §7:**

```sh
# Real Microsoft Edge for Linux (Microsoft's own apt repo - the standard way to get it on Ubuntu):
curl -fsSL https://packages.microsoft.com/keys/microsoft.asc | sudo gpg --dearmor -o /usr/share/keyrings/microsoft.gpg
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/microsoft.gpg] https://packages.microsoft.com/repos/edge stable main" | \
  sudo tee /etc/apt/sources.list.d/microsoft-edge.list
sudo apt-get update
sudo apt-get -y install microsoft-edge-stable
```

**Part 2 — the Python venv, requires `stir-main` already cloned. Do §7 first if you have not yet,
then come back here:**

```sh
cd /srv/stir/stir/stir-main   # or wherever §7 actually put it on this VM
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements-browser.txt
pip install cryptography     # not declared anywhere in this repo - see note above
playwright install-deps      # Ubuntu Server has no GUI libs preinstalled; this installs the
                              # shared libraries (libnss3, libatk, libgbm, fonts, ...) headless
                              # Chromium/Edge needs to actually launch on a bare server
```

Confirm Edge actually launches headless before trusting any browser script:

```sh
python3 -c "
from playwright.sync_api import sync_playwright
with sync_playwright() as p:
    b = p.chromium.launch(channel='msedge', headless=True)
    print('Edge launched OK:', b.version)
    b.close()
"
```

### A. Automated browser E2E inside the VM

This is what every `scripts/*_browser*.py` script does today, and what the not-yet-written
`webauthn_hardware_custody_browser.py` will do (§17): headless Edge, driven by Playwright, against
`http://localhost:8089` (the VM's own local dev stack — no need to go through the public
`https://dev.stir.es` hostname for this). For WebAuthn specifically, use a **virtual authenticator**
via Chrome DevTools Protocol, which Playwright exposes through raw CDP access:

```python
# Sketch for the not-yet-written webauthn_hardware_custody_browser.py (§17) - not yet implemented,
# documented here so the approach is decided before you resume that work:
cdp = page.context.new_cdp_session(page)
cdp.send('WebAuthn.enable')
authenticator = cdp.send('WebAuthn.addVirtualAuthenticator', {
    'options': {
        'protocol': 'ctap2', 'transport': 'internal',
        'hasResidentKey': False, 'hasUserVerification': True,
        'isUserVerified': True, 'automaticPresenceSimulation': True,
    }
})['authenticatorId']
# ... drive navigator.credentials.create()/get() through the page as normal; the virtual
# authenticator answers them without any real hardware or human interaction.
```

This is standard CDP functionality (`WebAuthn.enable`/`addVirtualAuthenticator`), available in
Edge/Chromium and reachable from Playwright 1.62 via `BrowserContext.new_cdp_session()` — no extra
package beyond what §6 already installs.

### B. Real browser on your PC, against `https://dev.stir.es`

For manual verification (including a real hardware authenticator — see §14), you do **not** run
anything on the VM. On your own laptop/desktop, open a real browser (any modern one — this path
has no `channel='msedge'` constraint, that only applies to the automated Python scripts above) and
navigate to `https://dev.stir.es` once §12–13 are done. This is the only path that can exercise a
genuine security key or platform authenticator (Windows Hello, Touch ID, a YubiKey, etc.) — a
headless VM has no such hardware attached and cannot simulate "a human touched a real device."

---

## 7. Git / GitHub

All five repos are real, public GitHub repos under the same owner:

```
https://github.com/toni-soler/stir-workspace.git   (this is "stir-doc"'s sibling parent; clones as "stir")
https://github.com/toni-soler/stir-doc.git
https://github.com/toni-soler/stir-backend.git
https://github.com/toni-soler/stir-frontend.git
https://github.com/toni-soler/stir-main.git
```

SSH access (recommended over HTTPS+PAT for daily agent use — no token to rotate in scripts):

```sh
ssh-keygen -t ed25519 -C "stir-dev VM"
cat ~/.ssh/id_ed25519.pub
# Add this public key at https://github.com/settings/keys (your own GitHub account)
ssh -T git@github.com   # expect "Hi <username>! You've successfully authenticated"
```

Clone as siblings, exactly as `README.md`/`HOST_PROVISIONING.md` §5 already establish — do not
invent a different layout:

```sh
sudo mkdir -p /srv/stir
sudo chown "$(whoami):$(whoami)" /srv/stir
cd /srv/stir
git clone git@github.com:toni-soler/stir-workspace.git stir
cd stir
git clone git@github.com:toni-soler/stir-doc.git
git clone git@github.com:toni-soler/stir-backend.git
git clone git@github.com:toni-soler/stir-frontend.git
git clone git@github.com:toni-soler/stir-main.git
```

Result:

```
/srv/stir/
  stir/                 (stir-workspace: CLAUDE.md, README, .code-workspace)
    stir-doc/
    stir-backend/
    stir-frontend/
    stir-main/
```

**Why `/srv/stir`**: `HOST_PROVISIONING.md` §5 explicitly recommends *"a real working directory...
e.g. `/opt/stir`... do not leave it at `~` of an interactive root shell"*. `/srv` is the FHS-correct
location for "data for services this system provides" on Ubuntu, and keeps the tree out of any one
human user's home directory — appropriate for a box that both you and two agents (Claude Code,
Codex) will operate on. Adjust to `/opt/stir` or `~/source/stir` if you have an existing convention
you'd rather keep; nothing below depends on the exact prefix, only on the five repos being sibling
directories with `stir-main` at the same level as the other three named repos.

Never put a personal access token or SSH private key inside any script in these repos — SSH agent
forwarding or a key generated directly on the VM (as above) are the only credential paths that
belong here.

---

## 8. Initial repo state check

Before any build, confirm every repo is in the state you expect — this is the exact discipline
`stir/CLAUDE.md` already establishes for this workspace ("Local checkouts are shared with Codex...
`git checkout` in one session silently redirects the other's next commands"):

```sh
for r in stir-doc stir-backend stir-frontend stir-main; do
  echo "=== $r ==="
  (cd /srv/stir/stir/$r && git remote -v && git branch --show-current && git status --short && git log --oneline -3)
done
```

Expect: `origin` pointing at the right `toni-soler/<repo>.git`, branch `main` (or whatever this
guide's own checkout should be on — see §17 for the WebAuthn branch specifically), a **clean**
working tree (`git status --short` prints nothing), and `git log` matching what you see on GitHub.

**Contamination-avoidance procedure** (already the standing convention in `stir/CLAUDE.md`, applies
identically here — this is not a new rule, just restated because it matters even more with two
agents on one VM):

```
1. fetch          git fetch origin
2. verify main     git log --oneline origin/main..main   (expect nothing if main is clean/current)
3. branch/worktree  git checkout -b claude/<capability>-mvp   (or a worktree - see §18)
4. build/test      mvn verify / npm test / docker compose build, from that branch
5. ancestry check  git fetch origin && git log --oneline origin/main..HEAD   (only your own commits)
6. merge/push      git checkout main && git merge --ff-only <branch> && git push
```

Step 5 is the one that catches a parallel session's contamination before it merges — never skip it
on a shared VM.

---

## 9. Secrets and configuration

Real inventory, read directly from `scripts/initialize.py`, `compose.yml`,
`compose.production.yml`, and `stir-backend/src/main/resources/application.yml` — not guessed.

| Variable / file | Purpose | DEV source | Secret? | Versionable? |
|---|---|---|---|---|
| `.local/secrets/postgres_password` | PostgreSQL superuser password | generated by `initialize.py` (`secrets.token_urlsafe(36)`) | yes | **no** — `.gitignore`d |
| `.local/secrets/runtime_password` | `idax_backend` app DB role password (NOSUPERUSER, NOBYPASSRLS) | generated by `initialize.py` | yes | **no** |
| `.local/secrets/jwt_private_key` / `jwt_public_key` | RSA-3072 signing keypair for local JWT auth (`idax.auth.local.public-key-location`) | generated by `initialize.py` via `openssl genpkey` | yes (private half) | **no** |
| `.local/secrets/bootstrap_token` | Shell's one-time bootstrap token | generated by `initialize.py` | yes | **no** |
| `.local/secrets/login_password` | The auto-provisioned `admin@stir.test` dev login password | generated by `initialize.py` | yes | **no** |
| `.local/secrets/storage_access_key` / `storage_secret_key` | MinIO (object storage) credentials | generated by `initialize.py` | yes | **no** |
| `.env` (in `stir-main/`) | Compose-level overrides: ports (dev), or `STIR_PUBLIC_HOSTNAME`/`STIR_PUBLIC_BASE_URL`/`STIR_SITE_NAME`/`STIR_TENANT_CODE`/`STIR_ADMIN_EMAIL`/`STIR_SUPPORT_CONTACT` (prod overlay) | you create it (copy `.env.example`, then edit) | some values are identifying, not cryptographic | **no** — `.gitignore`d |
| `.local/production` (empty marker file) | Tells every script (`deploy.py`, `status.py`, `restore.py`, ...) to target the `stir-prod` compose project instead of `stir-dev` | **create only if you deliberately want this VM's stack addressed as if it were the production overlay** (see the HTTPS note in §12) | no (empty file) | n/a |
| `STIR_WEBAUTHN_RP_ID` (env, backend) | WebAuthn Relying Party ID | `application.yml` default `localhost`; **must be `dev.stir.es`** once real HTTPS is live | no | n/a (env var, not a file) |
| `STIR_WEBAUTHN_RP_NAME` (env, backend) | Display name shown by authenticators | default `STIR`; override to distinguish from prod if desired, e.g. `STIR (dev)` | no | n/a |
| `STIR_WEBAUTHN_ALLOWED_ORIGINS` (env, backend) | Comma-separated allowed origins for WebAuthn `clientDataJSON.origin` checks | `application.yml` default `http://localhost:8089`; **must become `https://dev.stir.es`** | no | n/a |
| osTRIS/idax-ledger/idax-shell service credentials | Internal, module-to-Postgres | provisioned automatically by `runtime-provision`/`module-migrations` from the secrets above — no separate manual step | yes (derived) | **no** |

Regenerate DEV secrets from scratch (e.g. after cloning fresh, or deliberately rotating this VM's
own local secrets — **never** do this by copying files from the production host):

```sh
cd /srv/stir/stir/stir-main
python3 scripts/initialize.py
# "if not p.exists(): p.write_text(...)" - never overwrites an existing secret. To force a full
# regeneration on THIS VM only: rm -rf .local/secrets && python3 scripts/initialize.py
```

**Never** copy `.local/secrets/`, `.env`, or any `postgres_password`/`jwt_private_key`/
`login_password` file from the `stir.es` production host onto this VM, and never the reverse. Each
environment's secrets must be independently generated. This applies equally to Seven
Keys/Guardian/WebAuthn material once the MVP is complete: a Seven Keys authority bootstrapped on
this DEV VM (real or `stir-pruebas`-equivalent tenant) must use its own keys/credentials, never
production's.

---

## 10. Database

PostgreSQL 17 (`postgres:17-alpine`), started and migrated automatically as part of the normal
compose dependency chain — there is no separate manual "set up Postgres" step:

```
postgres (healthy)
  → core-migrations       (idax_core schema, via vendored idax-core-runtime's Flyway)
  → module-migrations      (idax_shell, stir, idax_ledger, ostris schemas, via flyway/flyway:11.12.0-alpine
                             reading each module's real migration-* directory)
  → runtime-provision       (creates the idax_backend runtime role, NOSUPERUSER/NOBYPASSRLS, RLS-scoped)
  → shell / stir / ostris / ledger (application containers, using that runtime role only)
```

`stir`'s own migrations live at `stir-backend/src/main/resources/db/migration-stir/` (currently
**V1 through V16**, the last being `V16__webauthn_hardware_custody.sql` — the in-progress
migration for this MVP, see §17). They apply automatically on every `docker compose up`/`deploy.py`
run via the `module-migrations` container above; you never run Flyway by hand against the compose
stack.

Verify the applied version:

```sh
docker compose exec -T -e PGPASSWORD="$(cat .local/secrets/postgres_password)" postgres \
  psql -U postgres -d idax -c "select version, description, installed_on from stir.flyway_schema_history order by version desc limit 5;"
```

RLS: every `stir.*` table is `FORCE ROW LEVEL SECURITY`, scoped by `app.tenant_id`; the app-level
`idax_backend`/`idax_app` role has no bypass. This is enforced by the migrations themselves — no
VM-specific configuration needed.

Resetting **only this VM's DEV data** (never do this against a host with `.local/production`
present unless you have just confirmed with `cat .local/production` that this really is meant to be
wiped):

```sh
# ⚠️ DESTRUCTIVE - deletes all local DEV data (Postgres + MinIO volumes). Confirm you are NOT
# looking at the production compose project first: `cat .local/production` should NOT exist, and
# `docker compose ps` should show project "stir-dev", not "stir-prod".
docker compose down -v
docker compose up -d
```

`docker compose down` alone (no `-v`) **preserves** the database, per `README.md`'s own explicit
note — prefer that whenever you just want to stop the stack.

---

## 11. osTRIS / IDAX

Real, verified service topology (`compose.yml`) — nothing here is assumed:

| Service | Role | Needed for normal dev? | Needed for E2E only? |
|---|---|---|---|
| `postgres` | shared database for every schema | always | always |
| `shell` (IDAX Shell) | auth, tenant/user/role management, extension host, the login UI itself | always | always |
| `stir` | the STIR backend proper | always | always |
| `stir-ui` | STIR's static frontend bundle | always | always |
| `ostris` | economic authority: balances, EXCHANGE commit/reconciliation, participant activation | always — STIR's economic-exchange and Seven Keys flows call it directly | economic exchange / Seven Keys E2E specifically |
| `ostris-ui` | osTRIS's own extension UI | only if you open osTRIS's own screens directly; STIR's UI talks to `ostris` (backend) via the API, not through this UI | rarely |
| `ledger` (IDAX Ledger) | XRPL/ledger proof delivery, configured with `OSTRIS_LEDGER_ENABLED`/proof paths disabled by default in dev | present because osTRIS/Shell expect it in the topology; STIR itself does not call it directly | not typically exercised by STIR's own E2E |
| `ledger-ui` | Ledger's static UI | rarely opened directly | rarely |
| `minio` | S3-compatible object storage for Listing photos/avatars | always (photo upload paths) | attachment-related E2E |
| `proxy` (Caddy) | the one browser-facing origin | always | always |

None of this needs separate "install osTRIS" or "install IDAX" steps — they are all pulled in as
pinned public source (`upstream.lock.json`, §9 pins `idax-core-runtime`, `idax-shell`,
`idax-ledger`, `ostris`, all `github.com/toni-soler/*`) and built by the same
`docker compose build` as STIR itself. `scripts/initialize.py` is what actually fetches/pins them
(`vendor/` directory inside `stir-main`) — see §9's secrets step, which already runs this.

Community/network identity: STIR's own economic binding (community, unit) is created through
STIR's real bootstrap HTTP flow (`POST /api/stir/tenants/{tenant}/economic/marketplace/bootstrap`),
the same one every E2E script in `stir-main/scripts/` already uses via `fixtures()` helpers — no
manual osTRIS-side network/community provisioning step exists or is needed. Never point any of this
at a real production osTRIS network/community; the dev stack's `ostris` container is its own
isolated instance sharing only this VM's local Postgres.

---

## 12. Reverse proxy + HTTPS

There are two valid paths here, and which one applies depends on whether **this VM itself** is
directly reachable from the internet on 80/443, or whether an existing reverse proxy already sits
in front of it on your own network. Real `stir.es` production is confirmed to be **Path A** (its
own isolated VM, direct internet access, no LAN, no other reverse proxy in front of it) — its own
deployment already has its own guide (`stir-main/DEPLOYMENT.md` + `HOST_PROVISIONING.md`), which
this document does not replace or duplicate. This dev VM turned out to be **Path B**: a home LAN
behind a router, with Nginx Proxy Manager (NPM) already handling TLS/Let's Encrypt for several
other services on that same LAN — so this section documents both, and marks which one this
particular VM actually uses.

**Network-isolation note, confirmed**: `stir.es` production runs on a separate, isolated VM with
its own direct internet access — not on this home LAN at all. The separation between DEV and PROD
here is therefore not just secrets-level (§9) but network-level too: nothing on this LAN can reach
production, and production cannot reach this LAN. Keep it that way — never add a route, VPN, or
firewall rule connecting the two "to make testing easier."

### Path A — Caddy does its own ACME (this VM is directly internet-facing; **this is real
`stir.es` production's actual setup**, documented here only for completeness/contrast)

`compose.yml`'s own Caddy (`deploy/Caddyfile`) listens on plain `:8088` inside the network, mapped
to `127.0.0.1:${STIR_HTTP_PORT:-8089}` — this is what `http://localhost:8089` on the VM itself
already gives you (§6.A), no HTTPS, no domain. `compose.production.yml`'s overlay
(`deploy/Caddyfile.production`) is what gives a real domain automatic HTTPS: Caddy performs its own
ACME (Let's Encrypt) issuance/renewal for whatever hostname you put in `STIR_PUBLIC_HOSTNAME`,
publishes real `80/tcp` + `443/tcp` + `443/udp` on every interface, and adds an HSTS header.
Certificate/account state persists in the `caddy_data` Docker volume.

```sh
cd /srv/stir/stir/stir-main
cat > .env <<'EOF'
STIR_PUBLIC_HOSTNAME=dev.stir.es
STIR_PUBLIC_BASE_URL=https://dev.stir.es
STIR_SITE_NAME=STIR (dev)
STIR_TENANT_CODE=stir-dev
STIR_ADMIN_EMAIL=you@yourdomain.example
STIR_SUPPORT_CONTACT=you@yourdomain.example
EOF
docker compose -f compose.yml -f compose.production.yml up -d
```

Only use this path if the VM's own public IP genuinely receives 80/443 traffic directly (no other
reverse proxy in between) — running this *and* Path B's NPM at the same time means two ACME clients
racing for the same certificate, which will not work.

### Path B — an existing reverse proxy (NPM) already terminates TLS on your LAN — **this is what
this dev VM actually uses**

Skip `compose.production.yml`/`Caddyfile.production` entirely — NPM already does ACME/TLS for
`dev.stir.es` (confirmed: NPM proxy host with a Let's Encrypt cert, Force SSL, HSTS all already
configured for other services on the same LAN), so a second ACME client inside the stack would be
redundant and would fight NPM over the same domain. The stack's own Caddy just needs to serve plain
HTTP, reachable from NPM over the LAN instead of only from `127.0.0.1`.

`compose.yml`'s `proxy` service is deliberately loopback-only (`127.0.0.1:${STIR_HTTP_PORT:-8089}`)
— NPM runs on a different host on the LAN and cannot reach `127.0.0.1` on the VM. Fix this with a
**local, never-committed** compose override rather than editing the tracked `compose.yml`:

```sh
cat > /srv/stir/stir/stir-main/compose.override.yml <<'EOF'
services:
  proxy:
    ports:
      - "192.168.1.125:80:8088"
EOF
echo 'compose.override.yml' >> /srv/stir/stir/stir-main/.git/info/exclude
```

(`192.168.1.125` is this specific VM's confirmed LAN IP — adjust if yours differs.
`compose.override.yml` is the filename Docker Compose v2 auto-loads alongside `compose.yml`, no
`-f` needed. `.git/info/exclude` is git's own mechanism for a personal, local-only ignore rule —
nothing tracked or shared is touched. Compose *appends* to the base file's `ports:` list rather than
replacing it, so `http://localhost:8089` on the VM itself keeps working too, per §6.A.)

In NPM: Forward Hostname/IP = this VM's LAN IP (`192.168.1.125`), Forward Port = `80`, Scheme =
`http` — matches what was already configured. No `.env`/`STIR_PUBLIC_HOSTNAME` marker or
`compose.production.yml` overlay needed on the VM side; bring the stack up the normal dev way
(§15/§16), no `-f compose.production.yml`.

**Optional but recommended hardening**: §2's firewall opened `80/tcp`/`443/tcp`/`443/udp` to
"Anywhere," anticipating Path A. Since Path B means NPM — not this VM — is the real internet edge,
this VM no longer needs to accept 80/443 from the raw internet, only from NPM's own host. Narrow it:

```sh
sudo ufw delete allow 80/tcp
sudo ufw delete allow 443/tcp
sudo ufw delete allow 443/udp
sudo ufw allow from 192.168.1.0/24 to any port 80 proto tcp   # adjust to your real LAN subnet
sudo ufw status verbose
```

This is defense in depth, not a fix for something broken — even if your router only ever forwards
80/443 to NPM's own host and never to this VM directly, scoping the VM's own firewall to the LAN
removes the dependency on the router being configured correctly forever.

---

## 13. DNS

`PROVIDER-SPECIFIC STEP` for the actual registrar/DNS host, but the procedure itself is fixed.

If this VM was installed from Ubuntu Server's **"minimized"** image flavor (strips non-essential
utilities), `dig`/`nslookup` are not present — install them first:

```sh
sudo apt-get -y install dnsutils
```

1. Create an `A` record (and `AAAA` if the VM has IPv6) for `dev.stir.es` (or whatever hostname you
   settle on) pointing at whatever host actually terminates TLS for it — **this VM's public IP for
   Path A**, or **your home connection's public IP (reaching NPM via your router's port-forward)
   for Path B**.
2. Wait for propagation, then confirm from **outside** your network entirely (a phone on mobile
   data works well for this — testing from inside the same LAN can misleadingly succeed via local
   DNS/routing even if the public path is broken):
   ```sh
   dig +short dev.stir.es
   # or: nslookup dev.stir.es
   ```
3. Confirm HTTPS actually works end to end:
   ```sh
   curl -sv http://dev.stir.es 2>&1 | head -20      # expect a redirect to https://
   curl -sv https://dev.stir.es 2>&1 | head -20      # expect a valid TLS handshake + STIR/Shell response
   ```
4. **Path A only**: Caddy's ACME issuance happens automatically on first request once DNS resolves
   correctly — `docker compose -f compose.yml -f compose.production.yml logs -f proxy` shows the
   certificate negotiation. `HOST_PROVISIONING.md` §7 notes propagation can take minutes to hours;
   do not restart Caddy repeatedly while waiting, that only risks Let's Encrypt rate limits.
   **Path B**: certificate issuance/renewal is entirely NPM's own concern (its "SSL" tab) — nothing
   to watch on the VM side; `docker compose logs -f proxy` on this VM will only ever show plain HTTP
   requests arriving from NPM's LAN IP, never any ACME activity.

---

## 14. WebAuthn — critical section

Do not change any of this during setup; this section only documents the existing, real
configuration surface so you know what to point at the new domain.

**What actually controls it today** (`stir-backend/src/main/resources/application.yml`, added by
the in-progress MVP — see §17):

```yaml
stir:
  webauthn:
    rp-id: ${STIR_WEBAUTHN_RP_ID:localhost}
    rp-name: ${STIR_WEBAUTHN_RP_NAME:STIR}
    allowed-origins: ${STIR_WEBAUTHN_ALLOWED_ORIGINS:http://localhost:8089}
```

| | `http://localhost:8089` (VM-local, §6.A) | `https://dev.stir.es` (real domain, §12–13) | `https://stir.es` (production — never reuse) |
|---|---|---|---|
| `STIR_WEBAUTHN_RP_ID` | `localhost` (default; browsers accept this one specific unregistered value) | **`dev.stir.es`** | `stir.es` |
| `STIR_WEBAUTHN_ALLOWED_ORIGINS` | `http://localhost:8089` (default) | **`https://dev.stir.es`** | `https://stir.es` |
| HTTPS required? | No — `localhost` is WebAuthn's one exception | **Yes** — any other RP ID requires a secure context (HTTPS) | Yes |
| Credentials portable between columns? | **No.** WebAuthn binds a credential to the exact RP ID it was created under. A credential registered under `rp-id: localhost` is cryptographically scoped to `localhost` and will not be offered by the authenticator for `dev.stir.es`, and vice versa. | — | — |

**Set the two env vars via `compose.production.yml`'s `stir` service environment** (or your own
`.env`, following the same `${VAR:?...}`/`${VAR:-...}` pattern the file already uses for
`STIR_PUBLIC_HOSTNAME` etc.) once you actually run this VM behind `https://dev.stir.es` — the RP ID
must be a registrable domain suffix of the origin (WebAuthn spec requirement), which `dev.stir.es`
satisfies for the origin `https://dev.stir.es` trivially (they're the same host).

**Never let `dev.stir.es` and `stir.es` share a Guardian, a Seven Keys authority, or any registered
WebAuthn credential.** A constitutional seat bootstrapped on this DEV VM (§17) is a disposable
development/test authority — real per `WEBAUTHN_HARDWARE_CUSTODY.md`'s "TEST CEREMONY / NOT
DISTRIBUTED CUSTODY" labeling once that doc exists (§17), never something to later "promote" to
production by copying credential material. Production Seven Keys material must be generated fresh,
by real independent custodians, directly against `https://stir.es`.

**Automated E2E (virtual authenticator)**: §6.A already covers the CDP-based
`WebAuthn.addVirtualAuthenticator` approach — this is what the not-yet-written
`webauthn_hardware_custody_browser.py` (§17) will use, entirely inside the VM, no real hardware
involved, and it never needs a real domain — `http://localhost:8089`'s default `rp-id: localhost`
is sufficient for it.

**Manual validation with a real authenticator** (only meaningful over HTTPS, so only against
`https://dev.stir.es` from your own machine, §6.B): use your laptop's own Windows Hello/Touch ID,
or a real USB security key, in an actual browser session against `https://dev.stir.es`. This is a
one-off manual check, not part of any automated gate — document what you tried and the result, but
do not attempt to script it or make it a CI requirement (matching the original MVP brief: *"Añadir
además una validación manual/documentada con authenticator real... sin convertirla en requisito
para CI"*).

---

## 15. First clean build

Backend (outside Docker, for fast local iteration/tests):

```sh
cd /srv/stir/stir/stir-backend
mvn -s .mvn/public-settings.xml clean verify
```

`.mvn/public-settings.xml` is an intentionally near-empty Maven settings override — it exists so
this build never picks up a developer's own `~/.m2/settings.xml` (private mirrors/credentials) and
only ever resolves from Maven Central plus the one public repo `pom.xml` declares
(`https://toni-soler.github.io/idax-core-runtime/maven2`, the pinned Core artifact). `clean verify`
runs the full test suite including the Testcontainers-backed Postgres tests (§16) — Docker must
already be up (§4) for those to pass.

**Known, already-fixed issue on very new Docker Engine (29+)**: Testcontainers 1.x's bundled
`docker-java` client can fail its own API-version auto-negotiation against a Docker Engine this
new, falling back to a hardcoded old version the daemon then rejects
(`client version 1.32 is too old. Minimum supported API version is 1.40`) — every
Testcontainers-backed test fails with `IllegalStateException: Could not find a valid Docker
environment`, even though `docker` itself works fine from the CLI. This is a real, externally
documented compatibility gap
([testcontainers-java#11210](https://github.com/testcontainers/testcontainers-java/issues/11210)),
discovered and fixed live while first validating this guide on Docker Engine 29.8.1 — already
resolved as of `stir-backend` commit `49e94c1` (`src/test/resources/docker-java.properties`
pinning `api.version=1.44`, test-scope only). If `mvn clean verify` on a fresh checkout still shows
this exact error, confirm that file exists and `git log` includes that commit; if not, `git pull`.

Frontend:

```sh
cd /srv/stir/stir/stir-frontend
npm ci
npm run i18n:validate
npm test
npm run build
```

Full Docker stack (dev profile):

```sh
cd /srv/stir/stir/stir-main
python3 scripts/initialize.py     # idempotent - safe to re-run
docker compose config --quiet     # validates compose.yml + .env without starting anything
docker compose up -d --build
docker compose ps                 # every service should reach "healthy" or "running" within a few minutes
```

Or, preferably, use the deploy script (build → recreate → health-wait → smoke check → auto-rollback
on failure) instead of raw compose:

```sh
python3 scripts/deploy.py           # dev profile (stir-dev project)
# python3 scripts/deploy.py --prod  # only once .local/production + .env (§12) are set
```

**Known, already-fixed issue: MinIO's official images are gone from both `docker.io` and
`quay.io`.** MinIO Inc. archived the upstream repo (2026-04) and pulled the compiled Community
Edition binaries from every public registry (`docker.io` 2026-09-11, `quay.io` 2026-09-24) in
favor of their commercial AIStor product — not a registry migration this time, a full retirement
of free binary distribution, with no anonymous-login workaround (confirmed: no authentication
credentials for that repository are publicly available at all). If `docker compose up -d --build`
fails pulling `minio` with `401 Unauthorized` or `pull access denied`, this is why. Already fixed
as of `stir-main` commit `077b69d`: `compose.yml`'s `minio` service now pulls `pgsty/minio`
instead, a maintained AGPLv3 fork of MinIO CE (github.com/pgsty/minio, the Pigsty project) that
continues building and publishing the same upstream source after the archival, still anonymously
pullable and confirmed a true drop-in (runs as root by default, `/data` pre-exists in the image,
matching this compose file's existing `deploy/run-minio.sh` exactly — unlike `bitnamilegacy/minio`,
also tried and rejected during that investigation: non-root user, no `/data` by default). Pinned by
digest rather than tag, since this is a third-party rebuild, not the official image. See
`CHANGELOG.md`'s "MinIO registry dead again" entry for the full investigation. If a fresh checkout
still references `quay.io/minio/minio`, `git pull`.

RAM/disk consumption: not independently re-measured for this document (no reliable measurement
tool was run against this exact VM) — do not repeat a specific number here as fact. Watch it
yourself the first time with `docker stats` (live) and `df -h` (before/after) rather than trusting
a guessed figure.

---

## 16. Test matrix

Every command below is real — copied from `stir-backend/pom.xml`/`package.json` scripts and the
actual filenames in `stir-main/scripts/`. "Destructive?" means "touches or deletes real DEV data on
this VM," not "unsafe to run" — all of them are safe to run repeatedly against a disposable DEV
stack.

| Suite | Command | Prerequisites | Duration | Destructive? |
|---|---|---|---|---|
| Backend unit + Postgres (Testcontainers) | `mvn -s .mvn/public-settings.xml clean verify` (from `stir-backend`) | Docker running (spins up disposable `postgres:17-alpine` containers itself) | a few minutes (216 tests as of §17's checkpoint) | No — Testcontainers containers are ephemeral |
| Frontend unit | `npm test`, or `node --test` directly if that fails (see note below) (from `stir-frontend`) | Node 22, `npm ci` once | seconds | No |
| Frontend i18n | `npm run i18n:validate` (from `stir-frontend`) | none beyond `npm ci` | seconds | No |
| Frontend build | `npm run build` (from `stir-frontend`) | `npm ci` | seconds | No |
| Full stack up | `python3 scripts/deploy.py` (from `stir-main`) | Docker, `.local/secrets/*` present | minutes (first build longest) | No (never touches volumes) |
| HTTP smoke | `python3 scripts/smoke.py` | stack up | seconds | No |
| Multitenant fixtures | `python3 scripts/multitenant.py` | stack up | seconds | Creates disposable tenants only |
| Economic exchange E2E | `python3 scripts/economic_exchange_e2e.py` | stack up, venv (§6) for `cryptography` | seconds–low minutes | No (disposable tenant) |
| Community references (HTTP) | `python3 scripts/community_references_e2e.py` | stack up | seconds | No |
| Community references (browser) | `python3 scripts/community_references_browser.py` | stack up, Edge + Playwright (§6) | ~1 min | No |
| Market integrity / Seven Keys (HTTP, Ed25519) | `python3 scripts/market_integrity_e2e.py` | stack up, venv+`cryptography` | seconds–low minutes | No |
| Ordinary governance (HTTP + browser) | `python3 scripts/ordinary_governance_e2e.py` / `ordinary_governance_browser.py` | stack up (+ Edge for the browser one) | ~1 min each | No |
| Participant independence (HTTP + browser) | `python3 scripts/participant_independence_e2e.py` / `participant_independence_browser.py` | stack up (+ Edge) | ~1 min each | No |
| Community value governance (HTTP + browser) | `python3 scripts/community_value_governance_e2e.py` / `_browser.py` | stack up (+ Edge) | ~1 min each | No |
| Consent/retention (HTTP + browser) | `python3 scripts/consent_retention_e2e.py` / `_browser.py` | stack up (+ Edge) | ~1 min each | No |
| Multi-source value evidence (HTTP + browser) | `python3 scripts/multi_source_value_evidence_e2e.py` / `_browser.py` | stack up (+ Edge) | ~1 min each | No |
| **WebAuthn hardware custody (HTTP)** | `python3 scripts/webauthn_hardware_custody_e2e.py` | stack up, venv+`cryptography` | unverified — **never yet run successfully end to end**, see §17 | No (disposable tenant), but currently unproven |
| **WebAuthn hardware custody (browser)** | *not yet written* | Edge + Playwright + CDP virtual authenticator (§6.A) | n/a | n/a |
| Backup/restore round trip | `python3 scripts/backup_restore_e2e.py` | stack up | a few minutes | Operates on a throwaway copy, but read `stir-doc/VALIDATION.md` before assuming that on a shared VM |
| Public-source audit | `python3 scripts/audit-public.py` | none | seconds | No |

For the two rows marked unproven/not-yet-written, §17 is the authoritative next step — do not treat
this matrix's presence of a filename as proof the script currently passes.

**Known, environment-specific Node quirk**: on this VM's Node 22.23.3 (NodeSource build), `npm test`
(which runs `package.json`'s literal `node --test tests/`) fails with
`Error: Cannot find module '.../stir-frontend/tests'` — Node appears to resolve the bare `tests/`
positional argument as a module specifier rather than a relative path in this build. Bare
`node --test` (no path argument at all) correctly auto-discovers and runs the same suite instead
(confirmed: 40/40 pass) - use that directly if `npm test` fails with this exact error. Not a real
test failure; not chased further since a working equivalent exists.

---

## 17. Resuming the in-progress WebAuthn MVP — read this before touching anything

**Status: COMPLETE, merged to `main` in all four repos** (`stir-backend e127f20`,
`stir-frontend 9fb2211`, `stir-main bbfbba4`, `stir-doc f85f9f7`), verified on this VM: `mvn verify`
216/216, `npm test` 45/45 + build + i18n (12 locales, 560 keys), `webauthn_hardware_custody_e2e.py`
full pass, `webauthn_hardware_custody_browser.py` full pass (real CDP virtual-authenticator
ceremonies through the rendered UI), `governance_ui_browser.py` regression re-confirmed passing,
clean `docker compose up -d --build` with every service healthy. Full design in
`WEBAUTHN_HARDWARE_CUSTODY.md`; the complete list of real bugs found (three backend/script bugs
plus a four-part regression in a previously-passing browser E2E, every one found only by actually
running code that had never been executed before) is in
`VALIDATION_WEBAUTHN_HARDWARE_CUSTODY.md`. The rest of this section is kept as-is below as the
historical record of how the transfer from the memory-constrained local PC to this VM actually
happened — useful if the same situation (uncommitted multi-repo work stuck on a machine that can't
run it) recurs for a future capability.

**This was the single most important section for getting back to exactly where work stopped.**

### 17.0 What actually stopped it

Not a code defect. The local development PC ran out of memory during `docker compose build`
(Testcontainers-backed `mvn test` runs plus a concurrent Docker image build together exceeded
available RAM) — twice. That is the direct reason this VM is being provisioned (§1's RAM
recommendation is sized specifically to not repeat this).

### 17.1 Exactly what state exists, and where

All five repos are, on the original PC, checked out on the **same branch name**:

```
claude/webauthn-hardware-custody-mvp
```

in every one of `stir-backend`, `stir-frontend`, `stir-main`, `stir-doc`, and the `stir-workspace`
root. **None of this work has been committed yet** — every file below exists only as an uncommitted
working-tree change on that PC. This is a real, explicit risk (see §17.3) — confirm it yourself
with `git status --short` in each repo on that PC before doing anything else; do not trust this list
as a substitute for that command (it could not be run to produce this document, see the note at the
end of this section).

**stir-backend** (new files marked *new*; the rest are edits to existing files):
- `pom.xml` — added the `com.fasterxml.jackson.dataformat:jackson-dataformat-cbor` dependency
  (parses WebAuthn's CBOR-encoded `attestationObject`/`authenticatorData`/COSE public keys — no
  general WebAuthn relying-party library was added, by design).
- `src/main/resources/application.yml` — added the `stir.webauthn.*` block (§14).
- `src/main/resources/db/migration-stir/V16__webauthn_hardware_custody.sql` *(new)* — widens
  `constitutional_seat`/`constitutional_credential_history`/`constitutional_authority`'s public-key
  columns to `text`, adds `credential_type`/`algorithm` columns (default `SOFTWARE_ED25519`/
  `Ed25519`, so every existing Ed25519 credential keeps working unchanged), adds
  `constitutional_webauthn_credential` and `constitutional_webauthn_challenge` tables, widens
  `constitutional_signature` to also carry a WebAuthn envelope (`client_data_json`,
  `authenticator_data`) alongside the existing `signature_base64url`.
- `src/main/java/org/stir/reference/CredentialEnvelope.java` *(new)* — the
  `GovernancePayload`/`CredentialSignatureEnvelope` abstraction the MVP brief asked for.
- `src/main/java/org/stir/reference/WebAuthnCrypto.java` *(new)* — CBOR/COSE/authenticatorData
  parsing and ES256/RS256/EdDSA assertion+registration verification, attestation format `none`
  only.
- `src/main/java/org/stir/reference/WebAuthnCredentialService.java` *(new)* — registration
  challenge lifecycle (single-use, expiring, actor/authority/context-bound) and credential storage.
- `src/main/java/org/stir/reference/SevenKeysService.java` — generalized to dispatch signature
  verification by credential type; every new field on every input record is additive (old 3–7 arg
  constructors preserved via secondary constructors), so the existing Ed25519-only wire shape still
  works byte-for-byte.
- `src/main/java/org/stir/reference/SevenKeysController.java` — added
  `/webauthn/register/begin`/`/webauthn/register/finish`.
- `src/test/java/org/stir/reference/WebAuthnCryptoTest.java` *(new, 7 tests)*.
- `src/test/java/org/stir/reference/WebAuthnSevenKeysPostgresTest.java` *(new, 7 tests)* — mixed
  Ed25519+WebAuthn bootstrap, valid signing, replay/cross-proposal/cross-community/cross-tenant
  rejection, suspended-credential rejection, same-controller Ed25519→WebAuthn and
  WebAuthn→WebAuthn rotation, sign-counter cloning detection.
- `src/test/java/org/stir/reference/WebAuthnCredentialServicePostgresTest.java` *(new, 8 tests)* —
  registration challenge single-use/expiry/actor/context binding, duplicate-credential rejection,
  tenant isolation.
- `src/test/java/org/stir/reference/SevenKeysPostgresTest.java` — two call sites updated for the
  widened `SevenKeysService` constructor (now takes a `WebAuthnCredentialService` too).

**Verified state**: `mvn clean test` (full suite, run cleanly — not concurrently with any other
build in the same directory, which is what caused a false failure earlier in this session) passed
**216/216**, zero regressions on the pre-existing 194.

**stir-frontend**:
- `src/webauthn-signer.js` *(new)* — `navigator.credentials.create()`/`get()` wrappers, a
  same-device IndexedDB cache for a registered credential's `webauthnCredentialId`/`algorithm`/
  `rpId` (mirroring `governance-signer.js`'s own per-role storage convention), and
  `challengeFor()` (the SHA-256-of-the-domain-separated-message challenge construction).
- `src/governance.jsx` — heavily extended: a `CredentialRegistrar` component offering a
  same-device-Ed25519-vs-WebAuthn choice (with an explicit "TEST CEREMONY / NOT DISTRIBUTED
  CUSTODY" warning on the Ed25519 path) everywhere a credential is registered (bootstrap seats/
  guardian, `ROTATE_CREDENTIAL`/`REPLACE_CONTROLLER`/`APPOINT_GUARDIAN` proposals); WebAuthn signing
  paths added to proposal signing, activation (guardian + new-key possession signatures), and
  Guardian suspension; credential-type badges throughout (seat list, credential history).
- `src/api.js` — added `beginWebauthnRegistration`/`finishWebauthnRegistration`.
- `src/style.css` — added `.stir-badge-warning` (used by the test-ceremony warning badge).
- `src/locales.json` — +8 new keys × 12 locales (560 keys/locale total, up from 552).
- `tests/webauthn-signer.test.mjs` *(new, 5 tests)*.

**Verified state**: `npm test` 45/45, `npm run build` clean, `npm run i18n:validate` clean (560
keys × 12 locales).

**stir-main**:
- `scripts/webauthn_hardware_custody_e2e.py` *(new, untracked)* — a from-scratch Python WebAuthn
  ceremony simulator (hand-rolled minimal CBOR encoder, ES256 key generation, real
  `authenticatorData`/`attestationObject`/`clientDataJSON` construction, real ECDSA signing) driving
  the real HTTP endpoints. **Never successfully executed against a live backend** — Docker builds
  were killed by host memory pressure before a deploy ever completed far enough to run it. Treat
  this script as **unverified, possibly containing bugs**, not as a passing gate.
- `scripts/webauthn_hardware_custody_browser.py` — **not started**.

**stir-doc**: no files changed yet. Still pending from the original MVP brief: updates to
`SEVEN_KEYS_GOVERNANCE.md`, `CREDENTIAL_RECOVERY.md`, a new `WEBAUTHN_HARDWARE_CUSTODY.md` design
doc, a `VALIDATION_WEBAUTHN_HARDWARE_CUSTODY.md` record, and a `GOVERNANCE_CAPTURE_THREAT_MODEL.md`
update.

**stir-workspace** (root `CLAUDE.md`): no changes yet. Still pending: a new section summarizing this
capability, mirroring the existing "Consent and retention" / "Multi-source value evidence"
sections.

### 17.2 What is genuinely left to do

1. Get the full Docker stack to build and reach healthy on **this VM** (§15) — the step that kept
   failing for lack of RAM.
2. Run `webauthn_hardware_custody_e2e.py` for the first time; fix whatever it surfaces (treat this
   exactly like every prior MVP in this session's history — expect and fix 1–3 real bugs on first
   real run, do not assume it will pass unmodified).
3. Write and run `webauthn_hardware_custody_browser.py` (§6.A's CDP virtual-authenticator sketch is
   the starting point), covering the 18 adversarial scenarios the original MVP brief listed
   (registration, possession proof, valid signature, challenge replay, cross-proposal/community/
   tenant replay, revoked-credential rejection, historical verifiability, software→WebAuthn and
   WebAuthn→WebAuthn rotation, Guardian suspension, same-controller recovery, SuperAdmin exclusion,
   6-of-7 still insufficient, Guardian-never-substitutes-a-seat, test-ceremony-vs-hardware-custody
   UI distinction, existing-Ed25519-path-still-works).
4. Regression: re-run the pre-existing HTTP/browser E2E scripts (§16) to confirm nothing in the
   `SevenKeysService` generalization broke the Ed25519-only path over real HTTP (the Postgres-level
   tests already prove this at the service layer; the E2E layer has not yet re-confirmed it).
5. Write the four `stir-doc` files and the `stir-workspace` `CLAUDE.md` section listed above.
6. Full gate re-run: `mvn verify`, `npm test`/`build`/`i18n:validate`, clean `docker compose build`,
   full E2E regression — then branch-ancestry check (§8 step 5) and merge/push, per this workspace's
   standing convention.

### 17.3 Moving the work to this VM — the actual risk, and the correct procedure

**The risk, stated plainly**: because none of the WebAuthn work described in §17.1 has been
committed, **Git alone cannot recover it on a new machine**. `git fetch`/`git clone` on the VM will
only ever retrieve what has actually been pushed to `origin` — and nothing has been pushed for this
branch, because nothing has been committed. If the original PC's working directories were lost right
now (disk failure, accidental `git checkout .`/`git clean -fd`, a bad `git reset --hard`), this
entire increment would have to be redone from scratch.

**Do not** "solve" this by zipping the working directories and copying the archive to the VM. Git
can do this correctly and safely; a raw file copy cannot distinguish tracked-modified files from
build output/`node_modules`/`target/` from genuinely untracked new files, and gives you no commit
history, no diff, no ancestry check (§8 step 5).

**The correct procedure**, to run **on the original PC**, before touching the VM at all:

```sh
# Repeat for each of: stir-backend, stir-frontend, stir-main, stir-doc, stir (workspace root)
cd <repo>
git status --short                      # confirm this matches §17.1's file list - if it doesn't,
                                          # STOP and reconcile before committing anything
git diff                                 # review the actual diff, not just filenames
git add <the specific files from §17.1>  # never `git add -A`/`git add .` blindly - review what's
                                          # staged (`git status`) before committing
git commit -m "wip(webauthn): <describe what this repo's slice covers>"
git push -u origin claude/webauthn-hardware-custody-mvp
```

This makes each repo's branch a normal, real branch on GitHub — nothing special about "wip", it is
just an honest commit message signaling the work is not yet a finished, gate-passing increment.
Since all five repos are **public** GitHub repos (`toni-soler/stir-*`), pushing this branch makes
its content publicly visible immediately, exactly as every other feature branch in this session's
history already has been — this is the established workflow, not a new exposure.

Then, **on the VM**, after §7's clone:

```sh
for r in stir-backend stir-frontend stir-main stir-doc; do
  (cd /srv/stir/stir/$r && git fetch origin && git checkout claude/webauthn-hardware-custody-mvp)
done
cd /srv/stir/stir && git fetch origin && git checkout claude/webauthn-hardware-custody-mvp
```

Verify the transfer actually succeeded before trusting it — compare against §17.1's file list:

```sh
for r in stir-backend stir-frontend stir-main stir-doc; do
  echo "=== $r ==="
  (cd /srv/stir/stir/$r && git log --oneline -3 && git diff --stat origin/main..HEAD)
done
```

**Note on how this section was verified**: `git status --short` was run against all five repos
while writing this document (after an earlier transient tool-availability issue in this session
resolved) and the output matched §17.1's file list exactly, with one addition: `stir-doc` now also
shows `?? DEV_VM_SETUP.md` — this document itself, written on the same branch as the rest of this
MVP's uncommitted work, but not part of the WebAuthn implementation. Re-run `git status --short`
yourself on the original PC immediately before committing regardless — this was a point-in-time
check, and confirming nothing changed since is cheap insurance.

---

## 18. Daily operation: Claude Code / Codex

Workflow: `SSH → repo/worktree → agent → tests → Docker`. The two agents must never operate on the
same working tree at the same time — `stir/CLAUDE.md` already warns that a `git checkout` in one
session silently redirects the other's next commands; on a shared VM with two agents that risk is
constant, not occasional.

**Prefer separate git worktrees over separate clones** — one clone per repo, multiple worktrees off
it, so both agents share one `.git` (one fetch keeps both current) without ever sharing one checked-
out branch:

```sh
# One-time, per repo, after the plain clone from §7:
cd /srv/stir/stir/stir-backend
git worktree add /srv/stir/worktrees/claude/stir-backend claude/some-capability-mvp -b claude/some-capability-mvp
git worktree add /srv/stir/worktrees/codex/stir-backend codex/some-other-capability -b codex/some-other-capability
```

Suggested layout:

```
/srv/stir/
  stir/                          (the five plain clones from §7 - always stay on `main` here)
  worktrees/
    claude/
      stir-backend/  stir-frontend/  stir-main/  stir-doc/
    codex/
      stir-backend/  stir-frontend/  stir-main/  stir-doc/
```

Each agent's session `cd`s into its own `worktrees/<agent>/...` tree and never into `stir/<repo>`
directly (that one stays on `main`, used only for the fetch/branch/merge/push steps of §8's
procedure). Docker Compose is the one resource that genuinely cannot be trivially duplicated per
worktree (it is one stack, one set of ports, one set of container names) — coordinate who "owns" a
live `docker compose up` at a given moment the same way you would with two human engineers sharing
one dev server; do not attempt to run two full stacks in parallel on one VM without first thinking
through port collisions (§9's `.env` port variables are the only knobs `compose.yml` exposes for
that, and you would need a second, differently-named compose project — this is real added
complexity, only worth it if genuinely needed).

`mvn test`/`npm test` from an agent's own worktree work independently of the shared Docker stack
(Testcontainers spins up its own disposable Postgres per run) and are safe to run concurrently from
both worktrees.

---

## 19. Snapshots and backups (DEV)

Four genuinely different things — do not conflate them:

| | What it captures | How | When |
|---|---|---|---|
| VM snapshot | Entire disk state (OS, Docker images, everything) | `PROVIDER-SPECIFIC STEP` (your hypervisor/cloud's own snapshot feature) | Before a risky OS-level change (kernel upgrade, Docker major version bump) |
| DB backup | `pg_dump` of every schema | `python3 scripts/backup.py [--keep N]` (default keeps last 14 runs under `.local/backups/`) | Before a migration you're unsure about, before a destructive test, on a cron (`DEPLOYMENT.md`'s own example: daily 03:15) |
| Docker volume backup | MinIO's `minio_data` volume (photos/avatars) | included automatically in `backup.py`'s same run (tar of the volume) | same cadence as the DB backup — they're one script, one timestamped directory |
| Git | Source code and its history | `git commit`/`git push` (§8, §17.3) | Continuously — this is not a periodic snapshot, it's the actual mechanism |

**A VM snapshot is not a substitute for a DB/volume backup**, and neither is a substitute for git.
Recommended before: an important migration (e.g. §17's `V16`), any "let's try this and see" audit
of production-adjacent config, or a Docker/OS package upgrade.

---

## 20. Reset / recovery

```sh
# Restart just the app services (keeps data):
docker compose restart

# Rebuild + redeploy from current source (keeps data, auto-rolls-back on failed health/smoke):
python3 scripts/deploy.py            # or --prod once §12's overlay is in use

# ⚠️ DESTRUCTIVE - wipe ONLY this VM's DEV data (Postgres + MinIO), keep source/images:
docker compose down -v
docker compose up -d

# Restore from a specific backup.py snapshot (⚠️ DESTRUCTIVE - replaces ALL current data):
python3 scripts/restore.py .local/backups/<timestamp>
# In dev this asks you to type "yes"; if .local/production is present it instead asks you to type
# the exact compose project name (stir-prod) - never pass --yes on a copy-pasted command without
# reading what you're about to overwrite first.

# Restore a whole-VM snapshot: PROVIDER-SPECIFIC STEP (your hypervisor/cloud's own restore flow).

# After any reset: re-verify health before resuming work.
python3 scripts/status.py
python3 scripts/smoke.py
```

---

## 21. Final checklist

```
[ ] SSH key-based login works; password auth disabled
[ ] ufw (or provider firewall) allows only 22, 80, 443/tcp, 443/udp
[ ] free -h / swapon --show show the RAM+swap from §1/§3
[ ] docker version / docker compose version / docker run hello-world all succeed
[ ] java -version → 21; mvn -version; node --version → v22.x; python3 --version → 3.10+
[ ] Microsoft Edge installed; playwright install-deps run; pip install -r requirements-browser.txt + cryptography
[ ] Five repos cloned as siblings under /srv/stir/stir (or your chosen root)
[ ] git remote -v / git branch --show-current / git status --short clean on each, verified per §8
[ ] .local/secrets/* generated via scripts/initialize.py (this VM's own, never copied from prod)
[ ] PostgreSQL up, module-migrations completed, V1..V16 present in stir.flyway_schema_history
[ ] osTRIS / IDAX Shell / IDAX Ledger containers healthy (docker compose ps)
[ ] DNS for dev.stir.es resolves to this VM (dig +short)
[ ] HTTPS live at https://dev.stir.es (curl -sv, valid cert, Caddy logs show ACME success)
[ ] STIR_WEBAUTHN_RP_ID / STIR_WEBAUTHN_ALLOWED_ORIGINS set to dev.stir.es / https://dev.stir.es
[ ] mvn clean verify → 216/216 (stir-backend, on this VM, not just carried over from the old PC)
[ ] npm test / npm run build / npm run i18n:validate all green (stir-frontend, on this VM)
[ ] docker compose build (or scripts/deploy.py) completes without OOM
[ ] Pre-existing HTTP E2E suite (§16 table) passes on this VM
[ ] Pre-existing browser E2E suite (§16 table) passes on this VM
[ ] webauthn_hardware_custody_e2e.py run for the first time; bugs found fixed (§17.2)
[ ] webauthn_hardware_custody_browser.py written and passing (§17.2)
[ ] stir-doc WebAuthn docs written (§17.2 item 5)
```

If every box above is checked: **VM READY FOR DEVELOPMENT.** If any box in the first 14 is
unchecked, stop and resolve it before resuming the WebAuthn MVP — the last 4 boxes are the MVP's own
remaining work (§17.2), not VM-readiness criteria, and should be tackled only once the VM itself is
confirmed solid.

---

## Security notes (apply throughout, not just once)

- No real password/secret value appears anywhere in this document — every secret is generated by
  `scripts/initialize.py` on the target host itself.
- PostgreSQL and MinIO are never exposed beyond `127.0.0.1` / the internal Docker `data` network —
  do not add a port mapping for them "just to inspect the DB easily"; use
  `docker compose exec postgres psql ...` instead (§10).
- The Docker daemon itself is never exposed on a TCP socket to the network — only local Unix-socket
  access via the `docker`-group membership from §4.
- Day-to-day work (both human and agent) runs as the non-root admin user from §1/§2, never `root`.
- `.env` and `.local/` are `.gitignore`d in `stir-main` — never force-add or commit them.
- This guide never asks you to lower any STIR constitutional/governance invariant (7-of-7, Guardian
  separation, RLS, etc.) to make DEV "easier" — if a step here ever seems to require that, stop and
  treat it as a bug in this document, not a shortcut to take.
