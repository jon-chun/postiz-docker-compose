# Postiz Local Setup & Configuration Plan

> **Audience:** A skilled developer who knows the shell but has never set up Postiz, OrbStack, or LiteLLM. Execute top-to-bottom; each task is independent and self-contained.

**Goal:** Install, configure, run, and operate Postiz locally on a MacBook Pro M4 using OrbStack (instead of Docker Desktop), with **all bulk storage on the external OWS 4 TB SSD** (`/Volumes/RESEARCH_SSD`) and the AI copilot reaching either a remote commercial LLM (OpenAI / Anthropic) *or* a local Ollama model via a LiteLLM OpenAI-compatible bridge.

**Architecture:**
- Postiz stack = 9 containers (app, Postgres 17, Redis 7.2, Spotlight monitor, Temporal + ES + Postgres 16 + admin + UI), wired by Docker Compose.
- OrbStack runs the containers; its VM data dir is symlinked to the external SSD, so all images/volumes live there automatically.
- AI copilot is configured via `OPENAI_API_KEY`. To use Ollama instead, a sidecar **LiteLLM** container exposes an OpenAI-compatible endpoint that forwards to Ollama running on the host.

**Tech stack:** macOS Darwin 24.6 (Apple Silicon M4) · OrbStack 2.1.3 · Ollama 0.24 · Docker Compose v2 · Postiz `ghcr.io/gitroomhq/postiz-app:latest` · Postgres 17 · Redis 7.2 · Temporal 1.28.1 · Elasticsearch 7.17 · LiteLLM (optional)

**Environment baseline (confirmed on this machine, 2026-05-25):**

| Item | Value |
|------|-------|
| External SSD | `/Volumes/RESEARCH_SSD` (APFS, 3.5 TB free of 4.0 TB) |
| OrbStack data dir | `~/Library/Group Containers/HUAQ24HBR6.dev.orbstack/data` → symlink → `/Volumes/RESEARCH_SSD/orbstack/data` |
| OrbStack resources | 10 vCPU, 12 288 MiB RAM, Rosetta on |
| Ollama models | `OLLAMA_MODELS=/Volumes/RESEARCH_SSD/ai_caches/ollama_models` |
| Repo location | `/Users/jonc/code/postiz-docker-compose` (this directory) |
| Postiz port mapping | `4007:5000` (host : container) |
| Frontend URL | `http://localhost:4007` |
| Temporal UI | `http://localhost:8080` |
| Spotlight (dev only) | `http://localhost:8969` |

> **Grok-doc errata you should ignore:** Grok's note said the Postiz container maps `4007:3000` and recommended `mv` + symlink the OrbStack `data` dir. Both are out of date for the version of the repo and OrbStack present on this machine — see Task 2 and Task 6.

---

## Task 0 — Pre-flight Verification

**Files:** none (read-only checks)

- [ ] **Step 0.1 — Confirm the external SSD is mounted and healthy**

```bash
df -h /Volumes/RESEARCH_SSD
diskutil info /Volumes/RESEARCH_SSD | grep -E "(Name|Type|Free|Total|Read-Only)"
```

Expected: filesystem mounted, APFS, ≥ 50 GB free (Postiz stack pulls ~3–4 GB of images, plus DB volumes that grow over time). **Do not proceed if the drive is read-only or unmounted** — every volume in Task 6 anchors to it.

- [ ] **Step 0.2 — Confirm OrbStack is running and reachable**

```bash
orb status
docker version --format '{{.Server.Version}}'
```

Expected: `orb status` reports `Running`, `docker version` returns a version (OrbStack is shimming the Docker CLI). If `docker` errors with "Cannot connect", launch OrbStack from /Applications or `open -a OrbStack` and re-test.

- [ ] **Step 0.3 — Confirm OrbStack storage already lives on the external SSD**

```bash
ls -la ~/Library/Group\ Containers/HUAQ24HBR6.dev.orbstack/data
```

Expected output (must show a symlink → external SSD):

```
data -> /Volumes/RESEARCH_SSD/orbstack/data
```

If it's a real directory instead of a symlink, **stop and complete Task 2** before continuing — pulling Postiz images now would write ~4 GB to the internal SSD.

- [ ] **Step 0.4 — Confirm Ollama is running and models are on the external SSD**

```bash
echo "OLLAMA_MODELS=$OLLAMA_MODELS"
ollama --version
curl -fsS http://localhost:11434/api/tags | head -c 300
```

Expected: `OLLAMA_MODELS=/Volumes/RESEARCH_SSD/ai_caches/ollama_models`, version printed, and the tags endpoint returns JSON (even if `{"models":[]}`). If `curl` fails, run `ollama serve &` in another shell, or `brew services start ollama`.

- [ ] **Step 0.5 — Confirm host ports are free**

```bash
for p in 4007 5432 6379 7233 8080 8969 4000 11434; do
  printf 'port %-6s ' "$p"; lsof -nP -iTCP:$p -sTCP:LISTEN 2>/dev/null | tail -n +2 || echo free
done
```

Expected: only `11434` is in use (by Ollama). Free everything else, or remap in Task 6 if you have other workloads on those ports.

---

## Task 1 — Storage Layout on the External SSD

**Files:**
- Create: `/Volumes/RESEARCH_SSD/postiz/` (project root for state, configs, backups)

This task ensures every byte of mutable state lives on the external drive. Docker Compose **named volumes** are stored inside OrbStack's data dir (already symlinked to the SSD — Task 0.3), so they're handled. We only need an extra tree for hand-edited configs and backups.

- [ ] **Step 1.1 — Create the directory layout**

```bash
mkdir -p /Volumes/RESEARCH_SSD/postiz/{config,backups,litellm,scripts}
ls /Volumes/RESEARCH_SSD/postiz
```

Expected: four directories listed.

- [ ] **Step 1.2 — Sanity-check write permissions**

```bash
touch /Volumes/RESEARCH_SSD/postiz/.writable && rm /Volumes/RESEARCH_SSD/postiz/.writable && echo "OK"
```

Expected: `OK`. If permission denied, run `sudo chown -R $USER /Volumes/RESEARCH_SSD/postiz`.

---

## Task 2 — OrbStack: Move Data to External SSD (only if Task 0.3 failed)

**Files:**
- Modify: `~/Library/Group Containers/HUAQ24HBR6.dev.orbstack/data` (turn into a symlink)
- Create: `/Volumes/RESEARCH_SSD/orbstack/data/`

Skip this entire task if Step 0.3 already showed the symlink. Otherwise:

- [ ] **Step 2.1 — Quit OrbStack cleanly**

```bash
orb stop
osascript -e 'quit app "OrbStack"'
# Wait for it to actually stop
while pgrep -x OrbStack >/dev/null; do sleep 1; done
echo "OrbStack quit."
```

- [ ] **Step 2.2 — Move existing data (preserves images & volumes; might be empty on a fresh install)**

```bash
SRC="$HOME/Library/Group Containers/HUAQ24HBR6.dev.orbstack/data"
DST="/Volumes/RESEARCH_SSD/orbstack/data"
mkdir -p /Volumes/RESEARCH_SSD/orbstack
if [ -d "$SRC" ] && [ ! -L "$SRC" ]; then
  rsync -aH --info=progress2 "$SRC"/ "$DST"/
  mv "$SRC" "$SRC.backup-$(date +%Y%m%d-%H%M%S)"
fi
ln -s "$DST" "$SRC"
ls -la "$SRC"
```

Expected: `…/data -> /Volumes/RESEARCH_SSD/orbstack/data`.

- [ ] **Step 2.3 — Relaunch OrbStack and verify**

```bash
open -a OrbStack
sleep 5
orb status
docker info | grep -i "Docker Root Dir"
```

Expected: status `Running`, `docker info` works. After confirming everything boots, delete the backup directory created in Step 2.2.

- [ ] **Step 2.4 — Tune OrbStack resources** (recommended for Postiz + Ollama)

In the OrbStack menu bar icon → **Settings → System**, confirm:

| Setting | Value |
|---------|-------|
| CPU | 8–10 cores |
| Memory | 12–16 GiB |
| Rosetta | On |

Current machine already at 10 CPU / 12 288 MiB — fine. If you plan to run a 13 B+ Ollama model concurrently, bump RAM further or run Ollama on the host (default) and keep OrbStack at 12 GB.

---

## Task 3 — Verify Ollama (only if you'll use a local model in Task 4)

**Files:** none

- [ ] **Step 3.1 — Pull a small starter model on the external SSD**

```bash
ollama pull llama3.2:3b           # ~2 GB, fits easily in M4 RAM
ollama list
du -sh /Volumes/RESEARCH_SSD/ai_caches/ollama_models
```

Expected: model listed, disk usage on the external SSD grew. Pick a different tag (`qwen2.5:7b`, `phi3:mini`, etc.) if you prefer.

- [ ] **Step 3.2 — Confirm host-network reachability from inside a container**

```bash
docker run --rm curlimages/curl:8.10.1 \
  -fsS http://host.docker.internal:11434/api/tags
```

Expected: JSON response listing the model from 3.1. **If this fails**, OrbStack's `host.docker.internal` is broken — fall back to your LAN IP: `ipconfig getifaddr en0`, then use `http://<that-ip>:11434` everywhere Ollama is referenced below.

---

## Task 4 — Choose Your LLM Provider Path

Postiz officially supports **only** `OPENAI_API_KEY`. There are two viable paths:

| Path | LLM source | Pros | Cons |
|------|-----------|------|------|
| **A. Commercial** | OpenAI / Anthropic key direct | Zero proxy, best quality, no extra container | $$$, data leaves machine |
| **B. Ollama via LiteLLM** | Local Ollama, OpenAI-compatible proxy | Free, private, runs on M4 | Smaller models, extra container, **relies on Postiz honoring `OPENAI_BASE_URL`** (works with current openai-node SDK; treat as experimental — see Step 8.3 fallback) |

- [ ] **Step 4.1 — Pick a path** and record it. If unsure, start with A (it always works) and switch later by changing two env vars.

---

## Task 5 — Generate Secrets

**Files:**
- Create: `/Volumes/RESEARCH_SSD/postiz/secrets.env` (chmod 600, never commit)

- [ ] **Step 5.1 — Generate a JWT secret**

```bash
SECRET=$(openssl rand -base64 48 | tr -d '\n=' | tr '/+' '_-')
echo "JWT_SECRET=$SECRET"
```

Expected: a long random string, ~64 chars. Copy it.

- [ ] **Step 5.2 — Write the secrets file**

```bash
umask 077
cat > /Volumes/RESEARCH_SSD/postiz/secrets.env <<EOF
JWT_SECRET=$SECRET
# Path A only — paste your real key, otherwise leave empty:
OPENAI_API_KEY=
# Optional — Stripe / social keys can go here too, one KEY=value per line.
EOF
chmod 600 /Volumes/RESEARCH_SSD/postiz/secrets.env
ls -la /Volumes/RESEARCH_SSD/postiz/secrets.env
```

Expected: file permissions `-rw-------`, owner = you.

---

## Task 6 — Customize `docker-compose.yaml`

**Files:**
- Modify: `/Users/jonc/code/postiz-docker-compose/docker-compose.yaml`
- Create: `/Users/jonc/code/postiz-docker-compose/.env` (Compose interpolation source)

The repo ships a working compose file. We will (a) externalize secrets via `.env` interpolation, (b) wire AI env vars, and (c) leave the volume layout alone — named volumes already land on the external SSD via the OrbStack symlink.

- [ ] **Step 6.1 — Create a project `.env` for Compose interpolation**

```bash
cd /Users/jonc/code/postiz-docker-compose
cat > .env <<'EOF'
# Pulled in by docker compose for ${VAR} interpolation in docker-compose.yaml.
# This file is gitignored (verify with: git check-ignore -v .env). Do NOT commit secrets.
JWT_SECRET=__PASTE_FROM_secrets.env__
OPENAI_API_KEY=
OPENAI_BASE_URL=
EOF
chmod 600 .env
```

Then paste the actual `JWT_SECRET` from `/Volumes/RESEARCH_SSD/postiz/secrets.env` into the new `.env`. Leave `OPENAI_*` empty for now; Task 8 fills them.

- [ ] **Step 6.2 — Verify `.env` is gitignored**

```bash
git check-ignore -v .env
```

Expected: a line referencing `.gitignore`. If empty, add `.env` to `.gitignore` before continuing — this file holds secrets.

- [ ] **Step 6.3 — Replace the hard-coded `JWT_SECRET` with interpolation**

Edit `docker-compose.yaml`, line 11. Change:

```yaml
      JWT_SECRET: 'random string that is unique to every install - just type random characters here!'
```

to:

```yaml
      JWT_SECRET: ${JWT_SECRET:?JWT_SECRET must be set in .env}
```

The `:?` form makes Compose refuse to start with an empty secret — a guardrail against ever shipping the placeholder to a real deploy.

- [ ] **Step 6.4 — Wire OpenAI / LiteLLM env vars**

In the same `postiz` service, find line 85 (`OPENAI_API_KEY: ''`). Replace it with:

```yaml
      OPENAI_API_KEY: ${OPENAI_API_KEY:-}
      OPENAI_BASE_URL: ${OPENAI_BASE_URL:-}
```

The `:-` form lets these stay empty (Postiz will then run with the copilot disabled) without breaking Compose.

- [ ] **Step 6.5 — Sanity-check the rendered config**

```bash
docker compose config --quiet && echo "compose file OK"
docker compose config | grep -E "(JWT_SECRET|OPENAI_)" 
```

Expected: no errors, and the `OPENAI_*` lines show the values from `.env` (or empty strings).

> **What we did NOT change & why:**
> - `ports: "4007:5000"` — Postiz' container listens on **5000**, not 3000. Leaving it.
> - `BACKEND_INTERNAL_URL: http://localhost:3000` — that's the *intra-container* loopback for the Nest API. Don't touch.
> - Temporal stack — untouched; it's required since v2.12.0.
> - Volumes (`postgres-volume`, `postiz-redis-data`, `postiz-config`, `postiz-uploads`) — Compose-managed named volumes living under OrbStack's data dir, which is already on the SSD. No bind-mount needed.

---

## Task 7 — (Path B only) Add a LiteLLM Sidecar

**Files:**
- Create: `/Users/jonc/code/postiz-docker-compose/docker-compose.litellm.yaml`
- Create: `/Volumes/RESEARCH_SSD/postiz/litellm/config.yaml`

Skip this task entirely if you chose Path A (commercial).

- [ ] **Step 7.1 — Write the LiteLLM config**

```bash
cat > /Volumes/RESEARCH_SSD/postiz/litellm/config.yaml <<'EOF'
model_list:
  - model_name: gpt-4o-mini            # what Postiz will request
    litellm_params:
      model: ollama_chat/llama3.2:3b   # what LiteLLM actually calls
      api_base: http://host.docker.internal:11434
  - model_name: gpt-4o
    litellm_params:
      model: ollama_chat/llama3.2:3b
      api_base: http://host.docker.internal:11434

litellm_settings:
  drop_params: true       # ignore OpenAI-only params Ollama doesn't accept
  set_verbose: false

general_settings:
  master_key: sk-litellm-local-dev    # the "API key" Postiz will send
EOF
```

The `model_name` field is what clients ask for; LiteLLM rewrites the call to Ollama. Both `gpt-4o` and `gpt-4o-mini` are aliased so Postiz' built-in model selector finds something whatever it picks.

- [ ] **Step 7.2 — Write an overlay compose file**

```bash
cat > /Users/jonc/code/postiz-docker-compose/docker-compose.litellm.yaml <<'EOF'
services:
  litellm:
    image: ghcr.io/berriai/litellm:main-stable
    container_name: postiz-litellm
    restart: unless-stopped
    command: ["--config", "/app/config.yaml", "--port", "4000"]
    ports:
      - "4000:4000"
    volumes:
      - /Volumes/RESEARCH_SSD/postiz/litellm/config.yaml:/app/config.yaml:ro
    networks:
      - postiz-network
    extra_hosts:
      - "host.docker.internal:host-gateway"   # belt-and-braces for OrbStack/Linux

networks:
  postiz-network:
    external: false
EOF
```

The overlay joins the same `postiz-network` so Postiz can reach it as `http://litellm:4000`.

- [ ] **Step 7.3 — Point Postiz at LiteLLM via `.env`**

Edit `/Users/jonc/code/postiz-docker-compose/.env`:

```bash
OPENAI_API_KEY=sk-litellm-local-dev
OPENAI_BASE_URL=http://litellm:4000/v1
```

> **Caveat:** Whether `OPENAI_BASE_URL` is honored depends on Postiz' internal OpenAI client. The official `openai-node` SDK reads it; the Postiz config-reference page does not list it explicitly. If Task 9 reveals the copilot still hits api.openai.com, see Step 9.4 fallback (DNS override at the daemon level).

---

## Task 8 — Bring Up the Stack

**Files:** none

- [ ] **Step 8.1 — Pull images first (lets you watch the download separately from boot)**

```bash
cd /Users/jonc/code/postiz-docker-compose
docker compose pull
```

Expected: 7 images pulled (Postiz, Postgres 17, Postgres 16, Redis 7.2, ES 7.17, Temporal auto-setup, Temporal admin-tools, Temporal UI, Spotlight). Total ~3–4 GB. All written to `/Volumes/RESEARCH_SSD/orbstack/data`.

- [ ] **Step 8.2 — Start the core stack (Path A)**

```bash
docker compose up -d
docker compose ps
```

Expected: 9 containers `Up`. The `postiz` container's `Up` state requires Postgres + Redis healthchecks to pass first (~15–30 s).

- [ ] **Step 8.3 — Start with the LiteLLM overlay (Path B only)**

```bash
docker compose \
  -f docker-compose.yaml \
  -f docker-compose.litellm.yaml \
  up -d
docker compose -f docker-compose.yaml -f docker-compose.litellm.yaml ps
```

Expected: 10 containers `Up`, including `postiz-litellm`. For brevity, set `export COMPOSE_FILE=docker-compose.yaml:docker-compose.litellm.yaml` in your shell so subsequent `docker compose` calls pick up both files automatically.

- [ ] **Step 8.4 — Tail logs until you see the ready banner**

```bash
docker compose logs -f postiz
```

Expected: a line containing `listening on :5000` or `Nest application successfully started`. `Ctrl-C` to detach.

---

## Task 9 — Verify Each Service

**Files:** none (HTTP probes)

- [ ] **Step 9.1 — Frontend loads**

```bash
curl -fsS -o /dev/null -w "frontend: %{http_code}\n" http://localhost:4007
```

Expected: `200`. Then open `http://localhost:4007` in your browser — you should see the Postiz signup screen.

- [ ] **Step 9.2 — Backend healthcheck**

```bash
curl -fsS http://localhost:4007/api/health 2>&1 | head -c 400
```

Expected: a JSON `{"status":"ok"...}`-ish response. A 404 is also acceptable on some builds; what you must NOT see is a 502/504.

- [ ] **Step 9.3 — Temporal UI loads**

Open `http://localhost:8080`. Expected: Temporal dashboard with namespace `default`. If it 502s, the Temporal stack is still bootstrapping (Elasticsearch takes 30–60 s on first run).

- [ ] **Step 9.4 — (Path B) LiteLLM round-trips to Ollama**

```bash
curl -fsS http://localhost:4000/v1/chat/completions \
  -H "Authorization: Bearer sk-litellm-local-dev" \
  -H "Content-Type: application/json" \
  -d '{"model":"gpt-4o-mini","messages":[{"role":"user","content":"reply with the word: ok"}]}' \
  | head -c 400
```

Expected: a JSON response containing `"content":"ok"` (or close). If the response is an error, check `docker compose logs litellm` — most failures are wrong model name in `config.yaml` (must match `ollama list` exactly).

**Fallback for Path B if Postiz ignores `OPENAI_BASE_URL`:** Add a DNS override so Postiz' calls to `api.openai.com` resolve to LiteLLM. In `docker-compose.yaml` under the `postiz` service add:

```yaml
    extra_hosts:
      - "api.openai.com:host-gateway"
```

then run `docker compose run --rm postiz cat /etc/hosts | grep openai` to confirm. This forces the OpenAI SDK's default base URL to hit LiteLLM regardless of env-var handling.

- [ ] **Step 9.5 — Confirm data is landing on the external SSD**

```bash
docker volume ls --filter name=postiz-docker-compose
docker volume inspect postiz-docker-compose_postgres-volume --format '{{.Mountpoint}}'
ls -la /Volumes/RESEARCH_SSD/orbstack/data/docker/volumes/ | head
```

Expected: the mountpoint path lives under `/Volumes/RESEARCH_SSD/orbstack/data/...` (because the symlink is transparent). Roughly mirrored in the last `ls`.

---

## Task 10 — First-Run Setup in the UI

**Files:** none (browser actions)

- [ ] **Step 10.1 — Register the first account**

Open `http://localhost:4007`. Because `DISABLE_REGISTRATION` is `false` in the compose file (line 17), you can create the first user. After creating it, you may later flip that flag to `true` and `docker compose up -d --force-recreate postiz` to lock the install down to a single user.

- [ ] **Step 10.2 — Test the AI copilot**

Inside Postiz, start a new post and use the "AI write" / copilot feature. If you see a response, the `OPENAI_API_KEY` (Path A) or LiteLLM bridge (Path B) is wired correctly. If it errors silently, `docker compose logs -f postiz` will tell you whether the call is hitting OpenAI or being refused.

- [ ] **Step 10.3 — (Optional) Add social-media credentials**

Use the in-UI integrations panel rather than the env vars where possible. For OAuth callbacks to work, the social provider must accept `http://localhost:4007/...` as a redirect URI; X (Twitter), TikTok, and some others **require** a public HTTPS URL — for those, either skip until you set up a tunnel (Cloudflare Tunnel / `cloudflared`) or set them via env vars in `docker-compose.yaml` once you have them.

---

## Task 11 — zsh Quality-of-Life

**Files:**
- Modify: `~/.zshrc` (or `~/.config/zsh/aliases.zsh` if you split it)

- [ ] **Step 11.1 — Append a Postiz alias block**

Append to `~/.zshrc`:

```bash
# --- Postiz (OrbStack) ---
export POSTIZ_DIR="$HOME/code/postiz-docker-compose"
# Uncomment if you committed to Path B and want overlay every command:
# export COMPOSE_FILE="$POSTIZ_DIR/docker-compose.yaml:$POSTIZ_DIR/docker-compose.litellm.yaml"

alias pcd='cd "$POSTIZ_DIR"'
alias pup='(cd "$POSTIZ_DIR" && docker compose up -d)'
alias pdown='(cd "$POSTIZ_DIR" && docker compose down)'
alias pps='(cd "$POSTIZ_DIR" && docker compose ps)'
alias plogs='(cd "$POSTIZ_DIR" && docker compose logs -f --tail=200 postiz)'
alias plogsall='(cd "$POSTIZ_DIR" && docker compose logs -f --tail=100)'
alias prestart='(cd "$POSTIZ_DIR" && docker compose restart postiz)'
alias ppull='(cd "$POSTIZ_DIR" && docker compose pull && docker compose up -d)'
alias pexec='(cd "$POSTIZ_DIR" && docker compose exec postiz sh)'
alias ppsql='(cd "$POSTIZ_DIR" && docker compose exec postiz-postgres psql -U postiz-user -d postiz-db-local)'
```

Reload: `source ~/.zshrc`. `pup` / `pdown` / `plogs` / `ppsql` are the four you'll use most.

- [ ] **Step 11.2 — (Optional) `uv` venv for Postiz tooling scripts**

```bash
mkdir -p "$POSTIZ_DIR/scripts"
uv venv --python=3.12 "$POSTIZ_DIR/.venv"
# Activate with: source "$POSTIZ_DIR/.venv/bin/activate"
```

Use it for Python helper scripts (e.g. bulk-import posts, drive the REST API). Repo already has a `.venv/` directory listed in `ls -la`; this just standardizes its provenance.

---

## Task 12 — Operations Runbook

**Files:**
- Create: `/Volumes/RESEARCH_SSD/postiz/scripts/backup.sh`

- [ ] **Step 12.1 — Backup the Postgres volume**

```bash
cat > /Volumes/RESEARCH_SSD/postiz/scripts/backup.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
STAMP=$(date +%Y%m%d-%H%M%S)
OUT="/Volumes/RESEARCH_SSD/postiz/backups/postiz-db-$STAMP.sql.gz"
cd "$HOME/code/postiz-docker-compose"
docker compose exec -T postiz-postgres \
  pg_dump -U postiz-user -d postiz-db-local | gzip > "$OUT"
ls -lh "$OUT"
EOF
chmod +x /Volumes/RESEARCH_SSD/postiz/scripts/backup.sh
```

Run `/Volumes/RESEARCH_SSD/postiz/scripts/backup.sh` before any Postiz upgrade. Optionally wire it into a `launchd` agent or `cron` for nightly dumps.

- [ ] **Step 12.2 — Restore from a backup**

```bash
gunzip -c /Volumes/RESEARCH_SSD/postiz/backups/postiz-db-<STAMP>.sql.gz \
  | docker compose exec -T postiz-postgres psql -U postiz-user -d postiz-db-local
```

- [ ] **Step 12.3 — Upgrade Postiz**

```bash
cd "$POSTIZ_DIR"
/Volumes/RESEARCH_SSD/postiz/scripts/backup.sh   # ALWAYS back up first
docker compose pull
docker compose up -d
docker compose logs --tail=200 postiz
```

If you cross a major boundary (e.g. v2.11.x → v2.12.x), **read** https://docs.postiz.com/installation/migration first — the v2.12 jump added Temporal and required a one-time migration.

- [ ] **Step 12.4 — Stop the stack but keep data**

```bash
pdown               # docker compose down — preserves volumes
```

- [ ] **Step 12.5 — Nuke and start over (DANGEROUS — destroys all DB data)**

```bash
cd "$POSTIZ_DIR" && docker compose down -v
```

The `-v` flag removes named volumes. Only do this if you have a backup or are happy losing everything.

- [ ] **Step 12.6 — Reclaim image cache space**

```bash
docker image prune -f
docker builder prune -f
df -h /Volumes/RESEARCH_SSD
```

---

## Task 13 — Troubleshooting Recipes

Each recipe = "if you see X, do Y." Match symptoms before changing anything.

- [ ] **Recipe 13.1 — `postiz` container in `Restarting` loop**

```bash
docker compose logs --tail=200 postiz
```

Common causes:
1. `JWT_SECRET` empty — Step 6.3 not done. Fix `.env`, then `docker compose up -d --force-recreate postiz`.
2. Postgres healthcheck failing — `docker compose logs postiz-postgres` will show port conflicts or out-of-disk. Free space on `/Volumes/RESEARCH_SSD`.
3. Temporal unreachable — `docker compose logs temporal` shows ES startup errors. Bump OrbStack RAM (Task 2.4); ES wants ≥ 512 MB headroom.

- [ ] **Recipe 13.2 — `localhost:4007` returns "connection refused"**

```bash
docker compose ps postiz                 # is it Up?
docker compose port postiz 5000          # what host port is mapped?
lsof -nP -iTCP:4007 -sTCP:LISTEN          # is something else holding it?
```

If Postiz is `Up` but no host port, edit the `ports:` block. If a non-Docker process holds 4007, either kill it or change the host side (e.g. `"5007:5000"`).

- [ ] **Recipe 13.3 — `host.docker.internal` doesn't resolve from a container**

Diagnose:
```bash
docker run --rm alpine sh -c 'apk add --no-cache bind-tools >/dev/null && nslookup host.docker.internal'
```

If it fails, use your Mac's LAN IP everywhere it appears:
```bash
ipconfig getifaddr en0   # or en1 on some setups
```
Update `OPENAI_BASE_URL` / LiteLLM `api_base` to `http://<that-ip>:11434` and `docker compose up -d --force-recreate`.

- [ ] **Recipe 13.4 — AI copilot says "no model available" or 401**

Path A: confirm `OPENAI_API_KEY` is set and valid (`echo $OPENAI_API_KEY` in container: `docker compose exec postiz env | grep OPENAI`).

Path B: prove LiteLLM works from inside Postiz' container:
```bash
docker compose exec postiz sh -c \
  'apk add --no-cache curl >/dev/null 2>&1 || apt-get install -y curl >/dev/null 2>&1; curl -fsS -H "Authorization: Bearer sk-litellm-local-dev" http://litellm:4000/v1/models'
```
Expected: JSON listing `gpt-4o-mini` etc. If `Could not resolve host: litellm`, the overlay file wasn't loaded — re-run with both `-f` files (Step 8.3) and check `docker network inspect postiz-docker-compose_postiz-network`.

If LiteLLM works but Postiz still fails, apply the Step 9.4 fallback (`extra_hosts: api.openai.com:host-gateway`).

- [ ] **Recipe 13.5 — Temporal UI shows "no connection"**

```bash
docker compose logs temporal | tail -50
```

Wait 60 s on first boot — `auto-setup` runs the schema migration on the first call. If it's still down after 2 min, restart in order: `docker compose restart temporal-elasticsearch temporal-postgresql temporal`.

- [ ] **Recipe 13.6 — Disk filling up unexpectedly**

```bash
df -h /Volumes/RESEARCH_SSD
docker system df
du -sh /Volumes/RESEARCH_SSD/orbstack/data/docker/{image,volumes,containers,overlay2} 2>/dev/null
```

Usual culprits: Spotlight monitor accumulating events (delete its volume — it's a debug tool), or old image layers (`docker image prune -af`).

- [ ] **Recipe 13.7 — Mac sleep / SSD unmount kills containers**

OrbStack's data-dir symlink breaks the moment `/Volumes/RESEARCH_SSD` unmounts. Symptom: `docker ps` hangs or all containers are gone after sleep.

Fix:
1. In **System Settings → Energy → "Put hard disks to sleep when possible"**, turn OFF.
2. Make sure the SSD is set to "always available" in Disk Utility (don't eject manually).
3. If the dir does unmount: `osascript -e 'quit app "OrbStack"'`, remount the SSD, then relaunch OrbStack.

- [ ] **Recipe 13.8 — Forgot the JWT secret / sessions invalid**

Rotating `JWT_SECRET` invalidates **every** existing session — users must log in again. To rotate:
```bash
NEW=$(openssl rand -base64 48 | tr -d '\n=' | tr '/+' '_-')
sed -i.bak "s|^JWT_SECRET=.*|JWT_SECRET=$NEW|" "$POSTIZ_DIR/.env"
docker compose up -d --force-recreate postiz
```

---

## Appendix A — Port Reference

| Port | Service | Container | Where defined |
|------|---------|-----------|----------------|
| 4007 | Postiz frontend (proxies to backend) | `postiz` | `docker-compose.yaml:116` (`4007:5000`) |
| 8080 | Temporal Web UI | `temporal-ui` | `docker-compose.yaml:249` |
| 7233 | Temporal gRPC | `temporal` | `docker-compose.yaml:204` |
| 8969 | Spotlight (dev monitor) | `spotlight` | `docker-compose.yaml:162` |
| 4000 | LiteLLM (Path B only) | `litellm` | `docker-compose.litellm.yaml` |
| 11434 | Ollama (host process) | n/a | system, not Docker |

Postgres (5432) and Redis (6379) are **not exposed to the host** in the production compose; the dev compose (`docker-compose.dev.yaml`) does expose them — only use that file for local hacking with `pgAdmin` / `RedisInsight`.

## Appendix B — Environment Variable Quick Reference

Required for any boot:

| Var | Value here |
|-----|-----------|
| `JWT_SECRET` | from `openssl rand` (Task 5.1) |
| `DATABASE_URL` | `postgresql://postiz-user:postiz-password@postiz-postgres:5432/postiz-db-local` (leave default) |
| `REDIS_URL` | `redis://postiz-redis:6379` (leave default) |
| `MAIN_URL` / `FRONTEND_URL` | `http://localhost:4007` |
| `NEXT_PUBLIC_BACKEND_URL` | `http://localhost:4007/api` |
| `BACKEND_INTERNAL_URL` | `http://localhost:3000` (intra-container; do **not** change) |
| `TEMPORAL_ADDRESS` | `temporal:7233` |
| `IS_GENERAL` | `true` (self-host) |
| `DISABLE_REGISTRATION` | `false` initially, flip to `true` after first signup |
| `RUN_CRON` | `true` |
| `STORAGE_PROVIDER` | `local` (uploads land in the `postiz-uploads` named volume → external SSD) |

AI-related:

| Var | Path A (commercial) | Path B (Ollama via LiteLLM) |
|-----|---------------------|------------------------------|
| `OPENAI_API_KEY` | your real OpenAI key | `sk-litellm-local-dev` (matches LiteLLM `master_key`) |
| `OPENAI_BASE_URL` | unset | `http://litellm:4000/v1` (+ Step 9.4 fallback if ignored) |

## Appendix C — Files Created or Modified

```
/Users/jonc/code/postiz-docker-compose/
├── .env                              [created — secrets, gitignored]
├── docker-compose.yaml               [modified — JWT/OPENAI env interpolation]
├── docker-compose.litellm.yaml       [created — Path B only]
└── dev/
    ├── MacBookProM4-setup.md         [pre-existing — Grok notes]
    └── plan_setup-config.md          [this file]

/Volumes/RESEARCH_SSD/postiz/
├── secrets.env                       [created — JWT, OPENAI_API_KEY master copy]
├── config/                           [reserved — future postiz.env mount]
├── backups/                          [created — pg_dump targets]
├── litellm/
│   └── config.yaml                   [Path B — Ollama model aliases]
└── scripts/
    └── backup.sh                     [created — Postgres backup]

~/.zshrc                              [modified — pup/pdown/plogs aliases]
```

## Appendix D — One-Page Cheat Sheet

```bash
pup        # start everything
pps        # status
plogs      # tail Postiz log
pdown      # stop (keep data)
ppull      # pull latest image + restart
ppsql      # open psql shell on the DB
ppath_a()  { sed -i.bak 's|^OPENAI_BASE_URL=.*|OPENAI_BASE_URL=|' "$POSTIZ_DIR/.env"; (cd "$POSTIZ_DIR" && docker compose up -d --force-recreate postiz); }
ppath_b()  { sed -i.bak 's|^OPENAI_BASE_URL=.*|OPENAI_BASE_URL=http://litellm:4000/v1|' "$POSTIZ_DIR/.env"; (cd "$POSTIZ_DIR" && docker compose -f docker-compose.yaml -f docker-compose.litellm.yaml up -d); }
```

Browse to: http://localhost:4007 (Postiz) · http://localhost:8080 (Temporal) · http://localhost:8969 (Spotlight, dev only).
