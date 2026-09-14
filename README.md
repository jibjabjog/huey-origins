# jibjabjog / jibjabjob

Complete host setup documentation for an OCI ARM64 environment running Hermes Agent.

> **No secrets, API keys, or credentials included.**
> Secrets are stored in `~/.hermes/.env` and related files — never committed anywhere.

## Hardware & OCI Environment

| Property | Value |
|---|---|
| **Platform** | Oracle Cloud Free Tier |
| **Architecture** | ARM64 (aarch64) |
| **CPU** | Ampere A1, 4 cores |
| **RAM** | 24 GB |
| **GPU** | None (CPU-only inference) |
| **OS** | Ubuntu 24.04 LTS (Noble Numbat) |
| **User** | `huey` |
| **Home dir** | `/home/huey` |

## Hermes Agent

| Property | Value |
|---|---|
| **Version** | 0.20.5 (Aug 2026; ~71 commits behind main 0.20.6) |
| **Install path** | `~/.hermes/hermes-agent/venv/bin/python` |
| **Venv Python** | `~/.hermes/hermes-agent/venv/bin/python` |
| **Bare `python`** | System Python (not Hermes venv) |
| **Gateway** | systemd user service `hermes-gateway.service`, linger enabled |
| **Default model** | `openrouter/free` (provider: openrouter) |
| **Config file** | `~/.hermes/config.yaml` |

### Key `config.yaml` settings

```yaml
model.default: openrouter/free
model.provider: openrouter
agent.max_turns: 30
agent.gateway_timeout: 3000
agent.image_input_mode: disabled
terminal.backend: local
gateway_timeout_warning: 900
disabled_toolsets: []
personalities: 11 built-in
```

## Local LLM Services

| Service | Address | Purpose |
|---|---|---|
| **llama-server** | `127.0.0.1:8080` | Primary model server |
| **qwen35-tiny (Inky)** | `127.0.0.1:45072` | Failover backup, 4 threads, ctx=10240, CPU-only OpenBLAS |

Models live in `~/models/`:
- `Qwen3.5-0.8B-Q4_K_M.gguf` (primary local)
- `Qwen3.5-4B-Q4_K_M.gguf`
- `Qwen_Qwen3.5-9B-Q6_K_L.gguf`
- `Spark-X2.5-1.7B-Q4_K_M.gguf`

## Integrations

| Integration | Details |
|---|---|
| **GitHub** | `gh` CLI as `jibjabjog` (scopes: gist, read:org, repo, workflow) |
| **Google Workspace** | Full OAuth via `google-workspace` skill; tokens in `~/.hermes/` |
| **Telegram** | Home channel `8716017156`; used for cron delivery |
| **OpenRouter** | Primary provider; `OPENROUTER_API_KEY` in `~/.hermes/.env` |
| **Supabase** | Self-hosted; Tailscale IP `100.124.0.62` only |

## Cron Jobs

| Job | Schedule | Script | Delivery |
|---|---|---|---|
| **Freerouter** | `0 6 * * *` (daily 06:00) | `freerouter_failover.sh` | local |
| **local-llama-ping** | `*/15 * * * *` (every 15 min) | `local_llama_ping.sh` | Telegram |
| **hermes-backup** | `0 3 * * *` (daily 03:00) | `backup_hermes.sh` | local |

Scripts in `~/.hermes/scripts/`:
- `freerouter.py`, `freerouter.sh`, `freerouter_failover.sh` — OpenRouter → local qwen35-tiny failover
- `backup_hermes.sh` — sync config/scripts to GitHub `jibjabjog/hermes-backup`
- `local_llama_ping.sh` — health check with Telegram alert
- `model_manager.py`, `set_fallback_model.py`, `redact_config_secrets.py`

## Skills & Plugins

**Custom skills** (`~/.hermes/skills/`): ~25 active including hermes-local-llm-failover, hermes-set-moe-model, hermes-voice-optimization.

**Evey plugins** (30+): autonomy, bridge, cache, cost-guard, council, delegate, email-guard, goals, habits, identity, memory, moltbook, mqtt, news, proactive, rag, reflect, research, sandbox, scheduler, session-guard, status, telegram-ux, telemetry, validate, wallet.

**Symlinks** from `../../.agents/skills/` for dev workflows: brainstorming, dispatching-parallel-agents, executing-plans, finishing-a-development-branch, requesting-code-review, systematic-debugging, test-driven-development, writing-plans, writing-skills.

## Gaps vs Hermes v0.20.6 Docs

Features documented in latest docs but **not present** in this install:
- Bot Mode (group threads, bot roster)
- Wake word / hands-free voice
- Pets / Petdex mascots
- Mixture of Agents preset active (skill exists, preset not configured)
- Desktop app (not installed on headless server)
- Subscription Proxy / Nous Portal integration
- Voice mode / TTS pipeline (Whisper STT installed; TTS not configured)
- Recurring Loops (`/loop`) — only cron used
- Memory provider plugins (Honcho, Mem0, etc.) — only built-in SQLite
- Checkpoints v2 / rollback pipeline
- Image generation (`image_input_mode: disabled`)
- Kanban multi-agent orchestration (db exists, no multi-agent setup)

**Version drift**: 0.20.5 installed, 0.20.6 released Aug 27 2026 (~71 commits behind).

## Notes

- Secrets are stored in `~/.hermes/.env`, `google_token.json`, `google_client_secret.json` — never committed to any repo
- Supabase self-hosted on Tailscale IP `100.124.0.62` only
- `~/.hermes/hermes-gateway.service` runs via `systemctl --user` with linger enabled
- Backup repo: `jibjabjog/hermes-backup` for `.hermes/` config, failover scripts, presets
- This document mirrors `~/hermes-knowledge/AUDIT.md` locally

---

Created: 2026-09-14 | Purpose: backup setup reference / onboarding guide
