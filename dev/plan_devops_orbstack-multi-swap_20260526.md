# Multi-Stack OrbStack DevOps Plan — 2026-05-26

> **Source-of-truth synthesis** of the 2026-05-25/26 setup session. Captures the as-built state, decisions with rationale and pros/cons, and the operations runbook for managing multiple `docker compose` stacks on this MacBook Pro M4 with OrbStack and an external OWC 4 TB SSD.
>
> Audience: future-you, or a new operator who walks into this machine cold.

---

## 0. Scope

**In scope:**
- Five compose stacks living side-by-side: **Postiz**, **searcharvester**, **Stirling-PDF**, **n8n**, **Dockge** (the manager itself).
- A multi-provider AI gateway (**LiteLLM**) shared across stacks.
- All bulk storage (Docker images, container volumes, AI model weights, application data) on `/Volumes/RESEARCH_SSD`.
- A stack manager (**Dockge** web UI on `:5001`) + a terminal complement (**Lazydocker**).
- Restart policies and port allocations that let stacks run concurrently or be individually toggled.

**Out of scope:**
- Public exposure (no reverse proxy / tunnels / HTTPS).
- Multi-host orchestration (Kubernetes, Swarm).
- CI/CD pipelines.

---

## 1. Executive summary

Five stacks, ~17 containers, ~16 GB on the external SSD. Brought up incrementally with explicit per-stack isolation (each compose project has its own bridge network). One shared concern — the **LiteLLM** sidecar in the Postiz stack — exposes an OpenAI-compatible endpoint on `:4000` that any other stack can call through `host.docker.internal:4000` for unified LLM routing across providers (OpenAI, Anthropic, Google, OpenRouter, Fireworks, local Ollama). Stacks are managed end-to-end via **Dockge** (web UI) and **Lazydocker** (TUI).

**Documentation structure (post-2026-05-26):** the README is a tight orientation + 5-min quickstart (~250 lines), [`docs/user-manual.md`](../docs/user-manual.md) is the comprehensive walk-through (~1500 lines, Parts I–X + 5 appendices), and *this* plan is the architectural decision log. The repo lives at https://github.com/jon-chun/postiz-docker-compose as a fork of `gitroomhq/postiz-docker-compose`; first published 2026-05-27 as commit `5e35544`.

---

## 2. Final architecture (as built 2026-05-26)

### 2.1 System inventory

| Item | Value |
|------|-------|
| Host | MacBook Pro M4, macOS Darwin 24.6 |
| Container runtime | OrbStack 2.1.3 (drop-in for Docker Desktop), 10 vCPU, 12 GiB RAM, Rosetta on |
| Docker CLI | 29.4.0 (via OrbStack) |
| LLM runtime (host) | Ollama 0.24.0, models in `/Volumes/RESEARCH_SSD/ai_caches/ollama_models` |
| External SSD | `/Volumes/RESEARCH_SSD` (APFS, 4 TB, ~3.2 TB free) |
| OrbStack VM data | `~/Library/Group Containers/HUAQ24HBR6.dev.orbstack/data` → symlink → `/Volumes/RESEARCH_SSD/orbstack/data` |
| Stack manager | Dockge `louislam/dockge:1` at `http://localhost:5001` |
| TUI manager | Lazydocker 0.25.2 via `brew install lazydocker` |
| Repo (Postiz fork) | `/Users/jonc/code/postiz-docker-compose` (symlinked into stacks dir) |
| Git `origin` | `jon-chun/postiz-docker-compose` (your fork — read/write, HTTPS) |
| Git `upstream` | `gitroomhq/postiz-docker-compose` (read-only, for `git fetch upstream`) |

### 2.2 Stack inventory

| # | Stack | Compose file location | Containers | Primary URL |
|---|-------|----------------------|------------|-------------|
| 1 | **Postiz** | `/Volumes/RESEARCH_SSD/stacks/postiz/` (symlink → `~/code/postiz-docker-compose`) | 10 (postiz, litellm, postgres, redis, spotlight, temporal × 5) | http://localhost:4007 |
| 2 | **Dockge** | `/Volumes/RESEARCH_SSD/stacks/_dockge/` | 1 | http://localhost:5001 |
| 3 | **Stirling-PDF** | `/Volumes/RESEARCH_SSD/stacks/stirling-pdf/` | 1 | http://localhost:8080 |
| 4 | **n8n** | `/Volumes/RESEARCH_SSD/stacks/n8n/` | 1 | http://localhost:5678 |
| 5 | **searcharvester** | `/Volumes/RESEARCH_SSD/stacks/searcharvester/` (git clone) | 4 (frontend, tavily-adapter, searxng, redis) | http://localhost:9762 |

### 2.3 Port matrix (host side)

> **Rule:** every host port assigned exactly once. Container-internal ports irrelevant for collision.

| Port | Stack | Service | Purpose | Hardcoded elsewhere? |
|------|-------|---------|---------|----------------------|
| 4000 | Postiz | litellm | OpenAI-compatible AI gateway | searcharvester `.env` |
| 4007 | Postiz | postiz | Frontend (host) → 5000 (container) | OAuth callback URLs |
| 5001 | Dockge | dockge | Manager web UI | — |
| 5678 | n8n | n8n | Workflow editor + webhooks | `WEBHOOK_URL` in n8n .env |
| 7233 | Postiz | temporal | Temporal gRPC | — |
| 8000 | searcharvester | tavily-adapter | Tavily-compatible search API | `VITE_API_URL` in searcharvester .env |
| 8080 | Stirling-PDF | stirling-pdf | PDF tools UI | — |
| 8969 | Postiz | spotlight | Sentry Spotlight (dev) | — |
| 8999 | searcharvester | searxng | SearXNG metasearch | — |
| 9762 | searcharvester | frontend | Searcharvester UI | — |
| 11434 | (host) | ollama | Ollama API (not Docker) | LiteLLM `config.yaml` |
| **18080** | **Postiz** | **temporal-ui** | **Remapped from 8080** to dodge Stirling-PDF | — |

### 2.4 Storage layout

```
/Volumes/RESEARCH_SSD/                                      [4 TB APFS, 3.2 TB free]
├── orbstack/data/                                          OrbStack VM, Docker images & volumes
│   └── docker/volumes/                                     Named volumes for all stacks
├── ai_caches/
│   └── ollama_models/                                      Ollama models (llama3.2:3b, gemma4)
├── postiz/                                                 Postiz operational data (out-of-repo)
│   ├── secrets.env                                         chmod 600 master keys
│   ├── litellm/config.yaml                                 14-model routing config
│   ├── backups/                                            pg_dump targets
│   ├── scripts/backup.sh
│   └── config/                                             reserved
└── stacks/                                                 Dockge STACKS_DIR
    ├── _dockge/   (compose + data/)                        the manager
    ├── n8n/       (compose + .env + data/)                 SQLite + workflows
    ├── postiz/    -> /Users/jonc/code/postiz-docker-compose   symlink to IDE source
    ├── searcharvester/  (cloned repo + .env + config.yaml)
    └── stirling-pdf/    (compose + data/{trainingData,extraConfigs,...})
```

### 2.5 Network strategy

Each compose project gets its own bridge network by default — strong isolation between stacks. Cross-stack traffic uses one of two patterns:

| Pattern | When |
|---------|------|
| `http://host.docker.internal:<port>` | Sporadic / read-only calls. Used today by searcharvester → LiteLLM. |
| External shared network (`docker network create dev-shared`) | If/when stacks need persistent service-name addressing. Not in use yet. |

### 2.6 Restart policy strategy

| Policy | Behavior | Used for |
|--------|----------|----------|
| `always` | Restarts on exit *and* on `docker stop` | Postiz, postiz-postgres, postiz-redis (upstream choices, left intact) |
| `unless-stopped` | Restarts on exit, *respects* `docker stop` | Everything else (the 7 we added + Dockge + Stirling-PDF + n8n + LiteLLM + searcharvester's own services) |

`unless-stopped` is preferred for dev work — you can `docker compose stop <stack>` and it stays stopped. `always` is fine for the Postiz core but means it'll come back after an explicit stop too.

### 2.7 Git remote topology

After the 2026-05-27 push, this repo has two remotes:

| Remote | URL | Use |
|--------|-----|-----|
| `origin` | https://github.com/jon-chun/postiz-docker-compose | Your fork — read/write, default push target |
| `upstream` | https://github.com/gitroomhq/postiz-docker-compose | Postiz upstream — read-only via `git fetch upstream` |

Both are HTTPS (matches the `gh` CLI's auth config). To pull future upstream Postiz changes:

```bash
git fetch upstream
git merge upstream/main      # or `git rebase upstream/main` for linear history
git push origin main
```

Merge conflicts most likely on `docker-compose.yaml` (we've edited interpolation + restart policies + port + project name) and `README.md` (fully rewritten). The decisions in §3 are the rationale you'd cite when resolving them.

---

## 3. Decisions made — pros/cons

### Decision 3.1 — Stack manager: **Dockge** (primary) + Lazydocker (TUI)

| Option | Pros | Cons | Verdict |
|--------|------|------|---------|
| **Dockge** | Stack-first UX exactly matching "selectively activate" goal. Tiny (~100 MB image). Auto-discovers compose dirs. In-browser YAML edit + log stream. MIT. | Less feature-rich than Portainer. No RBAC. | **Chosen** |
| Portainer CE | Industry standard. Templates, registries, multi-host. | ~250 MB image, ~150 MB RAM, more complex UI. Container-first, stacks secondary. | Skipped |
| Lazydocker | TUI, single binary, no service. | No remote access. | **Adopted as complement** |
| Raw `docker compose` + zsh | Zero extra software. Scriptable. | No UI. Manual per-stack coordination. | Always available alongside |

**Why both Dockge and Lazydocker:** different ergonomics for different tasks. Dockge wins for "show me the state and let me click start/stop", Lazydocker wins for "kill this one container, exec into another, watch logs side-by-side, all from keyboard."

### Decision 3.2 — Multi-provider LLM via LiteLLM, not Postiz' built-in OpenAI

| Option | Pros | Cons | Verdict |
|--------|------|------|---------|
| **LiteLLM as Postiz sidecar** | Multi-provider routing, automatic fallback, single point of key management, exposes endpoint to other stacks. | Extra container. `OPENAI_BASE_URL` honoring depends on the OpenAI SDK Postiz uses (works in our test). | **Chosen** |
| Direct OpenAI key only | Simplest possible. | Lock-in, no local fallback, no cost optimization, key duplicated per stack. | Skipped |
| Modify Postiz code | Native Anthropic/etc. integration. | Maintenance burden, divergence from upstream. | Skipped |

**Why this is load-bearing:** searcharvester also accepts `OPENAI_BASE_URL` / `OPENAI_API_KEY`. By pointing both at LiteLLM, AI keys are managed **once** (in Postiz' `.env`) and routed to the right provider — local for free/private, commercial for quality.

### Decision 3.3 — README treatment: **full replacement**

| Option | Pros | Cons | Verdict |
|--------|------|------|---------|
| **Replace fully** | Fork has diverged enough (LiteLLM, .env, custom restart policies, port remap). New README documents what's *here*, not the upstream defaults. | Loses upstream content; future merges must re-resolve README conflicts. | **Chosen** |
| Append "Local customizations" | Preserves upstream. | Two parallel narratives, awkward to read. | Skipped |
| Two-file split | Keeps upstream pristine. | Discoverability: GitHub renders README.md first. | Skipped |

### Decision 3.4a — Pin Postiz' compose project name

When you access a compose project through a symlink, `docker compose` defaults the project name to the **symlink directory's basename**, not the target's — so `cd /Volumes/.../stacks/postiz/ && docker compose ps` showed empty even though the containers were happily running under project `postiz-docker-compose`. Same problem would have hit Dockge.

Fix: add `name: postiz-docker-compose` at the top of `docker-compose.yaml`. Now project name is stable regardless of access path. Changing it would orphan the existing named volumes (data loss), so the value matches the original auto-derived name.

### Decision 3.4 — Postiz repo location: **symlink from stacks/**

| Option | Pros | Cons | Verdict |
|--------|------|------|---------|
| **Symlink** `stacks/postiz` → `~/code/postiz-docker-compose` | IDE source-of-truth at `~/code`, Dockge sees stack under `/Volumes/.../stacks/`. Reversible. | One extra level of indirection. | **Chosen** |
| Move repo into `stacks/` | Single canonical location. | Breaks IDE bookmarks, terminal habits, gitignore globals. | Skipped |
| Point Dockge at `~/code` | No symlink. | Dockge scans everything in `~/code`, including unrelated repos. | Skipped |

### Decision 3.5 — Port remap **Temporal UI 8080 → 18080**

| Option | Pros | Cons | Verdict |
|--------|------|------|---------|
| **Remap now (18080)** | Future-proofs against Stirling-PDF (default 8080). One-line edit, no Postiz UI impact. | Bookmark drift if you had `:8080` saved. | **Chosen** |
| Defer | No edit. | Will be forced when Stirling-PDF or searcharvester starts; same edit but later. | Skipped |

### Decision 3.6 — `restart: unless-stopped` added to Postiz' 6 unpolicied services

Spotlight + the entire Temporal stack (5 containers) had no `restart:` policy in the upstream compose. They were killed by SIGKILL when OrbStack restarted during keychain config; only the 3 `restart: always` services came back. Added `unless-stopped` so future OrbStack/SSD/reboot recoveries are automatic.

**Left `always` intact on the 3 existing services** (postiz, postiz-postgres, postiz-redis) — they were explicit upstream choices and harmonizing them changes behavior for explicit `docker compose stop`.

### Decision 3.7 — All AI keys live in Postiz' `.env`; other stacks reference the LiteLLM proxy

Single source of truth. Adding a new SOTA provider = edit `.env` once, restart LiteLLM, every stack benefits.

### Decision 3.8 — Three-document split: README ↔ user-manual ↔ devops plan

| Document | Audience | Length | Job |
|----------|----------|--------|-----|
| [`README.md`](../README.md) | First-time visitor to the GitHub repo | ~250 lines | Orient + 5-min quickstart + top-3 troubleshooting recipes + index |
| [`docs/user-manual.md`](../docs/user-manual.md) | Operator (future-you, teammate) — install-through-troubleshoot | ~1500 lines | Comprehensive step-by-step. Parts I–X + 5 Appendices. Survives long absences. |
| `dev/plan_devops_orbstack-multi-swap_20260526.md` *(this file)* | Architect — *why* it's shaped this way | ~700 lines | Decision log, pros/cons, integration patterns. Doesn't repeat the manual. |

Rejected alternatives: one giant README (poor onboarding signal, undiscoverable interior); separate docs per concern (n8n manual, postiz manual, etc. — fragments cross-stack patterns).

**The contract between docs:**
- README **links into** user-manual section anchors for any topic deeper than a paragraph.
- User-manual **links back to** README only for the at-a-glance landing experience.
- This plan **does not** repeat user-manual content; it references the manual for "how" and focuses on rationale ("why").

When the stack composition changes (new stack added, model list edited, etc.), all three may need touches — see §8 "Future considerations" for the maintenance item.

---

## 4. Step-by-step implementation log (what we ran)

Reconstructed in execution order — each ✅ marks a `docker compose ps` / `curl` verification that passed at that step.

### Phase A — Preconditions and prep

1. ✅ **Pre-flight verification** — SSD mounted (3.5 TB free), OrbStack `Running`, OrbStack data dir already symlinked to SSD, Ollama 0.24 with two models pulled, host ports 4007/5432/6379/7233/8080/8969/4000 all free.
2. ✅ **SSD operational tree:** `mkdir -p /Volumes/RESEARCH_SSD/postiz/{config,backups,litellm,scripts}`
3. ✅ **JWT secret:** `openssl rand -base64 48 | tr ...` → `/Volumes/RESEARCH_SSD/postiz/secrets.env` (chmod 600)
4. ✅ **LiteLLM master key:** `sk-litellm-local-<random>` likewise.

### Phase B — Postiz with LiteLLM

5. ✅ Wrote project `/Users/jonc/code/postiz-docker-compose/.env` mirroring `secrets.env` for Compose interpolation. Verified gitignored via `~/.gitignore_global`.
6. ✅ Edited `docker-compose.yaml`:
    - `JWT_SECRET: ${JWT_SECRET:?JWT_SECRET must be set in .env}`
    - `OPENAI_API_KEY: ${LITELLM_MASTER_KEY:-}`
    - `OPENAI_BASE_URL: ${OPENAI_BASE_URL:-http://litellm:4000/v1}`
7. ✅ Wrote `/Volumes/RESEARCH_SSD/postiz/litellm/config.yaml` — 14 model aliases (gpt-4o/4o-mini/3.5/o3/o3-mini, claude-haiku/sonnet/opus, gemini-flash/pro, openrouter-claude/llama, fireworks-llama-70b, local-fast/local-quality) + fallback chain.
8. ✅ Wrote `docker-compose.litellm.yaml` overlay binding `litellm` on host `:4000` + healthcheck using `python -c urllib.request` (the image lacks wget).
9. ✅ Worked around `credsStore: osxkeychain` triggering a macOS Keychain prompt during `docker compose pull`: temporarily strip the key with `python3 -c "import json...; d.pop('credsStore',None); ..."`, pull, restore. (Later root-fixed by the user.)
10. ✅ `docker compose ... up -d` → 10 containers Up; Postiz `307→/auth`, LiteLLM `"I'm alive!"`, Temporal UI 200, end-to-end chat completion via gpt-4o-mini → falls back to local llama3.2:3b → returns "ok" in ~9 s.

### Phase C — Multi-stack harness

11. ✅ Added `restart: unless-stopped` to the 6 Postiz services missing a policy (spotlight + 5 temporal services).
12. ✅ Remapped Postiz Temporal UI host port `8080 → 18080`.
13. ✅ Created `/Volumes/RESEARCH_SSD/stacks/{_dockge,stirling-pdf,n8n}/` and symlinked `stacks/postiz → ~/code/postiz-docker-compose`.
14. ✅ `brew install lazydocker` → 0.25.2.

### Phase D — Manager + new stacks

15. ✅ Wrote `stacks/_dockge/docker-compose.yaml` (mounts docker socket + STACKS_DIR at same path inside/outside container). `up -d` → http://localhost:5001 returns 200.
16. ✅ Wrote `stacks/stirling-pdf/docker-compose.yaml` (single container, data dirs for OCR/configs/logs). `up -d` → http://localhost:8080 returns 200.
17. ✅ Wrote `stacks/n8n/docker-compose.yaml` + `.env` (generated `N8N_ENCRYPTION_KEY` via `openssl rand -hex 32`, `GENERIC_TIMEZONE=America/New_York`, runners enabled). `up -d` → http://localhost:5678 returns 200.
18. ✅ Cloned `vakovalskii/searcharvester`, wrote `.env` (sending LiteLLM master key as `OPENAI_API_KEY`, `OPENAI_BASE_URL=http://host.docker.internal:4000/v1`, UID/GID set, frontend `VITE_API_URL`), generated `config.yaml` with rotated `secret_key`. `up -d` → searxng :8999, tavily-adapter :8000, frontend :9762 all return 200.

### Phase E — Operational hardening (2026-05-26)

19. ✅ Pinned `name: postiz-docker-compose` at the top of `docker-compose.yaml` (Decision 3.4a) — fixed empty `docker compose ps` output when invoked through the symlink.
20. ✅ Appended Postiz + multi-stack zsh aliases to `~/.zshrc` in a marker-guarded block (`# >>> postiz/orbstack stacks (added 2026-05-26) >>>`). Verified `pup`, `dups`, `pllm`, etc. resolve in a fresh subshell.
21. ✅ Wrote Claude memory at `~/.claude/projects/-Users-jonc-code-postiz-docker-compose/memory/hardware-and-paths.md` indexed by `MEMORY.md` — captures hardware/paths/tooling so future Claude sessions don't re-discover them.

### Phase F — Documentation and publication (2026-05-26 → -27)

22. ✅ Wrote this plan (`dev/plan_devops_orbstack-multi-swap_20260526.md`) — synthesis of decisions and operations runbook.
23. ✅ Rewrote `README.md` from upstream's brief intro to a comprehensive professional README.
24. ✅ Trimmed `README.md` to ~250 lines (orientation + quickstart + top recipes), with depth pushed to `docs/user-manual.md` (~1500 lines, Parts I–X + 5 Appendices) — Decision 3.8.
25. ✅ Created `.env.example` template (with `openssl` invocations in comments) so future clones can bootstrap without reading the manual.
26. ✅ Lived through a real **SSD-disconnect incident** — diagnosed via `diskutil list` (no entry → physical disconnect), reseated cable, restarted OrbStack via `orb start` (not `open -a OrbStack`), brought all stacks back with the loop in §6.1. Captured the full recipe at §7.2.
27. ✅ `git commit -m "feat: LiteLLM gateway, .env interpolation, multi-stack ops docs"` → commit `5e35544` (8 files, +3334/-298).
28. ✅ `gh repo fork --remote=true --remote-name=origin --clone=false` → created `jon-chun/postiz-docker-compose`. Resolved `gh`'s refusal to overwrite the existing `origin` by renaming `origin → upstream` and adding the fork as a new `origin`. Pushed `main` with `-u origin main`.

### Phase G — Steady state (current)

- All 17 containers `Up`, ~16 GB on external SSD, all HTTP probes green.
- Public fork at https://github.com/jon-chun/postiz-docker-compose tracks upstream via `upstream` remote.
- zsh aliases live; Claude memory live; three docs (README + user-manual + this plan) live and cross-linked.

---

## 5. Cross-stack integration patterns

### Pattern A — Shared LiteLLM gateway (current)

Every stack that needs LLM access points `OPENAI_API_KEY` at the LiteLLM master key and `OPENAI_BASE_URL` at `http://host.docker.internal:4000/v1`. LiteLLM routes to the best available provider based on:
1. The model name the caller requested (`gpt-4o-mini`, `claude-sonnet`, `local-fast`, …)
2. Which provider keys are present in Postiz' `.env`
3. The fallback chain in `litellm/config.yaml`

**Currently using this pattern:** Postiz (in-network via `litellm:4000`), searcharvester (out-of-network via `host.docker.internal:4000`).

### Pattern B — Postiz-as-publisher for n8n workflows (future)

n8n can hit the Postiz REST API to schedule posts programmatically. To enable, you'd:
1. Generate a Postiz API key from the UI.
2. Configure an n8n credential `Postiz API`.
3. Use the HTTP Request node against `http://host.docker.internal:4007/api/...`.

### Pattern C — searcharvester research feeding n8n (future)

n8n triggers searcharvester `/research` endpoint, ingests the markdown report, posts to Postiz. All via `host.docker.internal` URLs.

### Pattern D — Shared `dev-shared` network (when you outgrow host.docker.internal)

```bash
docker network create dev-shared
```

In each stack's compose:
```yaml
networks:
  default: { name: <stack>_default }
  dev-shared:
    external: true

services:
  <svc>:
    networks: [default, dev-shared]
```

Containers then reach each other by service name across stacks. Use this when you need persistent inter-stack DNS (e.g., n8n holds a long-lived connection to a service that occasionally restarts and gets a new host gateway IP).

---

## 6. Operations runbook

### 6.1 Start / stop a stack — CLI

```bash
# Start a specific stack
cd /Volumes/RESEARCH_SSD/stacks/<name> && docker compose up -d

# Stop (preserve volumes)
cd /Volumes/RESEARCH_SSD/stacks/<name> && docker compose down

# Stop and remove volumes (DESTRUCTIVE)
cd /Volumes/RESEARCH_SSD/stacks/<name> && docker compose down -v

# Postiz uses two compose files
cd /Volumes/RESEARCH_SSD/stacks/postiz && \
  docker compose -f docker-compose.yaml -f docker-compose.litellm.yaml up -d
```

### 6.2 Start / stop a stack — Dockge UI

http://localhost:5001 → first-launch admin signup → each stack appears as a card → ▶ Start / ⏹ Stop / ↻ Restart / ⬆ Update buttons. Compose YAML is editable inline; logs stream live per service.

### 6.3 Start / stop a stack — Lazydocker TUI

```bash
lazydocker
```
Navigation: `←/→` containers/projects, `Enter` for logs, `r` restart, `s` stop, `x` menu. `?` for help.

### 6.4 Upgrade a stack

```bash
cd /Volumes/RESEARCH_SSD/stacks/<name>
docker compose pull && docker compose up -d
```

Or in Dockge UI: click ↻ on the stack card.

**Special case — Postiz:** always run `/Volumes/RESEARCH_SSD/postiz/scripts/backup.sh` first; major versions (v2.11 → v2.12) added Temporal and required a migration. See https://docs.postiz.com/installation/migration.

### 6.5 Backup

```bash
# Postiz Postgres
docker compose -f docker-compose.yaml -f docker-compose.litellm.yaml exec -T postiz-postgres \
  pg_dump -U postiz-user -d postiz-db-local | gzip > /Volumes/RESEARCH_SSD/postiz/backups/postiz-$(date +%Y%m%d).sql.gz

# n8n (SQLite + workflows + creds)
tar czf /Volumes/RESEARCH_SSD/postiz/backups/n8n-$(date +%Y%m%d).tar.gz \
  -C /Volumes/RESEARCH_SSD/stacks/n8n data

# Stirling-PDF — typically nothing to back up (PDFs are transient); export Settings via the UI if needed.

# searcharvester — config.yaml only; logs and caches are recoverable.
```

### 6.6 Monitoring

| Signal | How |
|--------|-----|
| Container state | `docker compose ps`, Dockge UI |
| Resource usage | `docker stats`, Lazydocker top pane, OrbStack menubar |
| Logs (one stack) | `docker compose logs -f --tail=200` |
| Logs (one container) | `docker logs -f <container>` |
| Postiz workflow state | Temporal UI → http://localhost:18080 |
| LiteLLM call success | `docker logs -f postiz-litellm` |

### 6.7 zsh aliases (recommended)

Append to `~/.zshrc`:

```bash
# --- Stack ergonomics ---
export STACKS_DIR=/Volumes/RESEARCH_SSD/stacks
alias scd='cd $STACKS_DIR'
# Postiz
export POSTIZ_DIR=$STACKS_DIR/postiz
export COMPOSE_FILE=docker-compose.yaml:docker-compose.litellm.yaml  # so plain `docker compose` works in postiz dir
alias pup='(cd $POSTIZ_DIR && docker compose up -d)'
alias pdown='(cd $POSTIZ_DIR && docker compose down)'
alias plogs='(cd $POSTIZ_DIR && docker compose logs -f --tail=200 postiz)'
alias ppsql='(cd $POSTIZ_DIR && docker compose exec postiz-postgres psql -U postiz-user -d postiz-db-local)'
# Generic
alias dups='for d in $STACKS_DIR/*/; do (cd "$d" && echo "=== $(basename $d) ===" && docker compose ps); done'
alias dlog='lazydocker'
```

`dups` (docker-ups) prints status of every stack in one screen. `dlog` opens Lazydocker.

### 6.8 Git workflow (fork ↔ upstream)

```bash
# Day-to-day: commit and push your changes
git add <files>
git commit -m "<message>"
git push                                # `main` tracks `origin/main` (the fork)

# Periodically: pull Postiz upstream changes into your fork
git fetch upstream
git log HEAD..upstream/main --oneline   # see what's new upstream
git merge upstream/main                 # or `git rebase upstream/main` for linear history
git push origin main

# Resolving merge conflicts (most likely on docker-compose.yaml + README.md):
# - For docker-compose.yaml — keep your interpolation/restart/port/name customizations,
#   cherry-pick any new env vars or service additions from upstream.
# - For README.md — yours is a full rewrite; upstream changes here usually concern
#   upstream's quickstart text and don't apply. Resolve in your favor.

# Compare a local file with upstream's:
git show upstream/main:docker-compose.yaml > /tmp/upstream-compose.yaml
diff docker-compose.yaml /tmp/upstream-compose.yaml
```

You **cannot** accidentally `git push` to upstream: HTTPS without write tokens returns a 403, so a misdirected push fails loudly instead of corrupting upstream.

---

## 7. Troubleshooting recipes

### 7.1 OrbStack stuck "Starting"

Symptom: `orb status` reports `Starting` forever; `docker ps` hangs; socket missing.

```bash
orb stop
sleep 3
orb start    # NOT `open -a OrbStack` — that launches the GUI but doesn't kick the VM reliably
# wait 30 s
orb status   # should say Running
```

### 7.2 Containers gone after sleep/wake or SSD unmount

**Symptom progression (lived through 2026-05-27):**
1. Browser can't reach `http://localhost:5001` (or any stack URL).
2. `docker ps` hangs or returns "failed to connect to the docker API at unix:///Users/jonc/.orbstack/run/docker.sock".
3. `orb status` reports `Stopped` (or sometimes `Starting` indefinitely).
4. `ls /Volumes/RESEARCH_SSD/` returns "Permission denied" — directory entry stale.
5. `diskutil list | grep RESEARCH_SSD` returns nothing — the SSD is physically disconnected (most common cause) or in a broken state.

**Why it cascades:** OrbStack's VM data dir is symlinked to `/Volumes/RESEARCH_SSD/orbstack/data`. If the SSD unmounts (USB-C cable loose, hub power glitch, "put hard disks to sleep" enabled, manual eject), the symlink dangles → OrbStack can't access its VM → VM dies → every container dies with it.

**Recovery procedure:**

```bash
# 1. Confirm the SSD really is gone
diskutil list | grep RESEARCH_SSD
ls /Volumes/RESEARCH_SSD/ >/dev/null 2>&1 && echo "mounted" || echo "NOT mounted"

# 2. Physically reseat the USB-C / Thunderbolt cable.
#    Try a different port. If on a hub, bypass it (connect straight to the Mac).

# 3. Wait 5 s, re-verify
diskutil list | grep RESEARCH_SSD                    # should show now
df -h /Volumes/RESEARCH_SSD                          # should print the volume

# 4. Start OrbStack — `orb start` is more reliable than the GUI here
orb start
for i in $(seq 1 30); do
  s=$(orb status 2>/dev/null)
  echo "[$i] $s"
  [ "$s" = "Running" ] && break
  sleep 2
done

# 5. Bring stacks back. `restart: always` / `unless-stopped` services auto-recover;
#    the rest need this loop:
cd ~/code/postiz-docker-compose && \
  docker compose -f docker-compose.yaml -f docker-compose.litellm.yaml up -d
for d in /Volumes/RESEARCH_SSD/stacks/{_dockge,stirling-pdf,n8n,searcharvester}/; do
  (cd "$d" && docker compose up -d)
done

# 6. Verify (every probe should return non-000)
docker ps --format 'table {{.Names}}\t{{.Status}}' | head -20
for p in 4007 5001 5678 8080 18080 4000 8000 8999 9762; do
  printf '  :%-5s -> %s\n' "$p" \
    "$(curl -fsS -o /dev/null -w '%{http_code}' --max-time 5 http://localhost:$p/ 2>/dev/null || echo "000")"
done
```

**Prevention (do once):**
- macOS → **System Settings → Battery → Options → "Put hard disks to sleep when possible" → OFF.**
- Bypass bus-powered USB hubs for the SSD; connect straight to the Mac.
- OrbStack → Settings → General → "Start at login" → ON.

### 7.3 macOS Keychain popup during `docker compose pull`

`credsStore: osxkeychain` in `~/.docker/config.json` triggers a prompt. If you keep accidentally dismissing it:

```bash
# Permanent fix (you have private registries? skip this and click Always Allow instead):
python3 -c "import json; p='/Users/jonc/.docker/config.json'; d=json.load(open(p)); d.pop('credsStore',None); json.dump(d,open(p,'w'),indent=2)"
```

### 7.4 Port collision when bringing up a stack

```bash
lsof -nP -iTCP:<port> -sTCP:LISTEN
```

Edit the host side of the offending `ports:` entry. Container side stays unchanged.

### 7.5 LiteLLM returns 401 / "no provider"

Either:
- Postiz' `.env` is missing the requested provider's key — add it, then `docker compose restart litellm`.
- The model name doesn't match anything in `config.yaml` — list aliases:
  ```bash
  curl -fsS -H "Authorization: Bearer $(grep ^LITELLM_MASTER_KEY $POSTIZ_DIR/.env | cut -d= -f2)" \
    http://localhost:4000/v1/models | jq -r .data[].id
  ```

### 7.6 Postiz copilot ignores `OPENAI_BASE_URL`

If `docker compose logs -f postiz` shows requests still hitting `api.openai.com`:

```yaml
# Add to postiz service in docker-compose.yaml
extra_hosts:
  - "api.openai.com:host-gateway"
```

This forces the DNS resolution for `api.openai.com` to the host gateway, where LiteLLM listens on 4000. Then in `litellm/config.yaml` you'd also need an explicit listener for that path (deeper customization — only if env-var route fails).

### 7.7 Dockge can't see / start a stack

Check the path mount:
```bash
docker exec dockge ls -la /Volumes/RESEARCH_SSD/stacks/
```
Should list all 5 entries. If it doesn't, the volume mount line is wrong.

### 7.8 n8n encryption key lost

Without `N8N_ENCRYPTION_KEY` you can't decrypt stored credentials. Recovery: there is none — only restore from a backup of `stacks/n8n/data/`. **Back up after every credentials change.**

### 7.9 Stirling-PDF crashes on big PDFs

Increase OrbStack memory (Settings → System → Memory). The Java process can eat 2–4 GB on multi-hundred-page docs. Also clear `stacks/stirling-pdf/data/logs` periodically — log retention has no rotation by default.

### 7.10 searcharvester `/research` returns 500

Hermes child container failed to spawn — usually `OPENAI_API_KEY` or `OPENAI_BASE_URL` unset. Check `docker logs -f tavily-adapter`. Confirm LiteLLM is reachable from inside the container:
```bash
docker exec tavily-adapter sh -c 'apk add --no-cache curl >/dev/null 2>&1; curl -fsS http://host.docker.internal:4000/health/liveliness'
```

### 7.11 LiteLLM container shows "unhealthy" but actually works

`docker inspect postiz-litellm --format='{{ .State.Health.Status }}'` reports `unhealthy` while `curl http://localhost:4000/health/liveliness` returns `"I'm alive!"`.

**Cause:** an earlier `docker-compose.litellm.yaml` healthcheck used `wget`, but the LiteLLM image ships `curl` and Python — not `wget`. The healthcheck always fails; the container is marked unhealthy. Any `depends_on: { condition: service_healthy }` gated on this container would stall.

**Fix (already applied in this fork — verify if you cherry-pick a different overlay):**

```yaml
healthcheck:
  test: ["CMD-SHELL", "python -c \"import urllib.request,sys; urllib.request.urlopen('http://localhost:4000/health/liveliness', timeout=3); sys.exit(0)\" || exit 1"]
  interval: 30s
  timeout: 5s
  retries: 3
  start_period: 20s
```

### 7.12 `docker compose ps` empty when invoked through a symlink

You `cd /Volumes/RESEARCH_SSD/stacks/postiz/` (a symlink to `~/code/postiz-docker-compose/`) and `docker compose ps` returns nothing — but the containers are running.

**Cause:** project name defaults to the **symlink directory's** basename (`postiz`), not the target's (`postiz-docker-compose`). To Compose, those are two different projects.

**Fix (already applied — Decision 3.4a):** the `name:` field at the top of `docker-compose.yaml` pins project name to `postiz-docker-compose` regardless of cwd. If you remove or change it, **don't recreate** under the alternate name — the existing volumes (`postgres-volume`, `postiz-uploads`, etc.) would orphan and you'd lose data.

### 7.13 `gh repo fork --remote-name=origin` refuses to overwrite

**Symptom:** `gh repo fork --remote=true --remote-name=origin` errors with `a git remote named 'origin' already exists`. The fork on GitHub *is* created — but no local remote is added, leaving `origin` still pointing at upstream.

**Recovery (we did this 2026-05-27):**

```bash
git remote rename origin upstream
git remote add origin https://github.com/<you>/postiz-docker-compose.git
git push -u origin main
git remote -v   # confirm: origin → fork, upstream → gitroomhq
```

---

## 8. Future considerations

| Item | When to do it | Effort |
|------|---------------|--------|
| **Reverse proxy** (Caddy or Traefik) with `*.local.test` hostnames | When port numbers become tedious to remember | ~30 min — one Caddyfile, no per-stack changes if you use hostname routing |
| **`dev-shared` external network** | When two stacks need to talk regularly | ~5 min global + 1 line per stack |
| **Watchtower** auto-updates | When you trust upstream image tags | ~5 min, one container — but pair with backups, breaking changes happen |
| **launchd nightly backups** | Now if data matters | ~15 min, one `~/Library/LaunchAgents/*.plist` |
| **Postgres migration: n8n SQLite → Postgres** | If n8n workflows become heavy | ~20 min |
| **Pin image tags** instead of `:latest` / `:1` | When you hit a "worked yesterday" bug | 30 s per stack |
| **Sync README + user-manual + this plan** | After any stack composition change (new stack, model edit, port change) | ~10 min — README/user-manual section anchors usually need updating; this plan needs an entry in §4 |
| **Pull Postiz upstream changes** | When upstream tags a new release | ~5 min via `git fetch upstream && git merge` (conflicts likely in `docker-compose.yaml` + `README.md` — see §6.8) |

---

## 9. Appendices

### Appendix A — Env var quick reference (Postiz fork)

| Variable | Location | Purpose |
|----------|----------|---------|
| `JWT_SECRET` | `postiz/.env` | Postiz session signing — rotating logs everyone out |
| `LITELLM_MASTER_KEY` | `postiz/.env` | Bearer token Postiz sends → LiteLLM accepts |
| `OPENAI_API_KEY` | `postiz/.env` | Real OpenAI key (LiteLLM container only) |
| `ANTHROPIC_API_KEY` | `postiz/.env` | Real Anthropic key |
| `GEMINI_API_KEY` | `postiz/.env` | Real Google AI Studio key |
| `OPENROUTER_API_KEY` | `postiz/.env` | Real OpenRouter key |
| `FIREWORKS_API_KEY` | `postiz/.env` | Real Fireworks key |
| `N8N_ENCRYPTION_KEY` | `n8n/.env` | n8n credential encryption — lose it, lose your creds |
| (searcharvester) `OPENAI_BASE_URL` | `searcharvester/.env` | `http://host.docker.internal:4000/v1` — points at LiteLLM |

### Appendix B — File inventory

```
Created or modified by this work — all in commit 5e35544 unless noted:

/Users/jonc/code/postiz-docker-compose/                git origin:   jon-chun/postiz-docker-compose
                                                       git upstream: gitroomhq/postiz-docker-compose
├── .env                                          [LOCAL ONLY, gitignored via ~/.gitignore_global]
├── .env.example                                  [created — template for new clones]
├── docker-compose.yaml                           [modified — name pin + interpolation + 7 restart policies + 8080→18080 port remap]
├── docker-compose.litellm.yaml                   [created — LiteLLM overlay, python healthcheck]
├── README.md                                     [rewritten 2x: full replacement → trimmed to ~250 lines (Decision 3.8)]
├── docs/
│   └── user-manual.md                            [created ~1500 lines — Parts I–X + 5 appendices (Decision 3.8)]
└── dev/
    ├── plan_setup-config.md                      [created — Postiz-only setup plan, predates multi-stack work]
    ├── plan_devops_orbstack-multi-swap_20260526.md  [this file]
    └── MacBookProM4-setup.md                     [pre-existing Grok notes]

/Volumes/RESEARCH_SSD/                            [NOT committed — operational data lives outside the repo]
├── postiz/
│   ├── secrets.env                               [chmod 600 master copy of .env]
│   ├── litellm/config.yaml                       [14-model router]
│   ├── backups/                                  [pg_dump targets]
│   ├── scripts/                                  [backup.sh helper]
│   └── config/                                   [reserved]
└── stacks/                                       [Dockge STACKS_DIR]
    ├── _dockge/
    │   ├── docker-compose.yaml                   [created]
    │   └── data/                                 [Dockge state — admin account, stack metadata]
    ├── n8n/
    │   ├── docker-compose.yaml                   [created]
    │   ├── .env                                  [chmod 600, encryption key — back this up!]
    │   └── data/                                 [SQLite + workflows + creds]
    ├── stirling-pdf/
    │   ├── docker-compose.yaml                   [created]
    │   └── data/{trainingData,extraConfigs,...}
    ├── searcharvester/                           [cloned from vakovalskii/searcharvester @ 2026-05-26]
    │   ├── .env                                  [chmod 600, wired through LiteLLM]
    │   └── config.yaml                           [secret_key rotated]
    └── postiz                                    [symlink → /Users/jonc/code/postiz-docker-compose]

Host-level installs (not in any repo):
/opt/homebrew/bin/lazydocker                      [v0.25.2, via brew]

~/.zshrc                                          [appended alias block, marker `>>> postiz/orbstack stacks (added 2026-05-26)`]

~/.docker/config.json                             [`credsStore: osxkeychain` — user accepted Keychain prompt permanently]

~/.claude/projects/-Users-jonc-code-postiz-docker-compose/memory/
├── MEMORY.md                                     [index, single entry]
└── hardware-and-paths.md                         [Claude memory: hardware/SSD/Ollama/ports/tooling]
```

### Appendix C — Glossary

| Term | Meaning |
|------|---------|
| **Stack** | One `docker-compose.yaml` (+ overlays) = one compose project = one bridge network = one row in Dockge. |
| **STACKS_DIR** | Directory Dockge scans for stack subdirectories. Here: `/Volumes/RESEARCH_SSD/stacks`. |
| **LiteLLM master key** | The `Bearer` token LiteLLM accepts from clients. Distinct from upstream provider API keys. |
| **host.docker.internal** | DNS alias OrbStack provides inside containers, resolving to the Mac host. |
| **Temporal** | Workflow engine Postiz uses for cron + reliable post scheduling. Not optional since v2.12. |
| **Spotlight** | Sentry's local debugger UI bundled in the Postiz compose for dev. |

### Appendix D — One-page operator cheat sheet

```bash
# ── Status everywhere ──
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
dups                              # alias — `compose ps` for every stack

# ── Open the manager ──
odockge                           # alias — open http://localhost:5001 in browser
lazydocker                        # or `dlog` alias — TUI

# ── Per-stack ──
pup / pdown / pps / plogs         # Postiz (auto-includes LiteLLM overlay)
prestart / ppull / pexec / ppsql  # Postiz: restart / pull-and-up / exec / psql
pllm                              # list LiteLLM model aliases
cd $STACKS_DIR/<name> && docker compose up -d
cd $STACKS_DIR/<name> && docker compose down

# ── AI gateway ──
curl -fsS http://localhost:4000/health/liveliness
LMK=$(grep ^LITELLM_MASTER_KEY $POSTIZ_DIR/.env | cut -d= -f2)
curl -fsS -H "Authorization: Bearer $LMK" http://localhost:4000/v1/models | jq

# ── Recovery after SSD/sleep ──
diskutil list | grep RESEARCH_SSD     # SSD mounted?
orb start                              # Start the OrbStack VM
for d in $STACKS_DIR/*/; do (cd "$d" && docker compose up -d); done

# ── Backups ──
$POSTIZ_DIR/scripts/backup.sh         # Postgres
tar czf ... -C $STACKS_DIR/n8n data

# ── Git (fork ↔ upstream) ──
git fetch upstream && git log HEAD..upstream/main --oneline
git merge upstream/main && git push origin main
```

**URLs:**
- http://localhost:4007 — Postiz
- http://localhost:5001 — Dockge
- http://localhost:5678 — n8n
- http://localhost:8080 — Stirling-PDF
- http://localhost:8000/docs — searcharvester API
- http://localhost:8999 — SearXNG
- http://localhost:9762 — searcharvester frontend
- http://localhost:18080 — Temporal UI (Postiz workflows)
- http://localhost:8969 — Spotlight (Postiz debug)
- http://localhost:4000 — LiteLLM gateway (set Authorization: Bearer ...)

**GitHub:**
- Your fork: https://github.com/jon-chun/postiz-docker-compose
- Upstream: https://github.com/gitroomhq/postiz-docker-compose

---

*End of plan. Live state of this machine is the operational source-of-truth; this document is a synthesis. See also: [`README.md`](../README.md) for the quickstart, [`docs/user-manual.md`](../docs/user-manual.md) for the comprehensive operator manual.*
