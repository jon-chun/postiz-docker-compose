# Postiz — OrbStack / Multi-Provider AI Edition

A customized fork of [`gitroomhq/postiz-docker-compose`](https://github.com/gitroomhq/postiz-docker-compose), hardened for **macOS + OrbStack** with bulk storage on an external SSD and an **AI gateway** ([LiteLLM](https://github.com/BerriAI/litellm)) in front of Postiz' copilot that routes calls across OpenAI · Anthropic · Google Gemini · OpenRouter · Fireworks · local Ollama, with automatic fallback when keys are missing.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        Browser :4007                                    │
│                            │                                            │
│                            ▼                                            │
│  ┌──────────┐  ┌───────────────────┐  ┌───────────┐  ┌──────────────┐   │
│  │ Postiz   │─▶│ LiteLLM gateway   │─▶│ OpenAI /  │  │ Ollama       │   │
│  │  (:5000) │  │  (:4000)          │  │ Anthropic │  │  (host:11434)│   │
│  │          │  │  14-model alias + │  │ /Gemini / │◀─│   llama3.2,  │   │
│  │          │  │  fallback chain   │  │  …        │  │   gemma4 …   │   │
│  └────┬─────┘  └───────────────────┘  └───────────┘  └──────────────┘   │
│       │                                                                 │
│       ├──▶ PostgreSQL 17 (postiz-postgres)                              │
│       ├──▶ Redis 7.2     (postiz-redis)                                 │
│       └──▶ Temporal 1.28 ─┬─ Elasticsearch 7.17                         │
│           UI: :18080      └─ Postgres 16                                │
│                                                                         │
│                    Spotlight (dev): :8969                               │
└─────────────────────────────────────────────────────────────────────────┘
```

## 📖 Documentation map

| If you want to… | Read |
|------------------|------|
| **Install and use this** — step-by-step, end-to-end | [`docs/user-manual.md`](docs/user-manual.md) |
| Understand **why** the architecture is shaped the way it is — decisions, port matrix, integration patterns | [`dev/plan_devops_orbstack-multi-swap_20260526.md`](dev/plan_devops_orbstack-multi-swap_20260526.md) |
| Skim **what's here** + a 5-minute quickstart | this file (below) |

---

## What's different from upstream

| Topic | Upstream | This fork |
|-------|----------|-----------|
| LLM provider | OpenAI key only | LiteLLM proxy — 14-model router across 6 providers + local Ollama, automatic fallback |
| Secrets | Inline in `docker-compose.yaml` | Externalized to `.env` interpolation (gitignored) |
| Temporal UI port | `:8080` | `:18080` (frees `:8080` for sibling Docker stacks like Stirling-PDF) |
| Restart policies | `always` on 3 services, **none** on 6 | `unless-stopped` added to the 6 unpolicied services |
| Storage | OrbStack default location | OrbStack VM data symlinked to external SSD; AI keys/configs likewise |
| Compose files | 1 | 2 (`docker-compose.yaml` + `docker-compose.litellm.yaml` overlay) |
| Project name | Auto-derived (drifts via symlink) | Pinned via `name:` field |
| Documentation | Brief README | This README + comprehensive user manual + devops plan |

---

## Prerequisites

| Requirement | Why | Verify |
|-------------|-----|--------|
| macOS Apple Silicon (M-series) | Tested M4 | `uname -m` → `arm64` |
| OrbStack ≥ 2.1 | Container runtime | `orb status` → `Running` |
| External SSD (APFS, ≥ 100 GB free) | Hosts images + DB volumes | `df -h /Volumes/<your-ssd>` |
| Ollama ≥ 0.20 *(optional)* | Local LLM fallback | `curl http://localhost:11434/api/tags` |

Full prerequisite + installation walkthrough: [user manual Part I](docs/user-manual.md#part-i--first-time-installation).

---

## Quickstart (5 minutes)

```bash
# 1. Clone
git clone https://github.com/<you>/postiz-docker-compose.git
cd postiz-docker-compose

# 2. Create .env from template
cp .env.example .env
chmod 600 .env

# 3. Generate JWT + LiteLLM master key
JWT=$(openssl rand -base64 48 | tr -d '\n=' | tr '/+' '_-')
LMK="sk-litellm-local-$(openssl rand -hex 8)"
sed -i.bak -e "s|^JWT_SECRET=.*|JWT_SECRET=$JWT|" -e "s|^LITELLM_MASTER_KEY=.*|LITELLM_MASTER_KEY=$LMK|" .env && rm .env.bak

# 4. Write the LiteLLM model list to the SSD
#    (see docs/user-manual.md §I.7 for the full config.yaml — copy/paste it)
mkdir -p /Volumes/<your-ssd>/postiz/litellm
$EDITOR /Volumes/<your-ssd>/postiz/litellm/config.yaml

# 5. Bring up (~3-4 GB image pull on first run)
docker compose -f docker-compose.yaml -f docker-compose.litellm.yaml up -d
```

Open **http://localhost:4007** → sign up → done.

> **Mac Keychain popup during pull?** Click **Always Allow**. Or [fix permanently](docs/user-manual.md#x3--symptom-macos-keychain-popup-during-docker-compose-pull).

---

## URLs at a glance

| URL | Service |
|-----|---------|
| http://localhost:4007 | **Postiz** (main UI) |
| http://localhost:4000 | LiteLLM gateway (`Bearer $LITELLM_MASTER_KEY`) |
| http://localhost:18080 | Temporal UI (workflow visibility) |
| http://localhost:8969 | Sentry Spotlight (dev) |
| http://localhost:11434 | Ollama (host process) |

Companion stacks (set up via [user manual Part I.11](docs/user-manual.md#i11--set-up-the-sibling-stacks-dockge-n8n-stirling-pdf-searcharvester)):

| URL | Service |
|-----|---------|
| http://localhost:5001 | Dockge — web UI to manage all stacks |
| http://localhost:5678 | n8n — workflow automation |
| http://localhost:8080 | Stirling-PDF — local PDF toolkit |
| http://localhost:9762 | searcharvester — research / search |

---

## Common operations

```bash
# Start
docker compose -f docker-compose.yaml -f docker-compose.litellm.yaml up -d

# Stop (preserve data)
docker compose -f docker-compose.yaml -f docker-compose.litellm.yaml down

# Logs
docker compose -f docker-compose.yaml -f docker-compose.litellm.yaml logs -f postiz

# Upgrade (always back up first!)
/Volumes/<ssd>/postiz/scripts/backup.sh
docker compose -f docker-compose.yaml -f docker-compose.litellm.yaml pull
docker compose -f docker-compose.yaml -f docker-compose.litellm.yaml up -d
```

Postiz-specific zsh aliases (auto-include the LiteLLM overlay): see [user manual Appendix D](docs/user-manual.md#appendix-d--command-cheat-sheet). The block is already in `~/.zshrc` on the install machine.

---

## Configuration

### `.env` essentials

| Variable | Required | Source |
|----------|:--------:|--------|
| `JWT_SECRET` | ✅ | `openssl rand -base64 48 \| tr -d '\n=' \| tr '/+' '_-'` |
| `LITELLM_MASTER_KEY` | ✅ (if using LiteLLM) | `echo "sk-litellm-local-$(openssl rand -hex 8)"` |
| `OPENAI_API_KEY` | optional | https://platform.openai.com/api-keys |
| `ANTHROPIC_API_KEY` | optional | https://console.anthropic.com/ |
| `GEMINI_API_KEY` | optional | https://aistudio.google.com/apikey |
| `OPENROUTER_API_KEY` | optional | https://openrouter.ai/ |
| `FIREWORKS_API_KEY` | optional | https://fireworks.ai/ |

**Leave commercial keys blank** and LiteLLM transparently falls back to local Ollama. Full reference: [user manual Part III](docs/user-manual.md#part-iii--ai-provider-configuration) + [Appendix B](docs/user-manual.md#appendix-b--environment-variable-reference).

### Social-media keys

In-UI (Settings → Integrations) is preferred, or via the `*_CLIENT_ID:` / `*_CLIENT_SECRET:` env vars in `docker-compose.yaml` (lines 35–66). OAuth callback URL is `http://localhost:4007/integrations/social/<provider>`. Several providers (X, TikTok) require HTTPS — use a tunnel like `cloudflared`. See [user manual Part IV](docs/user-manual.md#part-iv--social-media-provider-configuration-postiz).

---

## Storage layout

```
~/code/postiz-docker-compose/                # this repo
└── .env                                     # local secrets, gitignored

/Volumes/<your-ssd>/                         # external SSD, all bulk data
├── orbstack/data/                           # OrbStack VM (Docker images + named volumes)
├── ai_caches/ollama_models/                 # Ollama models
└── postiz/
    ├── secrets.env                          # master copy of .env (chmod 600)
    ├── litellm/config.yaml                  # model list + fallback chain
    ├── backups/                             # pg_dump targets
    └── scripts/backup.sh                    # backup helper
```

Walkthrough for moving OrbStack data to the SSD: [user manual §I.3](docs/user-manual.md#i3--move-orbstack-data-to-the-external-ssd).

---

## The big-three troubleshooting recipes

> Full troubleshooting library (15 recipes): [user manual Part X](docs/user-manual.md#part-x--troubleshooting).

### 1. Browser can't reach `http://localhost:<port>` after a sleep-wake

Almost always: **external SSD has disconnected**. OrbStack's VM data lives on it; without the SSD the whole stack dies silently.

```bash
diskutil list | grep RESEARCH_SSD      # if empty, SSD is unplugged
# Reseat the USB-C / Thunderbolt cable, then:
orb start
sleep 30
docker compose -f docker-compose.yaml -f docker-compose.litellm.yaml up -d
```

Prevent: **System Settings → Battery → Options → "Put hard disks to sleep when possible" → OFF**. Don't use a bus-powered USB hub for the SSD.

### 2. OrbStack stuck "Starting"

```bash
orb stop
sleep 3
orb start                              # NOT `open -a OrbStack`
```

### 3. Postiz copilot returns 401 or "unknown model"

```bash
LMK=$(grep ^LITELLM_MASTER_KEY .env | cut -d= -f2)
curl -fsS -H "Authorization: Bearer $LMK" http://localhost:4000/v1/models | jq -r .data[].id
```

If the model isn't listed, edit `/Volumes/<ssd>/postiz/litellm/config.yaml` and `docker compose restart litellm`. If LiteLLM is unreachable, the issue is upstream (see recipe 1).

---

## Multi-stack management

This repo is one of several compose stacks that run on the same host (n8n, Stirling-PDF, searcharvester, …). They're managed together via:

- **Dockge** web UI at http://localhost:5001 (cards per stack, start/stop/edit-YAML in browser)
- **Lazydocker** TUI (`brew install lazydocker`, then `lazydocker`)

Architecture, port plan, cross-stack integration patterns: [`dev/plan_devops_orbstack-multi-swap_20260526.md`](dev/plan_devops_orbstack-multi-swap_20260526.md). Step-by-step setup: [user manual §I.11](docs/user-manual.md#i11--set-up-the-sibling-stacks-dockge-n8n-stirling-pdf-searcharvester) and [Part VI](docs/user-manual.md#part-vi--managing-stacks-with-dockge).

---

## What this fork does *not* change

- Postiz' source code — we run the upstream `ghcr.io/gitroomhq/postiz-app:latest` image.
- Temporal stack composition or versions.
- Postgres / Redis defaults.
- The `dynamicconfig/` Temporal tuning files.
- Postiz licensing or branding.

Upstream upgrades therefore work cleanly via `docker compose pull`.

---

## License

Postiz is licensed under [AGPL-3.0](LICENSE). This fork preserves that license unchanged.

## Links

- **Upstream:** https://github.com/gitroomhq/postiz-docker-compose
- **Postiz docs:** https://docs.postiz.com
- **Postiz migration guide:** https://docs.postiz.com/installation/migration
- **LiteLLM:** https://docs.litellm.ai
- **OrbStack:** https://orbstack.dev
- **Ollama:** https://ollama.com
- **Dockge:** https://github.com/louislam/dockge
- **Lazydocker:** https://github.com/jesseduffield/lazydocker
