# Postiz + Multi-Stack User Manual

> **What this is:** the long-form, walk-me-through-everything manual for the customized Postiz fork and its sibling stacks (Dockge, n8n, Stirling-PDF, searcharvester) running on OrbStack with bulk storage on an external SSD.
>
> **When to use:** start here if you're installing fresh, returning after a long break, or troubleshooting. For an at-a-glance overview, see [README.md](../README.md). For the architectural reasoning and decision log, see [`dev/plan_devops_orbstack-multi-swap_20260526.md`](../dev/plan_devops_orbstack-multi-swap_20260526.md).

---

## Table of contents

- [Part I — First-time installation](#part-i--first-time-installation)
- [Part II — First login & admin setup](#part-ii--first-login--admin-setup)
- [Part III — AI provider configuration](#part-iii--ai-provider-configuration)
- [Part IV — Social-media provider configuration (Postiz)](#part-iv--social-media-provider-configuration-postiz)
- [Part V — Day-to-day operations](#part-v--day-to-day-operations)
- [Part VI — Managing stacks with Dockge](#part-vi--managing-stacks-with-dockge)
- [Part VII — Managing containers with Lazydocker](#part-vii--managing-containers-with-lazydocker)
- [Part VIII — Backup, restore, upgrade](#part-viii--backup-restore-upgrade)
- [Part IX — Customization recipes](#part-ix--customization-recipes)
- [Part X — Troubleshooting](#part-x--troubleshooting)
- [Appendix A — URL & port quick reference](#appendix-a--url--port-quick-reference)
- [Appendix B — Environment variable reference](#appendix-b--environment-variable-reference)
- [Appendix C — File inventory](#appendix-c--file-inventory)
- [Appendix D — Command cheat sheet](#appendix-d--command-cheat-sheet)
- [Appendix E — Glossary](#appendix-e--glossary)

---

## System overview

```
                   ┌────────────────────────────────────────────────────────┐
                   │                  Your Mac (Apple Silicon)              │
                   │                                                        │
   Browsers ──────▶│  :4007 Postiz  :5001 Dockge  :5678 n8n  :8080 Stirling │
                   │  :18080 Temporal UI         :9762 searcharvester UI    │
                   │                                                        │
                   │  ┌──────────────────────────────────────────────────┐  │
                   │  │              OrbStack VM                         │  │
                   │  │  ┌──────────────────────────────────────┐        │  │
                   │  │  │ postiz stack (10 containers)         │        │  │
                   │  │  │   postiz, postiz-postgres,           │        │  │
                   │  │  │   postiz-redis, postiz-litellm,      │◀───────┼──┼─ ollama (host :11434)
                   │  │  │   spotlight, temporal × 5            │        │  │
                   │  │  └──────────────────────────────────────┘        │  │
                   │  │  ┌────────────┐ ┌────────────┐ ┌─────────────┐   │  │
                   │  │  │  dockge    │ │ stirling   │ │     n8n     │   │  │
                   │  │  └────────────┘ └────────────┘ └─────────────┘   │  │
                   │  │  ┌──────────────────────────────────────┐        │  │
                   │  │  │ searcharvester (4 containers)        │        │  │
                   │  │  └──────────────────────────────────────┘        │  │
                   │  └──────────────────────────────────────────────────┘  │
                   │                                                        │
                   │  Data lives on:  /Volumes/RESEARCH_SSD (external)      │
                   └────────────────────────────────────────────────────────┘
```

**The 5 stacks:**

| # | Stack | What it does | URL |
|---|-------|--------------|-----|
| 1 | Postiz | Social-media scheduling + AI copilot | http://localhost:4007 |
| 2 | Dockge | Web UI for managing all stacks | http://localhost:5001 |
| 3 | n8n | Workflow automation | http://localhost:5678 |
| 4 | Stirling-PDF | Local PDF toolkit | http://localhost:8080 |
| 5 | searcharvester | Self-hosted search + research for LLMs | http://localhost:9762 |

**Cross-stack glue:**
- LiteLLM gateway (inside Postiz stack, on :4000) — provides one OpenAI-compatible endpoint that routes to OpenAI / Anthropic / Gemini / OpenRouter / Fireworks / local Ollama. Other stacks point at it via `host.docker.internal:4000`.
- All persistent data on the external SSD via OrbStack's data-dir symlink.

---

## Part I — First-time installation

> Skip this whole part if your machine already has the stack running (your `docker ps` already shows the 17 containers).

### I.1 — Hardware prerequisites

| Requirement | Why | How to verify |
|-------------|-----|---------------|
| Apple Silicon Mac (M-series) | ARM64 images, Rosetta for the occasional x86 image | `uname -m` returns `arm64` |
| ≥ 16 GB RAM (24 GB recommended) | OrbStack VM + Postiz + sibling stacks; ES + Postgres are hungry | `sysctl -n hw.memsize` |
| External SSD, APFS, ≥ 100 GB free | Stack images alone ~5 GB; volumes grow | `df -h /Volumes/<your-ssd>` |
| Stable USB-C / Thunderbolt connection to the SSD | Sleep-wake drops are the #1 cause of stack outages | Reseat if it's drop-prone |

### I.2 — Software prerequisites

#### I.2.1 — OrbStack

```bash
brew install --cask orbstack
open -a OrbStack
# Click through the setup wizard. Allow privileged helper.
orb status      # expect: Running
```

Recommended settings in OrbStack (menu bar icon → Settings):
- **System → CPU:** 8–10 cores
- **System → Memory:** 12–16 GiB
- **System → Rosetta:** on (only matters for the occasional x86 image)
- **General → Start at login:** ON (so stacks come back after reboot)

#### I.2.2 — Ollama (optional, for local LLM fallback)

```bash
brew install ollama
brew services start ollama
ollama --version
```

Set `OLLAMA_MODELS` to point at the SSD (in `~/.zshrc`):

```bash
export OLLAMA_MODELS=/Volumes/<your-ssd>/ai_caches/ollama_models
```

`source ~/.zshrc`. Then pull a small starter model:

```bash
ollama pull llama3.2:3b      # ~2 GB
ollama list
```

#### I.2.3 — Lazydocker (optional but recommended)

```bash
brew install lazydocker
```

### I.3 — Move OrbStack data to the external SSD

By default OrbStack stores its VM (containing all Docker images + volumes) at `~/Library/Group Containers/HUAQ24HBR6.dev.orbstack/data`. To use the external SSD:

```bash
SRC="$HOME/Library/Group Containers/HUAQ24HBR6.dev.orbstack/data"
DST="/Volumes/<your-ssd>/orbstack/data"
mkdir -p "$(dirname "$DST")"

# 1. Stop OrbStack cleanly
orb stop
osascript -e 'quit app "OrbStack"'
while pgrep -x OrbStack >/dev/null; do sleep 1; done

# 2. Move existing data if any
if [ -d "$SRC" ] && [ ! -L "$SRC" ]; then
  rsync -aH --info=progress2 "$SRC"/ "$DST"/
  mv "$SRC" "$SRC.bak-$(date +%s)"
fi

# 3. Replace with symlink
ln -s "$DST" "$SRC"

# 4. Verify
ls -la "$SRC"     # should print: data -> /Volumes/<your-ssd>/orbstack/data
```

Relaunch OrbStack:

```bash
open -a OrbStack
sleep 5
orb status
docker info | grep "Docker Root Dir"   # remains /var/lib/docker (that's *inside* the VM)
```

Once stable, delete the `*.bak-*` directory created in step 2.

### I.4 — Create the SSD directory layout

```bash
# Postiz operational data (out-of-repo)
mkdir -p /Volumes/<your-ssd>/postiz/{config,backups,litellm,scripts}

# Multi-stack root (used in Part I.7 onward)
mkdir -p /Volumes/<your-ssd>/stacks
```

### I.5 — Clone this repo

```bash
mkdir -p ~/code
cd ~/code
git clone <your-fork-url> postiz-docker-compose
cd postiz-docker-compose
```

### I.6 — Generate secrets and write `.env`

```bash
cp .env.example .env
chmod 600 .env

# JWT_SECRET — Postiz session signing
JWT=$(openssl rand -base64 48 | tr -d '\n=' | tr '/+' '_-')

# LITELLM_MASTER_KEY — bearer Postiz sends to LiteLLM
LMK="sk-litellm-local-$(openssl rand -hex 8)"

# Patch them into .env
sed -i.bak \
  -e "s|^JWT_SECRET=.*|JWT_SECRET=$JWT|" \
  -e "s|^LITELLM_MASTER_KEY=.*|LITELLM_MASTER_KEY=$LMK|" \
  .env && rm .env.bak

# Verify
grep -E "^(JWT_SECRET|LITELLM_MASTER_KEY)=" .env | head -2
```

**Master copy on the SSD:**

```bash
cp .env /Volumes/<your-ssd>/postiz/secrets.env
chmod 600 /Volumes/<your-ssd>/postiz/secrets.env
```

If you ever lose `~/code/postiz-docker-compose/.env`, restore from the SSD master copy.

### I.7 — Write the LiteLLM model list

```bash
cat > /Volumes/<your-ssd>/postiz/litellm/config.yaml <<'YAML'
model_list:
  # OpenAI direct
  - model_name: gpt-4o-mini
    litellm_params: { model: openai/gpt-4o-mini, api_key: os.environ/OPENAI_API_KEY }
  - model_name: gpt-4o
    litellm_params: { model: openai/gpt-4o, api_key: os.environ/OPENAI_API_KEY }
  - model_name: gpt-3.5-turbo
    litellm_params: { model: openai/gpt-3.5-turbo, api_key: os.environ/OPENAI_API_KEY }
  - model_name: o3-mini
    litellm_params: { model: openai/o3-mini, api_key: os.environ/OPENAI_API_KEY }

  # Anthropic direct (verify exact IDs at api.anthropic.com)
  - model_name: claude-haiku
    litellm_params: { model: anthropic/claude-haiku-4-5-20251001, api_key: os.environ/ANTHROPIC_API_KEY }
  - model_name: claude-sonnet
    litellm_params: { model: anthropic/claude-sonnet-4-6, api_key: os.environ/ANTHROPIC_API_KEY }

  # Google Gemini direct
  - model_name: gemini-flash
    litellm_params: { model: gemini/gemini-2.0-flash, api_key: os.environ/GEMINI_API_KEY }
  - model_name: gemini-pro
    litellm_params: { model: gemini/gemini-2.5-pro, api_key: os.environ/GEMINI_API_KEY }

  # OpenRouter (single key, many models)
  - model_name: openrouter-claude
    litellm_params: { model: openrouter/anthropic/claude-sonnet-4, api_key: os.environ/OPENROUTER_API_KEY }

  # Fireworks (fast OSS inference)
  - model_name: fireworks-llama-70b
    litellm_params: { model: fireworks_ai/accounts/fireworks/models/llama-v3p1-70b-instruct, api_key: os.environ/FIREWORKS_API_KEY }

  # Local Ollama (always available, no key needed)
  - model_name: local-fast
    litellm_params: { model: ollama_chat/llama3.2:3b, api_base: http://host.docker.internal:11434 }
  - model_name: local-quality
    litellm_params: { model: ollama_chat/gemma4:latest, api_base: http://host.docker.internal:11434 }

litellm_settings:
  drop_params: true
  fallbacks:
    - gpt-4o-mini: ["claude-haiku", "gemini-flash", "local-fast"]
    - gpt-4o:      ["claude-sonnet", "gemini-pro", "local-quality"]

general_settings:
  master_key: os.environ/LITELLM_MASTER_KEY
YAML
```

### I.8 — Confirm Postiz' `docker-compose.yaml` references your SSD path

The `docker-compose.litellm.yaml` overlay file expects `/Volumes/RESEARCH_SSD/postiz/litellm/config.yaml`. If your SSD has a different label, edit the path in that file:

```bash
sed -i.bak "s|/Volumes/RESEARCH_SSD|/Volumes/<your-ssd>|g" docker-compose.litellm.yaml
rm docker-compose.litellm.yaml.bak
```

### I.9 — First bring-up

```bash
cd ~/code/postiz-docker-compose

# Pull all images (~3-4 GB). If a macOS Keychain popup appears, click "Always Allow".
# If you accidentally dismiss it, see Troubleshooting X.3.
docker compose -f docker-compose.yaml -f docker-compose.litellm.yaml pull

# Start
docker compose -f docker-compose.yaml -f docker-compose.litellm.yaml up -d

# Watch the boot (orchestrator pre-compiles Temporal workflow bundles — ~30 s)
docker compose -f docker-compose.yaml -f docker-compose.litellm.yaml logs -f postiz
# Ctrl-C when you see "Workflow bundle created" lines stop streaming
```

### I.10 — First-time verification

```bash
# All containers should be Up
docker compose -f docker-compose.yaml -f docker-compose.litellm.yaml ps

# HTTP probes
curl -fsS -o /dev/null -w 'postiz:      %{http_code}\n' http://localhost:4007/      # 307
curl -fsS -o /dev/null -w 'temporal-ui: %{http_code}\n' http://localhost:18080/     # 200
curl -fsS                                              http://localhost:4000/health/liveliness  # "I'm alive!"

# LiteLLM end-to-end (no commercial keys yet → falls back to local-fast)
LMK=$(grep ^LITELLM_MASTER_KEY .env | cut -d= -f2)
curl -fsS http://localhost:4000/v1/chat/completions \
  -H "Authorization: Bearer $LMK" -H "Content-Type: application/json" \
  -d '{"model":"gpt-4o-mini","messages":[{"role":"user","content":"reply: ok"}],"max_tokens":10}' \
  | python3 -m json.tool | head -20
```

### I.11 — Set up the sibling stacks (Dockge, n8n, Stirling-PDF, searcharvester)

```bash
# Symlink Postiz into the multi-stack root (so Dockge sees it)
ln -s ~/code/postiz-docker-compose /Volumes/<your-ssd>/stacks/postiz

# Scaffold the other stack directories
mkdir -p /Volumes/<your-ssd>/stacks/{_dockge/data,n8n/data,stirling-pdf/data}
```

#### I.11.a — Dockge

```bash
cat > /Volumes/<your-ssd>/stacks/_dockge/docker-compose.yaml <<'YAML'
services:
  dockge:
    image: louislam/dockge:1
    container_name: dockge
    restart: unless-stopped
    ports: ["5001:5001"]
    environment:
      - DOCKGE_STACKS_DIR=/Volumes/<your-ssd>/stacks
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./data:/app/data
      - /Volumes/<your-ssd>/stacks:/Volumes/<your-ssd>/stacks
YAML

(cd /Volumes/<your-ssd>/stacks/_dockge && docker compose up -d)
```

#### I.11.b — Stirling-PDF

```bash
cat > /Volumes/<your-ssd>/stacks/stirling-pdf/docker-compose.yaml <<'YAML'
services:
  stirling-pdf:
    image: docker.stirlingpdf.com/stirlingtools/stirling-pdf:latest
    container_name: stirling-pdf
    restart: unless-stopped
    ports: ["8080:8080"]
    environment:
      - DOCKER_ENABLE_SECURITY=false
      - LANGS=en_US,en_GB
    volumes:
      - ./data/trainingData:/usr/share/tessdata
      - ./data/extraConfigs:/configs
      - ./data/customFiles:/customFiles
      - ./data/logs:/logs
      - ./data/pipeline:/pipeline
YAML

(cd /Volumes/<your-ssd>/stacks/stirling-pdf && docker compose up -d)
```

#### I.11.c — n8n

```bash
N8N_KEY=$(openssl rand -hex 32)
cat > /Volumes/<your-ssd>/stacks/n8n/.env <<EOF
TZ=America/New_York
N8N_ENCRYPTION_KEY=$N8N_KEY
EOF
chmod 600 /Volumes/<your-ssd>/stacks/n8n/.env

cat > /Volumes/<your-ssd>/stacks/n8n/docker-compose.yaml <<'YAML'
services:
  n8n:
    image: docker.n8n.io/n8nio/n8n:latest
    container_name: n8n
    restart: unless-stopped
    ports: ["5678:5678"]
    environment:
      - N8N_HOST=localhost
      - N8N_PORT=5678
      - N8N_PROTOCOL=http
      - WEBHOOK_URL=http://localhost:5678/
      - N8N_RUNNERS_ENABLED=true
      - GENERIC_TIMEZONE=${TZ:-America/New_York}
      - N8N_DIAGNOSTICS_ENABLED=false
      - N8N_PERSONALIZATION_ENABLED=false
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY:?Set N8N_ENCRYPTION_KEY in .env}
    volumes:
      - ./data:/home/node/.n8n
YAML

(cd /Volumes/<your-ssd>/stacks/n8n && docker compose up -d)
```

#### I.11.d — searcharvester

```bash
cd /Volumes/<your-ssd>/stacks
git clone --depth=1 https://github.com/vakovalskii/searcharvester.git
cd searcharvester

# Wire its .env so its Hermes /research agent uses Postiz' LiteLLM
LMK=$(grep ^LITELLM_MASTER_KEY ~/code/postiz-docker-compose/.env | cut -d= -f2)
cat > .env <<EOF
HERMES_UID=$(id -u)
HERMES_GID=$(id -g)
OPENAI_API_KEY=$LMK
OPENAI_BASE_URL=http://host.docker.internal:4000/v1
ANTHROPIC_API_KEY=
GEMINI_API_KEY=
OPENROUTER_API_KEY=
VITE_API_URL=http://localhost:8000
EOF
chmod 600 .env

# Generate a rotated secret_key
SEC=$(openssl rand -hex 32)
cp config.example.yaml config.yaml
sed -i.bak "s|CHANGE_ME_TO_RANDOM_SECRET_KEY_32_CHARS_OR_MORE|$SEC|" config.yaml
rm config.yaml.bak

docker compose up -d
```

### I.12 — Final verification

```bash
docker ps --format 'table {{.Names}}\t{{.Status}}' | head -20
# Expect 17 containers across 5 stacks.

# Open all UIs
for u in 4007 5001 5678 8080 18080 8000 8999 9762; do
  printf '  %5d -> %s\n' "$u" "$(curl -fsS -o /dev/null -w '%{http_code}' --max-time 4 http://localhost:$u/ 2>/dev/null)"
done
```

Then jump to **Part II** to log in to each service.

---

## Part II — First login & admin setup

### II.1 — Postiz (http://localhost:4007)

1. Open http://localhost:4007 in your browser. You'll be redirected to `/auth`.
2. Click **Sign up** (the first signup becomes the owner).
3. Fill email + password (use a strong password — this account controls all your social-media credentials).
4. Click through the initial onboarding.
5. **Lock down to a single user (recommended):**
   - Edit `docker-compose.yaml`, find `DISABLE_REGISTRATION: 'false'` (line 17) and change to `'true'`.
   - `docker compose -f docker-compose.yaml -f docker-compose.litellm.yaml up -d --force-recreate postiz`
   - Now no further signups are allowed.

### II.2 — Dockge (http://localhost:5001)

1. Open http://localhost:5001.
2. "Create your admin" screen → pick a username + password. **Save these in your password manager** — there is no recovery flow.
3. After login you'll see the **Stacks** view. Each subdirectory of `/Volumes/<ssd>/stacks/` that contains a `docker-compose.y*ml` appears as a card.
4. The first time, you may need to click each card → **Edit Stack** → **Save** to "adopt" it (Dockge claims management of an external-started stack).

### II.3 — n8n (http://localhost:5678)

1. Open http://localhost:5678. You'll see "Set up owner account".
2. Email + first/last name + password.
3. Click through the next screens (n8n asks about your role/use case — answer honestly or skip).
4. The dashboard appears. To validate it works, create a small workflow:
   - "Add first step" → **Manual Trigger** → **HTTP Request** → URL `http://host.docker.internal:4000/health/liveliness` → **Test workflow**. You should see `"I'm alive!"` in the response panel.

### II.4 — Stirling-PDF (http://localhost:8080)

By default **no login required** (we set `DOCKER_ENABLE_SECURITY=false`). Anyone with access to localhost can use it. That's fine for a single-user Mac.

To enable login:
1. Edit `/Volumes/<ssd>/stacks/stirling-pdf/docker-compose.yaml` and change `DOCKER_ENABLE_SECURITY=false` to `true`.
2. `docker compose up -d --force-recreate stirling-pdf`.
3. First visit prompts you to create an admin account.

### II.5 — searcharvester

The frontend at http://localhost:9762 has **no login**. The API endpoints at http://localhost:8000 (`/search`, `/extract`, `/research`) are open on localhost. That's fine for a single-user Mac.

**Quick functional test:**

```bash
# Search via tavily-adapter
curl -fsS 'http://localhost:8000/search' \
  -H 'Content-Type: application/json' \
  -d '{"query":"docker compose best practices","max_results":3}' | jq '.results[0].title'

# Research (uses LLM — requires Postiz' LiteLLM to be running; may take 30-60 s)
curl -fsS 'http://localhost:8000/research' \
  -H 'Content-Type: application/json' \
  -d '{"query":"What is Postiz?"}' | head -c 500
```

### II.6 — Spotlight (http://localhost:8969)

No login. It's a developer error/event viewer that captures Postiz exceptions in real time. Useful when something's wrong; harmless to ignore otherwise.

### II.7 — Temporal UI (http://localhost:18080)

No login. Used to inspect Postiz' scheduled workflows. After you schedule a post in Postiz, you'll see workflow runs here.

---

## Part III — AI provider configuration

The LiteLLM gateway is the single point of LLM configuration. Add provider keys to **one** `.env` file (`~/code/postiz-docker-compose/.env`) and every stack that uses LiteLLM benefits.

### III.1 — Where to get keys

| Provider | Console URL | What you'll create |
|----------|-------------|---------------------|
| OpenAI | https://platform.openai.com/api-keys | API key (starts `sk-...`) |
| Anthropic | https://console.anthropic.com/settings/keys | API key |
| Google AI Studio | https://aistudio.google.com/apikey | API key |
| OpenRouter | https://openrouter.ai/keys | API key |
| Fireworks | https://fireworks.ai/account/api-keys | API key |

### III.2 — Add a key

```bash
cd ~/code/postiz-docker-compose

# 1. Edit .env — paste the key after the matching variable
$EDITOR .env

# 2. Restart LiteLLM so it picks up the new env
docker compose -f docker-compose.yaml -f docker-compose.litellm.yaml restart litellm

# 3. Verify
LMK=$(grep ^LITELLM_MASTER_KEY .env | cut -d= -f2)
curl -fsS -H "Authorization: Bearer $LMK" \
  http://localhost:4000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"gpt-4o-mini","messages":[{"role":"user","content":"reply: ok"}],"max_tokens":10}' \
  | python3 -m json.tool | grep '"model":'
# Should now say "model": "openai/gpt-4o-mini" instead of falling back to local-fast.
```

### III.3 — Edit the model list

The model list lives at `/Volumes/<ssd>/postiz/litellm/config.yaml`. Edit there, not in the repo.

**Add a model:** append to `model_list:`. Example — Anthropic's Opus:

```yaml
  - model_name: claude-opus
    litellm_params:
      model: anthropic/claude-opus-4-7
      api_key: os.environ/ANTHROPIC_API_KEY
```

**Change a fallback chain:** edit the `litellm_settings.fallbacks:` block.

**Apply changes:** `docker compose restart litellm`.

### III.4 — Add a local Ollama model

```bash
# 1. Pull the model
ollama pull qwen2.5:7b

# 2. Add to LiteLLM config (model_list)
#    Append:
cat >> /Volumes/<ssd>/postiz/litellm/config.yaml <<'YAML'

  - model_name: local-coder
    litellm_params:
      model: ollama_chat/qwen2.5:7b
      api_base: http://host.docker.internal:11434
YAML

# 3. Restart LiteLLM
docker compose -f docker-compose.yaml -f docker-compose.litellm.yaml restart litellm

# 4. Test
LMK=$(grep ^LITELLM_MASTER_KEY ~/code/postiz-docker-compose/.env | cut -d= -f2)
curl -fsS -H "Authorization: Bearer $LMK" -H "Content-Type: application/json" \
  http://localhost:4000/v1/chat/completions \
  -d '{"model":"local-coder","messages":[{"role":"user","content":"hi"}]}' | python3 -m json.tool
```

### III.5 — Disable LiteLLM (use OpenAI directly)

Edit `.env`:

```bash
OPENAI_API_KEY=sk-<your-real-openai-key>
OPENAI_BASE_URL=               # blank — Postiz will hit api.openai.com directly
```

Then bring Postiz up **without** the LiteLLM overlay:

```bash
docker compose down                    # default — postiz only, no overlay
docker compose up -d
```

`OPENAI_API_KEY` in `.env` is now used as the real key by the Postiz container's openai-node SDK.

---

## Part IV — Social-media provider configuration (Postiz)

Postiz supports a long list of platforms (X, LinkedIn, Reddit, Discord, Facebook, Threads, YouTube, TikTok, Pinterest, Mastodon, Slack, Bluesky, etc.). Configuration follows a consistent pattern.

### IV.1 — General pattern

Each provider needs an **OAuth app** registered with that platform's developer console. You get back a Client ID + Client Secret. You give those to Postiz, which then runs the OAuth dance on your behalf.

**Two ways to give credentials to Postiz:**

| Method | When |
|--------|------|
| **A. Settings UI** | Postiz UI → Settings → Integrations. Click the platform → paste credentials → save. **Preferred** — keys never touch the compose file. |
| **B. `docker-compose.yaml` env vars** | Bulk-configure many platforms at once, or for reproducible IaC. Find the platform's `*_CLIENT_ID:` / `*_CLIENT_SECRET:` lines (around lines 35–66) and fill in. Then `docker compose up -d --force-recreate postiz`. |

### IV.2 — Common OAuth callback URLs

Most platforms ask for a redirect/callback URL when you set up the app. For local Postiz on this Mac:

```
http://localhost:4007/integrations/social/<provider>
```

Replace `<provider>` with what Postiz' Settings UI shows (it'll tell you the exact URL when you click into a provider).

**The catch:** several platforms require **HTTPS** for the callback (X, TikTok, sometimes Discord). For those, either:
- Set up a tunnel: `brew install cloudflared && cloudflared tunnel --url http://localhost:4007` → use the `*.trycloudflare.com` URL it prints
- Or set `MAIN_URL` to a public hostname you control and run behind a reverse proxy.

Postiz' [docs](https://docs.postiz.com) have per-platform setup guides — follow those for the actual OAuth app creation.

### IV.3 — Quick test after adding a provider

1. Postiz UI → **Channels** → **Add channel** → pick the platform → click through the OAuth dialog.
2. If successful, the channel appears in your channel list.
3. **Schedule a test post** (set the date 5 minutes ahead) → wait → check the platform.
4. Visit http://localhost:18080 (Temporal UI) to see the workflow run; useful when posts mysteriously fail.

---

## Part V — Day-to-day operations

### V.1 — After a Mac reboot (or sleep-wake with SSD drop)

```bash
# 1. Confirm SSD is mounted
diskutil list | grep RESEARCH_SSD     # should show the volume

# 2. Start OrbStack (if not auto-started)
orb start

# 3. Bring up everything (idempotent — won't touch running containers)
# Two-file Postiz:
cd ~/code/postiz-docker-compose && docker compose -f docker-compose.yaml -f docker-compose.litellm.yaml up -d

# Sibling stacks:
for d in /Volumes/<ssd>/stacks/{_dockge,stirling-pdf,n8n,searcharvester}/; do
  (cd "$d" && docker compose up -d)
done
```

Or set `COMPOSE_FILE` once and use `pup`:

```bash
pup          # if zsh aliases from README installed
```

### V.2 — Check status

| Command | Shows |
|---------|-------|
| `docker ps` | All running containers (every stack) |
| `dups` *(alias)* | `compose ps` for every stack in one screen |
| `lazydocker` | TUI: containers, logs, stats, exec |
| Dockge UI (`:5001`) | Per-stack cards with status indicators |

### V.3 — Tail logs

```bash
# One container
docker logs -f postiz

# One stack (all services)
cd ~/code/postiz-docker-compose && docker compose -f docker-compose.yaml -f docker-compose.litellm.yaml logs -f --tail=100

# Just one service in a stack
docker compose -f docker-compose.yaml -f docker-compose.litellm.yaml logs -f postiz-litellm

# With the alias
plogs       # postiz only
plogsall    # all postiz-stack services
```

### V.4 — Restart one container without disturbing the stack

```bash
docker restart postiz
# or via compose
docker compose -f docker-compose.yaml -f docker-compose.litellm.yaml restart postiz
```

### V.5 — Database access

```bash
# Postiz Postgres
ppsql                                                                # alias
# or
docker compose -f docker-compose.yaml -f docker-compose.litellm.yaml exec postiz-postgres \
  psql -U postiz-user -d postiz-db-local

# Redis
docker compose -f docker-compose.yaml -f docker-compose.litellm.yaml exec postiz-redis redis-cli
# > KEYS *
# > GET <key>

# Temporal admin tools
docker compose -f docker-compose.yaml -f docker-compose.litellm.yaml exec temporal-admin-tools sh
# tctl --address temporal:7233 namespace describe default
```

### V.6 — Get a shell inside a container

```bash
docker compose -f docker-compose.yaml -f docker-compose.litellm.yaml exec postiz sh
# or
pexec    # alias
```

### V.7 — Stop a stack temporarily

```bash
# Stop but keep data
cd ~/code/postiz-docker-compose && docker compose -f docker-compose.yaml -f docker-compose.litellm.yaml down

# In Dockge UI: click ⏹ Stop on the stack card

# Stop sibling stack:
(cd /Volumes/<ssd>/stacks/n8n && docker compose down)
```

The `restart: unless-stopped` policies respect this — containers stay down until you `up -d` again.

### V.8 — Stop EVERYTHING at end of day

```bash
# Brute method
docker stop $(docker ps -q)

# Cleaner — per stack
for d in /Volumes/<ssd>/stacks/{_dockge,stirling-pdf,n8n,searcharvester}/; do (cd "$d" && docker compose down); done
cd ~/code/postiz-docker-compose && docker compose -f docker-compose.yaml -f docker-compose.litellm.yaml down

# Quit OrbStack to release the VM RAM
orb stop
```

---

## Part VI — Managing stacks with Dockge

> Dockge URL: http://localhost:5001

### VI.1 — Dashboard layout

After login, you see the **Stacks** sidebar (left) and main panel (center).

- **Sidebar:** one row per detected stack (5 total). Status indicator: green dot = healthy/running, gray = stopped, orange = degraded.
- **Main panel:** when no stack is selected, shows summary stats. Click a stack to drill in.

### VI.2 — Stack detail view

Click a stack name in the sidebar. You see:

- **Top bar:** stack name, status, action buttons (▶ Start, ⏹ Stop, ↻ Restart, ⬇ Update, 🗑 Delete).
- **Tabs:**
  - **Services** — table of containers in the stack with per-container start/stop/restart/log links.
  - **Compose** — editable YAML view. Edit in browser, click **Save** then **Deploy** to apply.
  - **Logs** — tail of all services' logs combined, streaming live.
  - **Terminal** — shell into the Dockge container (rarely useful; use `pexec` etc.).

### VI.3 — Common Dockge actions

| Task | Action |
|------|--------|
| Start a stopped stack | Click stack → **▶ Start** |
| Stop a running stack | Click stack → **⏹ Stop** |
| Restart a service (one container) | **Services** tab → row → **↻** icon |
| Pull new images & restart | Click stack → **⬇ Update** (runs `docker compose pull && up -d`) |
| Edit compose YAML | Click stack → **Compose** tab → edit → **Save** → **Deploy** |
| See live logs | Click stack → **Logs** tab (auto-scrolls) |
| Delete a stack | **🗑 Delete** — keeps named volumes by default; doesn't delete the compose dir |

### VI.4 — Adopting an externally-started stack

If you brought a stack up via CLI before Dockge knew about it, the card shows "Unknown" status. To adopt:

1. Click the stack.
2. Open **Compose** tab.
3. Click **Save** (without changing anything). Dockge will now track the stack's state.

### VI.5 — Creating a new stack from Dockge

1. Sidebar → **Compose** (top of sidebar).
2. **+ New Stack** button.
3. Name it (becomes the subdirectory name in `STACKS_DIR`).
4. Paste a compose YAML.
5. Optionally add an `.env`.
6. **Deploy**.

Dockge creates `/Volumes/<ssd>/stacks/<name>/docker-compose.yaml` for you. Equivalent to writing it by hand.

### VI.6 — Dockge limitations to know

- **No multi-file compose support in the UI.** Dockge merges only the single `docker-compose.yaml` (or `compose.yaml`) in the stack dir. The Postiz LiteLLM overlay (`docker-compose.litellm.yaml`) is **invisible to Dockge** — when you click Start/Stop in Dockge for Postiz, it controls only the main file. To control LiteLLM via Dockge, either merge the overlay into the main file or manage LiteLLM with the CLI.
- **No secrets management.** `.env` files are read but not edited in the UI. Edit via your IDE or the **Compose** tab's separate env editor.
- **No RBAC.** One admin account; share carefully.

---

## Part VII — Managing containers with Lazydocker

> Launch: `lazydocker` (or alias `dlog`)

### VII.1 — Pane layout

Four panes:
- **Project** (top-left): list of compose projects (= stacks).
- **Containers** (middle-left): containers in the current project.
- **Images / Volumes / Networks** (bottom-left): tabbed.
- **Main** (right): logs, stats, config, or env depending on which item is selected.

### VII.2 — Keyboard cheat sheet

| Key | Action |
|-----|--------|
| `←` / `→` | Cycle panes |
| `↑` / `↓` | Move within a list |
| `Enter` | Open a focused-view of the selection |
| `Tab` | Cycle main pane (logs / stats / config / env / top) |
| `r` | Restart the selected container |
| `s` | Stop |
| `S` | Start |
| `d` | Remove |
| `e` | Edit (will exec into) — or `x` for the menu |
| `m` | View memory stats |
| `?` | Help / full keymap |
| `q` | Quit |

### VII.3 — Common Lazydocker patterns

- **"Why is X slow?"** → focus container → Tab to **stats** → see CPU/RAM/network in real time.
- **"What just broke?"** → focus container → Tab to **logs** → scroll up.
- **"Quick `docker exec sh`"** → focus container → `x` → choose **Exec**.
- **"Why does my volume eat 50 GB?"** → bottom-left → switch to **Volumes** tab → focus volume → see filesystem usage.

---

## Part VIII — Backup, restore, upgrade

### VIII.1 — Postiz Postgres backup

```bash
# Manual one-shot
STAMP=$(date +%Y%m%d-%H%M%S)
docker compose -f docker-compose.yaml -f docker-compose.litellm.yaml exec -T postiz-postgres \
  pg_dump -U postiz-user -d postiz-db-local | gzip > /Volumes/<ssd>/postiz/backups/postiz-db-$STAMP.sql.gz

# Verify
ls -lh /Volumes/<ssd>/postiz/backups/postiz-db-$STAMP.sql.gz
```

For automated nightly backups, create `~/Library/LaunchAgents/com.local.postiz-backup.plist` running a cron-like schedule. Skeleton in `/Volumes/<ssd>/postiz/scripts/backup.sh` (already created during install).

### VIII.2 — Postiz Postgres restore

```bash
# 1. Stop Postiz to release the DB
docker compose -f docker-compose.yaml -f docker-compose.litellm.yaml stop postiz

# 2. (Recommended) drop and recreate the DB to avoid merge conflicts
docker compose -f docker-compose.yaml -f docker-compose.litellm.yaml exec -T postiz-postgres \
  psql -U postiz-user -d postgres -c "DROP DATABASE \"postiz-db-local\"; CREATE DATABASE \"postiz-db-local\";"

# 3. Restore
gunzip -c /Volumes/<ssd>/postiz/backups/postiz-db-<STAMP>.sql.gz | \
  docker compose -f docker-compose.yaml -f docker-compose.litellm.yaml exec -T postiz-postgres \
  psql -U postiz-user -d postiz-db-local

# 4. Start Postiz
docker compose -f docker-compose.yaml -f docker-compose.litellm.yaml start postiz
```

### VIII.3 — n8n backup (SQLite + workflows + credentials)

```bash
tar czf /Volumes/<ssd>/postiz/backups/n8n-$(date +%Y%m%d).tar.gz \
  -C /Volumes/<ssd>/stacks/n8n data
```

**Critical:** without `N8N_ENCRYPTION_KEY` (in `stacks/n8n/.env`) the backup is unusable. Back up that file too — copy to `/Volumes/<ssd>/postiz/secrets.env` or a password manager.

### VIII.4 — n8n restore

```bash
cd /Volumes/<ssd>/stacks/n8n
docker compose down
rm -rf data
tar xzf /Volumes/<ssd>/postiz/backups/n8n-<DATE>.tar.gz
docker compose up -d
```

### VIII.5 — Stirling-PDF & searcharvester backup

Stirling-PDF: typically nothing — PDFs are processed transiently. The Settings → Export feature in the UI saves any custom config.

Searcharvester: just `config.yaml` and `.env`. Tiny — back up by copying to the SSD's `postiz/backups/`.

### VIII.6 — Upgrade a stack

```bash
# Always back up first for any stack with persisted data
/Volumes/<ssd>/postiz/scripts/backup.sh        # Postiz only

# Then:
cd <stack-dir>
docker compose pull
docker compose up -d
docker compose logs --tail=200 <main-service>
```

Or in Dockge: stack → **⬇ Update** button.

### VIII.7 — Cross-major-version Postiz upgrade

For Postiz major bumps (e.g., v2.11 → v2.12 which added Temporal), **read the upstream migration guide first**: https://docs.postiz.com/installation/migration. Failure to do so can corrupt the database.

---

## Part IX — Customization recipes

### IX.1 — Change a host port

Example: move Stirling-PDF from `:8080` to `:8081`:

```bash
$EDITOR /Volumes/<ssd>/stacks/stirling-pdf/docker-compose.yaml
# Change   - "8080:8080"
# To       - "8081:8080"   (host:container — container stays 8080)

cd /Volumes/<ssd>/stacks/stirling-pdf
docker compose up -d --force-recreate
```

### IX.2 — Pin image versions (predictability over freshness)

Replace `:latest` (or `:1`) with a specific tag:

```yaml
# Before
image: docker.n8n.io/n8nio/n8n:latest
# After
image: docker.n8n.io/n8nio/n8n:1.96.0
```

Check tags at https://hub.docker.com/ or the project's release page. Now `docker compose pull` only fetches that exact version.

### IX.3 — Switch Postiz from LiteLLM to direct OpenAI

In `~/code/postiz-docker-compose/.env`:

```diff
- OPENAI_BASE_URL=
+ OPENAI_BASE_URL=
- OPENAI_API_KEY=                        (blank)
+ OPENAI_API_KEY=sk-your-real-openai-key
```

(`OPENAI_BASE_URL` empty means: use the SDK default, which is `https://api.openai.com/v1`.)

Then bring Postiz up **without** the overlay:

```bash
docker compose down
docker compose up -d                     # one file, no LiteLLM
```

To go back:

```bash
docker compose down
# Restore LITELLM_MASTER_KEY as OPENAI_API_KEY and OPENAI_BASE_URL=http://litellm:4000/v1 in .env
docker compose -f docker-compose.yaml -f docker-compose.litellm.yaml up -d
```

### IX.4 — Add a custom Postiz environment variable

Anything Postiz accepts can be added to the `postiz:` service's `environment:` block in `docker-compose.yaml`. Example — set `API_LIMIT`:

```yaml
environment:
  # … existing entries …
  API_LIMIT: 60          # change from default 30
```

Then `docker compose up -d --force-recreate postiz`.

### IX.5 — Move stack data to a different SSD path

```bash
NEW_SSD_LABEL=NEW_SSD
OLD_SSD_LABEL=RESEARCH_SSD

# 1. Stop everything
docker stop $(docker ps -q)
orb stop

# 2. Move
rsync -aH --info=progress2 /Volumes/$OLD_SSD_LABEL/ /Volumes/$NEW_SSD_LABEL/

# 3. Fix symlinks
rm ~/Library/Group\ Containers/HUAQ24HBR6.dev.orbstack/data
ln -s /Volumes/$NEW_SSD_LABEL/orbstack/data ~/Library/Group\ Containers/HUAQ24HBR6.dev.orbstack/data

# 4. Update paths in compose files (find anywhere referring to OLD path)
grep -rl "/Volumes/$OLD_SSD_LABEL" ~/code/postiz-docker-compose /Volumes/$NEW_SSD_LABEL/stacks/ \
  | xargs sed -i.bak "s|/Volumes/$OLD_SSD_LABEL|/Volumes/$NEW_SSD_LABEL|g"
find ~/code/postiz-docker-compose /Volumes/$NEW_SSD_LABEL/stacks/ -name '*.bak' -delete

# 5. Update OLLAMA_MODELS env in ~/.zshrc
sed -i.bak "s|/Volumes/$OLD_SSD_LABEL|/Volumes/$NEW_SSD_LABEL|g" ~/.zshrc

# 6. Restart everything
orb start
source ~/.zshrc
```

### IX.6 — Rotate `JWT_SECRET` (logs out all Postiz users)

```bash
NEW=$(openssl rand -base64 48 | tr -d '\n=' | tr '/+' '_-')
sed -i.bak "s|^JWT_SECRET=.*|JWT_SECRET=$NEW|" ~/code/postiz-docker-compose/.env
rm ~/code/postiz-docker-compose/.env.bak
docker compose -f docker-compose.yaml -f docker-compose.litellm.yaml up -d --force-recreate postiz
```

### IX.7 — Cross-stack: n8n calling LiteLLM

In an n8n HTTP Request node:

```
URL:    http://host.docker.internal:4000/v1/chat/completions
Method: POST
Headers:
  Authorization: Bearer <LITELLM_MASTER_KEY value from postiz .env>
  Content-Type: application/json
Body (JSON):
  { "model": "gpt-4o-mini",
    "messages": [{ "role": "user", "content": "{{ $json.prompt }}" }] }
```

Now n8n workflows have the same multi-provider routing as Postiz.

### IX.8 — Add a new compose stack

```bash
mkdir -p /Volumes/<ssd>/stacks/<name>/data
cat > /Volumes/<ssd>/stacks/<name>/docker-compose.yaml <<'YAML'
services:
  <svc>:
    image: <image>:<tag>
    container_name: <name>
    restart: unless-stopped
    ports: ["<host_port>:<container_port>"]
    volumes:
      - ./data:<container_data_dir>
YAML

(cd /Volumes/<ssd>/stacks/<name> && docker compose up -d)
```

Then refresh Dockge — the new stack appears as a card.

---

## Part X — Troubleshooting

Each recipe lists the symptom first, then diagnosis, then fix.

### X.1 — Symptom: browser can't reach `http://localhost:<port>`

**Diagnose:**
```bash
# Is OrbStack alive?
orb status
# Is the container running?
docker ps --filter "publish=<port>"
# Can curl reach it?
curl -fsS -o /dev/null -w '%{http_code}\n' --max-time 5 http://localhost:<port>/
```

**Most common cause: external SSD disconnected.** If `diskutil list | grep RESEARCH_SSD` returns nothing, the SSD is gone — reseat the cable and follow X.2.

**Other causes:**
- Container died → `docker logs <container>` shows why; restart it.
- Chrome's HTTPS-First Mode upgraded localhost → Settings → Privacy & Security → "Always use secure connections" → off. Or use `http://127.0.0.1:<port>` instead of `localhost`.
- Port collision → another process owns the port (`lsof -nP -iTCP:<port> -sTCP:LISTEN`). Stop it or remap.

### X.2 — Symptom: external SSD unmounted; everything broken

**Diagnose:** `diskutil list | grep RESEARCH_SSD` returns nothing.

**Fix:**
1. Physically reseat the USB-C / Thunderbolt cable. Try a different port. Bypass any hubs.
2. Wait 5 s.
3. `diskutil list | grep RESEARCH_SSD` — should now show the volume.
4. `orb start` → wait 30 s → `orb status` says **Running**.
5. Bring stacks back:
   ```bash
   cd ~/code/postiz-docker-compose && docker compose -f docker-compose.yaml -f docker-compose.litellm.yaml up -d
   for d in /Volumes/<ssd>/stacks/{_dockge,stirling-pdf,n8n,searcharvester}/; do (cd "$d" && docker compose up -d); done
   ```

**Prevent:** System Settings → Battery → Options → "Put hard disks to sleep when possible" → **OFF**. Avoid bus-powered hubs for the SSD.

### X.3 — Symptom: macOS Keychain popup during `docker compose pull`

**Diagnose:** popup says "docker-credential-osxkeychain wants to use the 'Login Keychain'". If you dismiss it, pull fails with `error getting credentials - err: exit status 1, out: User canceled the operation. (-128)`.

**Fix (option 1, click-through):** click **Always Allow**. Done forever.

**Fix (option 2, remove credsStore):**
```bash
python3 -c "import json; p='/Users/$USER/.docker/config.json'; d=json.load(open(p)); d.pop('credsStore',None); json.dump(d,open(p,'w'),indent=2)"
docker compose pull
# If you want it back later, set credsStore back to "osxkeychain"
```

### X.4 — Symptom: OrbStack stuck on "Starting"

**Diagnose:** `orb status` reports `Starting` for more than 60 s; `docker ps` hangs.

**Fix:**
```bash
orb stop
sleep 3
orb start                          # NOT `open -a OrbStack` — start is more reliable
# wait 30 s
orb status                         # should say Running
```

If still stuck: check `~/.orbstack/log/vmgr.log` for actual errors; otherwise reboot.

### X.5 — Symptom: Postiz container restarts in a loop

**Diagnose:**
```bash
docker compose -f docker-compose.yaml -f docker-compose.litellm.yaml logs --tail=200 postiz
```

**Common causes:**
1. `JWT_SECRET` empty in `.env` → fix and `docker compose up -d --force-recreate postiz`.
2. Postgres or Redis still booting → wait 30 s, watch `docker compose logs postiz-postgres`.
3. Temporal unreachable → ES needs ≥ 512 MB headroom; bump OrbStack RAM (System → Memory) to 16 GiB.

### X.6 — Symptom: LiteLLM responds 401 / "model not found"

**Diagnose:**
```bash
LMK=$(grep ^LITELLM_MASTER_KEY ~/code/postiz-docker-compose/.env | cut -d= -f2)

# Is the master key right?
curl -fsS -H "Authorization: Bearer $LMK" http://localhost:4000/v1/models | jq -r .data[].id
# 401 → wrong key
# Empty / not in list → model alias missing in config.yaml

docker logs --tail=100 postiz-litellm
# Look for: "Provider API key not set" → upstream key missing in .env
```

**Fix:**
- Wrong master key: ensure `OPENAI_API_KEY` env in the Postiz container equals `LITELLM_MASTER_KEY` in `.env` (it does in this fork by default).
- Missing alias: edit `/Volumes/<ssd>/postiz/litellm/config.yaml`, add the model, `docker compose restart litellm`.
- Missing upstream key: paste real key into `.env`, restart litellm.

### X.7 — Symptom: AI copilot in Postiz UI says "Could not generate"

**Diagnose:** `docker logs -f postiz` shows the actual LLM call URL. If it's hitting `api.openai.com` directly, Postiz' SDK didn't pick up `OPENAI_BASE_URL`.

**Fix (rare — only if env-var route fails):**

```yaml
# Add to postiz service in docker-compose.yaml
extra_hosts:
  - "api.openai.com:host-gateway"
```

This forces api.openai.com to resolve to LiteLLM regardless of env-var behavior. Then `docker compose up -d --force-recreate postiz`.

### X.8 — Symptom: `docker compose ps` returns empty when invoked through symlink

**Diagnose:** you `cd /Volumes/<ssd>/stacks/postiz/` (a symlink to the real repo) and `docker compose ps` shows nothing — but containers ARE running.

**Cause:** project name defaults to the symlink directory's basename (`postiz`), not the target's (`postiz-docker-compose`). Two different "projects" as far as compose is concerned.

**Fix:** the `name:` field at the top of `docker-compose.yaml` (already set to `postiz-docker-compose` in this fork) pins the project name regardless of cwd. If you ever remove or change it, recreate via the canonical path:

```bash
cd ~/code/postiz-docker-compose         # not via the symlink
docker compose -f docker-compose.yaml -f docker-compose.litellm.yaml ps
```

### X.9 — Symptom: containers gone after Mac sleep/wake even though SSD is mounted

**Diagnose:** `docker compose ps` shows containers in `Exited (137)` state. 137 = SIGKILL (likely from OrbStack VM shutdown).

**Fix:**
```bash
# Bring everything back up
cd ~/code/postiz-docker-compose && docker compose -f docker-compose.yaml -f docker-compose.litellm.yaml up -d
for d in /Volumes/<ssd>/stacks/{_dockge,stirling-pdf,n8n,searcharvester}/; do (cd "$d" && docker compose up -d); done
```

The `restart: unless-stopped` policies handle most cases automatically; the line above catches the few that didn't.

### X.10 — Symptom: LiteLLM marked "unhealthy" but actually works

**Diagnose:**
```bash
docker inspect postiz-litellm --format='{{ .State.Health.Status }}'         # unhealthy
curl http://localhost:4000/health/liveliness                                # "I'm alive!"
```

**Cause:** older versions of `docker-compose.litellm.yaml` used `wget` for healthcheck, which isn't in the LiteLLM image.

**Fix:** ensure healthcheck uses Python (already patched in this fork):
```yaml
healthcheck:
  test: ["CMD-SHELL", "python -c \"import urllib.request,sys; urllib.request.urlopen('http://localhost:4000/health/liveliness', timeout=3); sys.exit(0)\" || exit 1"]
```

### X.11 — Symptom: n8n workflows lost after recreate

**Diagnose:** `stacks/n8n/data/` is empty or missing the `database.sqlite` file.

**Cause:** either:
- The volume mount line in `docker-compose.yaml` was edited wrong.
- `N8N_ENCRYPTION_KEY` was changed → workflows still on disk but credentials are unreadable.

**Fix:** restore from backup (Part VIII.4). **There is no other recovery for a lost encryption key** — credentials must be re-entered.

### X.12 — Symptom: Stirling-PDF crashes on big PDFs

**Diagnose:** `docker logs stirling-pdf` shows OutOfMemoryError or process killed.

**Fix:**
- Bump OrbStack RAM (Settings → System → Memory) to 16+ GiB.
- Or set a Java heap limit on the container:
  ```yaml
  environment:
    - JAVA_OPTS=-Xmx4g
  ```
- Clear `stacks/stirling-pdf/data/logs/` periodically (no rotation by default).

### X.13 — Symptom: searcharvester `/research` returns 500

**Diagnose:** `docker logs tavily-adapter` says "OPENAI_API_KEY not set" or "Hermes container failed to spawn".

**Fix:** confirm `stacks/searcharvester/.env` has `OPENAI_API_KEY=<litellm-master-key>` and `OPENAI_BASE_URL=http://host.docker.internal:4000/v1`, plus that LiteLLM is reachable:

```bash
docker exec tavily-adapter sh -c 'apk add --no-cache curl >/dev/null 2>&1; curl -fsS http://host.docker.internal:4000/health/liveliness'
```

If `host.docker.internal` doesn't resolve from inside the container, add to the searcharvester `docker-compose.yaml`:
```yaml
extra_hosts:
  - "host.docker.internal:host-gateway"
```

### X.14 — Symptom: Dockge doesn't see a stack you created from the CLI

**Diagnose:** the stack dir is missing from Dockge's sidebar.

**Fix:**
- Confirm Dockge can see the path: `docker exec dockge ls -la /Volumes/<ssd>/stacks/`. Should list all stack dirs.
- If your stack dir starts with `.` or isn't a real directory (e.g., a broken symlink), Dockge will skip it.
- Compose file must be named `docker-compose.yaml` or `compose.yaml` (or `.yml`).
- Refresh the Dockge browser tab; discovery polling is not instant.

### X.15 — Symptom: high RAM use on the Mac

**Diagnose:**
```bash
docker stats --no-stream
```

**Common culprits (sorted):**
1. **Elasticsearch (Temporal)** — bumps to ~500 MB easily. `ES_JAVA_OPTS=-Xms256m -Xmx256m` already pins it; lower further if needed.
2. **Postgres** — grows with db size. Vacuum after large deletes.
3. **Stirling-PDF (Java)** — see X.12.

**Fix:** in OrbStack Settings → System, the **Memory** slider caps total VM RAM. The OS pages other host apps out if it exceeds this.

---

## Appendix A — URL & port quick reference

| URL | Service | Notes |
|-----|---------|-------|
| http://localhost:4007 | **Postiz** | Main UI; redirects to `/auth` |
| http://localhost:4000 | LiteLLM gateway | Auth: `Bearer $LITELLM_MASTER_KEY` |
| http://localhost:5001 | **Dockge** | Stack manager |
| http://localhost:5678 | **n8n** | Workflow automation |
| http://localhost:8080 | **Stirling-PDF** | Local PDF toolkit |
| http://localhost:18080 | Temporal UI | Postiz workflow visibility (remapped from 8080) |
| http://localhost:8000 | searcharvester API (`/docs`) | OpenAPI spec at `/docs` |
| http://localhost:8999 | SearXNG (searcharvester) | Metasearch UI |
| http://localhost:9762 | searcharvester frontend | Research UI |
| http://localhost:8969 | Spotlight | Sentry-local debug viewer |
| http://localhost:11434 | Ollama (host process) | API for local models |

---

## Appendix B — Environment variable reference

### Postiz `.env`

| Variable | Required | Set by | Purpose |
|----------|:---:|---|---|
| `JWT_SECRET` | Yes | `openssl rand -base64 48 \| tr -d '\n=' \| tr '/+' '_-'` | Session signing |
| `LITELLM_MASTER_KEY` | Yes if using LiteLLM | `echo "sk-litellm-local-$(openssl rand -hex 8)"` | Bearer Postiz → LiteLLM |
| `OPENAI_API_KEY` | No | OpenAI console | Direct OpenAI or LiteLLM upstream |
| `OPENAI_BASE_URL` | No | Set to `http://litellm:4000/v1` for LiteLLM, blank for OpenAI direct | OpenAI SDK base URL override |
| `ANTHROPIC_API_KEY` | No | Anthropic console | Claude via LiteLLM |
| `GEMINI_API_KEY` | No | Google AI Studio | Gemini via LiteLLM |
| `OPENROUTER_API_KEY` | No | OpenRouter | Any-provider router via LiteLLM |
| `FIREWORKS_API_KEY` | No | Fireworks | OSS model inference via LiteLLM |

### n8n `stacks/n8n/.env`

| Variable | Required | Purpose |
|----------|:---:|---|
| `N8N_ENCRYPTION_KEY` | Yes | Encrypts stored OAuth creds — **losing it = losing them** |
| `TZ` | No | Timezone for cron/schedule nodes |

### searcharvester `stacks/searcharvester/.env`

| Variable | Required | Purpose |
|----------|:---:|---|
| `HERMES_UID` / `HERMES_GID` | Yes (`id -u` / `id -g`) | Owner UID/GID of files spawned containers create |
| `OPENAI_API_KEY` | For `/research` | LiteLLM master key (cross-stack pattern) |
| `OPENAI_BASE_URL` | For `/research` | `http://host.docker.internal:4000/v1` |
| `VITE_API_URL` | Yes | Where the frontend (browser) sends API calls |

---

## Appendix C — File inventory

### In this repo

```
.
├── README.md                                       # orientation + quickstart + index
├── .env.example                                    # template for .env
├── .env                                            # local, gitignored, secrets
├── docker-compose.yaml                             # postiz core
├── docker-compose.litellm.yaml                     # LiteLLM overlay
├── docker-compose.dev.yaml                         # dev-only (pgAdmin + RedisInsight)
├── dynamicconfig/                                  # Temporal dynamic config
├── docs/
│   └── user-manual.md                              # this file
└── dev/
    ├── MacBookProM4-setup.md                       # original Grok notes
    ├── plan_setup-config.md                        # initial setup plan
    └── plan_devops_orbstack-multi-swap_20260526.md # multi-stack plan + decision log
```

### Outside the repo, on the SSD

```
/Volumes/<ssd>/
├── orbstack/data/                                  # OrbStack VM (images + named volumes)
├── ai_caches/ollama_models/                        # Ollama model weights
├── postiz/
│   ├── secrets.env                                 # master copy of .env (chmod 600)
│   ├── litellm/config.yaml                         # LiteLLM model list + fallbacks
│   ├── backups/                                    # pg_dump targets
│   ├── scripts/backup.sh                           # backup helper
│   └── config/                                     # reserved
└── stacks/                                         # Dockge STACKS_DIR
    ├── _dockge/    (compose + data/)
    ├── n8n/        (compose + .env + data/)
    ├── postiz      -> /Users/jonc/code/postiz-docker-compose  (symlink)
    ├── searcharvester/  (cloned repo + .env + config.yaml)
    └── stirling-pdf/    (compose + data/)
```

### Aliases (in `~/.zshrc`, marked block)

See README section "zsh aliases" for the full block.

---

## Appendix D — Command cheat sheet

```bash
# ─── Postiz (two-file overlay) ───
pup                                                            # up -d
pdown                                                          # down (preserves data)
pps                                                            # compose ps
plogs                                                          # tail postiz logs
plogsall                                                       # tail all postiz-stack logs
prestart                                                       # restart postiz container
ppull                                                          # pull + up -d
pexec                                                          # exec sh into postiz
ppsql                                                          # psql into postiz Postgres
pllm                                                           # list LiteLLM model aliases

# ─── Across all stacks ───
dups                                                           # `compose ps` for each stack
dlog                                                           # lazydocker
scd                                                            # cd into STACKS_DIR
pcd                                                            # cd into POSTIZ_DIR

# ─── Open browser UIs ───
opostiz odockge on8n ostirling osearcharv otemporal

# ─── Raw docker ───
docker ps                                                      # all running containers
docker compose -f X.yaml -f Y.yaml up -d                       # multi-file bring-up
docker compose -f X.yaml -f Y.yaml down                        # multi-file teardown
docker compose ... logs -f <svc>                               # tail one service
docker compose ... exec <svc> <cmd>                            # one-shot exec
docker logs -f --tail=100 <container>                          # direct log tail
docker stats --no-stream                                       # CPU/mem snapshot
docker system df                                               # disk usage by type
docker image prune -af                                         # remove unused images
docker volume ls                                               # list volumes
docker network ls                                              # list networks

# ─── Disk / SSD ───
df -h /Volumes/<ssd>                                           # SSD free space
diskutil list                                                  # all disks
du -sh /Volumes/<ssd>/orbstack/data                            # OrbStack VM size

# ─── LiteLLM ───
LMK=$(grep ^LITELLM_MASTER_KEY ~/code/postiz-docker-compose/.env | cut -d= -f2)
curl -H "Authorization: Bearer $LMK" http://localhost:4000/v1/models
curl -H "Authorization: Bearer $LMK" -H "Content-Type: application/json" \
  http://localhost:4000/v1/chat/completions \
  -d '{"model":"gpt-4o-mini","messages":[{"role":"user","content":"hi"}]}'

# ─── Secrets generation ───
openssl rand -base64 48 | tr -d '\n=' | tr '/+' '_-'           # JWT_SECRET style
openssl rand -hex 32                                           # N8N_ENCRYPTION_KEY style
echo "sk-litellm-local-$(openssl rand -hex 8)"                 # LITELLM_MASTER_KEY style
```

---

## Appendix E — Glossary

| Term | Definition |
|------|------------|
| **Stack** | One compose project = one bridge network = one row in Dockge. Bounded by a directory containing a `docker-compose.yaml`. |
| **STACKS_DIR** | The directory Dockge scans (`/Volumes/<ssd>/stacks/`). |
| **Overlay (compose)** | A second `-f file.yaml` whose contents merge into the main compose. Used here for LiteLLM. |
| **LiteLLM master key** | Bearer token LiteLLM accepts from clients. Different from upstream provider API keys. |
| **host.docker.internal** | DNS alias OrbStack provides inside containers, resolving to the Mac host. |
| **Temporal** | Durable workflow engine Postiz uses for scheduled posting. Required since Postiz v2.12. |
| **Spotlight** | Sentry's local-only error viewer. Bundled in Postiz compose for dev visibility. |
| **OrbStack** | Drop-in Docker Desktop replacement for Apple Silicon. Hosts a Linux VM running Docker. |
| **Dockge** | Web UI for managing docker-compose stacks. Discovers stacks by scanning `STACKS_DIR`. |
| **Lazydocker** | TUI for docker — single-binary, no service to run. |
| **Restart policy** | `always` / `unless-stopped` / `on-failure` / `no`. Controls container behavior on exit and on daemon restart. |
| **Healthcheck** | A test in the compose file that decides if a container is `healthy` / `unhealthy`. Affects `depends_on: { condition: service_healthy }` gating. |

---

*Last revised 2026-05-26. Live state of the machine is the operational source of truth; this document is a synthesis kept in sync via the same git history.*
