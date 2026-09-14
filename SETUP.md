# jibjabjog/jibjabjob — Host Setup Documentation

Purpose: backup and reference for replicating the same environment.
No secrets, API keys, or credentials included.

## Hardware & OCI Environment
- **Platform**: Oracle Cloud Free Tier, ARM64
- **CPU**: Ampere A1 (4 cores)
- **RAM**: 24 GB
- **GPU**: None (CPU-only inference)
- **OS**: Ubuntu 24.04 LTS (Noble Numbat)

## Hermes Agent
- **Version**: 0.20.5 (Aug 2026; ~71 commits behind main)
- **Install**: `~/.hermes/hermes-agent/venv/bin/python`
- **Gateway**: systemd user service (`hermes-gateway.service`), linger enabled
- **Default model**: `openrouter/free` (provider: openrouter)
- **Config highlights** (`~/.hermes/config.yaml`):
  - `max_turns: 30`
  - `gateway_timeout: 3000`
  - `image_input_mode: disabled`
  - `terminal.backend: local`

## Local LLM Services
- **llama-server**: `127.0.0.1:8080` (primary)
- **qwen35-tiny (Inky)**: port 45072 (failover backup, 4 threads, ctx=10240)
- Models: `~/models/` (Qwen3.5 0.8B, 4B, 9B GGUFs)

## Integrations
- **GitHub**: `gh` CLI as `jibjabjog` (gist, read:org, repo, workflow)
- **Google Workspace**: OAuth (tokens stored in `~/.hermes/`)
- **Telegram**: Home channel for cron delivery
- **OpenRouter**: Primary provider (API key in `~/.hermes/.env`)

## Cron Jobs
| Job | Schedule | Script | Delivery |
|---|---|---|---|
| Freerouter | daily 06:00 | `freerouter_failover.sh` | local |
| llama-ping | every 15m | `local_llama_ping.sh` | Telegram |
| hermes-backup | daily 03:00 | `backup_hermes.sh` | local |

## Skills & Plugins
- Custom skills: `~/.hermes/skills/` (~25 active)
- Evey plugins: 30+ (autonomy, bridge, cache, delegate, memory, research, etc.)
- Symlinks from `../../.agents/skills/` for dev workflows

## Scripts
- `freerouter.py` / `freerouter_failover.sh` — OpenRouter → local failover
- `backup_hermes.sh` — config sync to GitHub
- `local_llama_ping.sh` — health check with Telegram alert

## Gaps vs Latest Hermes (v0.20.6)
- Bot Mode, wake-word voice, pets, Mixture of Agents preset
- Desktop app, subscription proxy, voice mode TTS pipeline
- Recurring Loops, memory provider plugins, checkpoints v2
- Image generation (disabled)

## Notes
- Version drift: 0.20.5 installed, 0.20.6 available
- Secrets stored in `~/.hermes/.env`, `google_token.json`, `google_client_secret.json` — excluded from this repo
- Supabase self-hosted on Tailscale IP `100.124.0.62` only
