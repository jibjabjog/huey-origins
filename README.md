# Setting Up This Box From Scratch — A Beginner's Guide

**Purpose:** a from-nothing-to-here walkthrough for rebuilding this exact
environment: a headless Hermes Agent deployment on an Oracle Cloud ARM64 VM,
with two local CPU LLMs for fallback and light auxiliary tasks,
GitHub/Google/Telegram integrations, and a
self-hosted Supabase over Tailscale.

**Audience:** someone who has never touched this box before. Every step
assumes a fresh, empty home directory for whatever user account runs Hermes.

> **A note on the username.** This guide uses `huey` throughout — that's
> just what this particular deployment happens to be called, not a
> requirement. Pick any username you like; the only thing that matters is
> using it *consistently* everywhere `huey` appears below (the `adduser`
> command in §1, every `/home/huey/...` path, `huey@<instance-ip>` when you
> SSH in, the `huey:huey` ownership in `chown` commands, etc.). If you're
> not doing a find-and-replace as you go, the easiest path is to actually
> name the account `huey` and skip the substitution entirely — there's
> nothing special about the name beyond that.

**Sources:** this guide was compiled by reading the actual running system
(systemd units, installed packages, git remotes, config files) on 2026-09-14,
cross-checked against an earlier self-documentation pass Hermes wrote for
this same repo that same day. Where the two disagreed, this guide follows
what's actually on disk and calls out the discrepancy — notably, Hermes's
own write-up described its cron jobs in a way that read like standard OS
`cron` entries, but they're actually managed by Hermes's internal scheduler
(see §7). That earlier pass has been superseded by this guide.

> **No secrets in this guide.** Anywhere a real API key, token, or OAuth
> secret is needed, this guide tells you *where* it goes and *how* to obtain
> it — never what the value is. Actual secrets on this box live in
> `~/.hermes/.env`, `~/.hermes/google_token.json`, and
> `~/.hermes/google_client_secret.json`, none of which were read to write this.

---

## Accounts you'll need before you start

Create these up front — nothing below assumes you already have them, but
having them ready saves backtracking mid-setup.

| Account | Cost | Used for | Covered in |
|---|---|---|---|
| Oracle Cloud Infrastructure | Free (Always Free tier) | The VM itself | §1 |
| GitHub — a **second** account for the bot, separate from your personal one | Free | `gh` CLI, backups, this guide's own repo | §3 |
| OpenRouter | Free (no card needed for `openrouter/free` models) | Primary + fallback LLM access | §4 |
| Google (with Cloud Console access) | Free | Gmail/Calendar/Drive integration | §8 |
| Telegram | Free | Bot notifications/alerts | §8 |
| Tailscale | Free (Personal plan, up to 100 devices) | Reaching Supabase and this box privately | §11 |

You'll also want an **SSH keypair** — if you don't already have one:
```bash
ssh-keygen -t ed25519 -C "your-email@example.com"
```
Accept the default file location; a passphrase is optional but recommended.
This gives you `~/.ssh/id_ed25519.pub` — that's the public key §1 asks you
to paste into OCI.

## 0. What you end up with

- Oracle Cloud "Always Free" ARM64 VM, Ubuntu 24.04, 4 OCPU / 24 GB RAM, no GPU
- Hermes Agent (Nous Research) running as a `systemd --user` background service
- A local llama.cpp build running two CPU models: gemma-4-E2B as the real
  `fallback_model` when OpenRouter errors, and a tiny Inky model for cheap
  auxiliary tasks + a health beacon (§6)
- GitHub, Google Workspace, and Telegram wired into Hermes, including
  local, free voice transcription and speech replies over Telegram (§8)
- A self-hosted Supabase stack, reachable only over Tailscale
- Hermes's own "evey-*" plugin suite (30+ plugins) and ~25 custom skills
- Two independent scheduling systems: the OS `crontab` (for one thing:
  backups unrelated to Hermes) and Hermes's *own* internal cron ticker (for
  everything Hermes-related — see §7, this trips people up)

---

## 1. Provision the host

1. Create an Oracle Cloud Infrastructure account, Free Tier is sufficient.
2. Launch a compute instance:
   - Shape: **Ampere A1** (ARM64/aarch64), 4 OCPU, 24 GB RAM — this is the
     free-tier ARM allocation.
   - Image: **Ubuntu 24.04 LTS (Noble Numbat)**.
   - Add your SSH public key at creation time — paste the contents of
     `~/.ssh/id_ed25519.pub` (or whichever keypair you generated above)
     into the "SSH keys" field.
3. Open the security list / NSG for at least SSH (22). Everything else
   (Hermes's gateway, Supabase, llama-server) will be bound to `127.0.0.1` or
   reached over Tailscale, not exposed publicly — don't open extra ports.
4. SSH in as `ubuntu` (or your chosen default user), and create the `huey`
   user if it isn't the default — swap `huey` for whatever username you
   want here (see the note above); just use that same name in every command
   from here on instead of `huey`:
   ```bash
   sudo adduser huey
   sudo usermod -aG sudo huey
   ```
5. **Give yourself SSH access as `huey`.** `adduser` alone won't let you log
   in as `huey` — the easiest path is reusing the SSH key already authorized
   for your current user:
   ```bash
   sudo mkdir -p /home/huey/.ssh
   sudo cp ~/.ssh/authorized_keys /home/huey/.ssh/authorized_keys
   sudo chown -R huey:huey /home/huey/.ssh
   sudo chmod 700 /home/huey/.ssh
   sudo chmod 600 /home/huey/.ssh/authorized_keys
   ```
   Then disconnect and reconnect as `huey` (`ssh huey@<instance-ip>`).
   **Every command in the rest of this guide assumes you're logged in as
   `huey`, not `ubuntu`** — all the paths (`~/.hermes`, `~/llama.cpp`,
   `~/models`, the `systemd --user` service) are `huey`'s.

   ✅ **Test it:** `whoami` should print `huey`; `groups` should list `sudo`.
   If either doesn't match, you're still on `ubuntu` or the key copy didn't
   take — fix that before going any further.
6. **Enable linger** for `huey` so user-level systemd services keep running
   after you log out and across reboots — this is what lets
   `hermes-gateway.service` run as a headless background service:
   ```bash
   sudo loginctl enable-linger huey
   ```
   ✅ **Test it:**
   ```bash
   loginctl show-user huey -p Linger   # should print Linger=yes
   ```

## 2. Base OS packages

Install the toolchain Hermes and llama.cpp both need:

```bash
sudo apt update
sudo apt install -y build-essential cmake git curl \
  libopenblas-dev python3-pip
```

- `build-essential` + `cmake` + `libopenblas-dev` — compiling llama.cpp with
  OpenBLAS acceleration (no GPU on this box, so BLAS matters for CPU
  throughput).
- `python3-pip` — needed later, for downloading models in §6.

**Don't `apt install python3.11` — it's not there.** Hermes pins to Python
3.11 (`.python-version` in the repo), but Ubuntu 24.04's default repos only
ship Python 3.12 as `python3`, and there's no `python3.11`/`python3.11-venv`
package or deadsnakes PPA on a stock install. `apt install` aborts the
*entire* command on any one unknown package name, so including them here
would fail the whole line, not just those two packages. You don't need
them anyway: `setup-hermes.sh` (§4) provisions its own isolated Python 3.11
via `uv python install 3.11` — a standalone build under
`~/.local/share/uv/python/`, nothing to do with the system package manager.
That's also why `uv` gets installed next, before Python 3.11 itself.

✅ **Test it:**
```bash
gcc --version && cmake --version
```
Both should print a version, not `command not found`.

Install Node.js 22 (Hermes's `.nvmrc` pins major version 22; the web UI and
some tooling are Node-based):

```bash
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs
```

✅ **Test it:**
```bash
node --version   # should print v22.x.x
```

Install `uv` (fast Python package manager Hermes's setup script prefers over
raw pip/venv):

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

This installs to `~/.local/bin/uv`, which only lands on `PATH` in *new*
shells. Either start a fresh SSH session, or run `source ~/.bashrc` (or
`source ~/.local/bin/env`, which the installer also writes) before
continuing — otherwise `uv --version` in the next step just says
`command not found`.

✅ **Test it:**
```bash
uv --version   # confirms both the install AND that PATH picked it up
```

## 3. GitHub CLI + bot account

Hermes uses `gh` to act on GitHub as a dedicated bot account rather than your
personal one — keeps its commits/PRs attributable and scoped. If you don't
already have a separate account for this, create one now at
[github.com/signup](https://github.com/signup) with its own email address —
using your personal account works too, but you lose the attribution/scoping
benefit.

```bash
sudo apt install -y gh
gh auth login
```

- Log in as the **bot account** (this box uses `jibjabjog`), not your
  personal GitHub identity.
- When prompted for scopes, this deployment uses: `gist`, `read:org`, `repo`,
  `workflow`. Grant what your use case needs; those four cover repo access,
  gists, workflow dispatch, and read-only org visibility.
- This writes credentials to `~/.config/gh/` — treat that directory as
  sensitive; don't commit or copy it anywhere.

✅ **Test it:**
```bash
gh auth status   # confirms account, scopes, and that the token is live
gh repo list     # confirms it can actually talk to the API, not just that a token is stored
```

## 4. Clone and install Hermes Agent

```bash
mkdir -p ~/.hermes
git clone https://github.com/NousResearch/hermes-agent.git ~/.hermes/hermes-agent
cd ~/.hermes/hermes-agent
./setup-hermes.sh
```

What `setup-hermes.sh` does (worth knowing rather than just trusting):
1. Detects desktop/server vs. Termux(Android) — picks the server path here.
2. Creates a Python 3.11 venv at `~/.hermes/hermes-agent/venv` using `uv`.
3. Installs dependencies into that venv (**this venv's `python` is the only
   one with Hermes's libraries** — the bare system `python`/`python3` will
   NOT have them; scripts and services must reference the venv path
   explicitly: `~/.hermes/hermes-agent/venv/bin/python`. This exact gotcha
   bit the Google Workspace skill on this box — see §8).
4. Copies `.env.example` → `.env` for you to fill in with real secrets. The
   one you need immediately is `OPENROUTER_API_KEY`:
   - Sign up at [openrouter.ai](https://openrouter.ai), then create a key
     under **Settings → Keys**.
   - No payment method is required just to use `openrouter/free`-tagged
     models (the default this box uses, see below) — you only need billing
     set up if you later switch to a paid model.
   - Open `~/.hermes/.env` and set `OPENROUTER_API_KEY=<your key>` by hand.
     Never commit this file or paste the key anywhere else.
5. Symlinks a `hermes` command into `~/.local/bin` (make sure that's on your
   `PATH`).
6. Optionally runs an interactive setup wizard — walks you through picking a
   default model/provider and writing initial `~/.hermes/config.yaml`.

At the end of this step you should have `~/.hermes/config.yaml` and
`~/.hermes/.env` (the latter with your real API keys filled in — this file
is off-limits to read/copy from here on).

✅ **Test it:**
```bash
~/.hermes/hermes-agent/venv/bin/python --version   # should print Python 3.11.x
hermes doctor                                       # full self-check: config,
                                                      # venv, keys, tool backends
```
`hermes doctor` is the single most useful command in this guide — run it
again any time something feels off later on, not just here.

### Key config.yaml choices made on this box

```yaml
model:
  default: openrouter/free
  provider: openrouter
agent:
  max_turns: 30
  gateway_timeout: 3000
  image_input_mode: disabled
  reasoning_effort: low
terminal:
  backend: local
```

- **Provider: OpenRouter**, model **`openrouter/free`** — routes to
  OpenRouter's free-tier model pool rather than paying per-token. Reasonable
  default for a hobby/always-on deployment; means output quality varies with
  whatever's free that day (see §9, "Freerouter").
- `image_input_mode: disabled` — no vision input configured; deliberate,
  since this is a headless text/voice agent.
- `terminal.backend: local` — Hermes runs shell commands directly on this VM
  rather than in a Docker/Modal/Daytona sandbox. Fine for a single-tenant
  personal box; would need reconsidering if untrusted input reaches the
  agent's tool-use loop.

## 5. Run Hermes as a systemd user service

Rather than a login-session process (which dies when you disconnect SSH),
Hermes runs as a `systemd --user` unit, kept alive after logout by the
linger flag set in §1.

Create `~/.config/systemd/user/hermes-gateway.service`:

```ini
[Unit]
Description=Hermes Agent Gateway - Messaging Platform Integration
After=network-online.target
Wants=network-online.target
StartLimitIntervalSec=0

[Service]
Type=simple
ExecStart=/home/huey/.hermes/hermes-agent/venv/bin/python -m hermes_cli.main gateway run
WorkingDirectory=/home/huey/.hermes
Environment="PATH=/home/huey/.hermes/hermes-agent/venv/bin:/home/huey/.hermes/hermes-agent/node_modules/.bin:/usr/bin:/home/huey/.local/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
Environment="VIRTUAL_ENV=/home/huey/.hermes/hermes-agent/venv"
Environment="HERMES_HOME=/home/huey/.hermes"
Restart=always
RestartSec=5
RestartForceExitStatus=75
RestartPreventExitStatus=78
KillMode=mixed
KillSignal=SIGTERM
ExecReload=/bin/kill -USR1 $MAINPID
ExecStopPost=-/home/huey/.hermes/hermes-agent/venv/bin/python -m gateway.cgroup_cleanup
TimeoutStopSec=90
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=default.target
```

Notes on choices worth keeping if you replicate this:
- `ExecStart` calls the **venv** interpreter directly and by absolute path —
  don't rely on `PATH` alone finding the right `python`.
- `Restart=always` / `RestartSec=5` — the gateway should self-heal from
  crashes; `StartLimitIntervalSec=0` on the `[Unit]` means systemd won't give
  up restarting after too many failures in a window.
- `RestartForceExitStatus=75` / `RestartPreventExitStatus=78` — Hermes uses
  specific exit codes as signals to systemd (e.g. "restart me even under
  normal rules" vs. "don't restart, this was a deliberate stop").

Enable and start it:

```bash
systemctl --user daemon-reload
systemctl --user enable --now hermes-gateway.service
systemctl --user status hermes-gateway.service
```

✅ **Test it:**
```bash
hermes status                               # gateway, config, integrations at a glance
journalctl --user -u hermes-gateway -n 20   # tail the actual startup log
```
`systemctl ... status` only tells you the *process* is running; `hermes
status` tells you whether the *agent* is actually healthy (right model
configured, gateway reachable). Worth checking both — a process can be
"active (running)" and still be misconfigured underneath. The linger check
from §1 is what makes this survive logout/reboot; no need to repeat it here.

## 6. Local LLM failover (llama.cpp)

The whole point: if OpenRouter's free tier is unreachable/rate-limited,
Hermes falls back to a model running locally on CPU. **As of 2026-09-18,
this box runs two local models with two different jobs** — don't skip
either half of this section:

- **`qwen35-tiny`** ("Inky", 0.8B) — handles all 11 `auxiliary.*`
  sub-configs in `config.yaml` (compression, skills_hub, approval, mcp,
  title_generation, triage_specifier, kanban_decomposer, profile_describer,
  curator, web_extract, session_search) and the `local-llama-ping` cron
  health beacon (§7). Small, cheap, always resident — right for frequent,
  low-stakes calls.
- **`gemma-4-E2B`** — Hermes's actual `fallback_model`: what a real
  conversation fails over to when the primary OpenRouter model errors.
  Chosen after a dedicated evaluation project (**"Inky's Gym"**, private
  repo `jibjabjog/ai-gym`) benchmarked several small local models —
  character coherence, a simulated-incident agent loop, a job interview —
  and found Inky capable of the auxiliary work above but not of carrying a
  real fallback conversation or reliably finishing an agentic task; gemma
  was the only candidate that did, reliably, at a low temperature.

**History worth knowing before you copy this blindly:** this box's fallback
setup has broken twice already, in two different ways — both are why the
setup below looks the way it does:
1. **2026-09-15:** an earlier two-tier design (4B "everyday" fallback + Inky
   as emergency backup) drifted silently for weeks — `fallback_model`
   pointed at a dead port, 10 of 11 `auxiliary.*` blocks had a placeholder
   key pointed at the real OpenRouter cloud instead of localhost. Fixed by
   retiring that design and pointing everything at Inky alone.
2. **2026-09-18→22:** gemma was deployed as the new fallback (retiring the
   09-15 simplification), but the router defaulted to **4 parallel slots**
   with nothing pinning requests to a consistent one. A dedicated warm-up
   script (below) that's supposed to keep gemma's ~19,000-token system
   prompt cached — so a real failover doesn't pay a ~17-20 minute cold-start
   penalty — kept landing on a *different* slot each cycle, so the cache
   never persisted: it silently redid the full prefill from scratch every
   ~20 minutes, non-stop, for days (~300% CPU sustained). Fixed 2026-09-22
   by pinning the model to a single slot (`parallel = 1` below) — with only
   one slot, there's no ambiguity left, and cache reuse now measurably works
   (`19057/19058` tokens served from cache on a repeat request, confirmed
   live).

**The lesson underneath both:** this setup has enough moving parts
(`config.yaml`, a systemd unit, a presets file, two guard scripts) that
nothing here should be trusted just because it once worked — see §7's
`fallback-guard` cron job below, which exists specifically to keep catching
this class of drift automatically.

### Build llama.cpp

```bash
git clone https://github.com/ggerganov/llama.cpp ~/llama.cpp
cd ~/llama.cpp
cmake -B build -DGGML_BLAS=ON -DGGML_BLAS_VENDOR=OpenBLAS
cmake --build build --config Release -j$(nproc)
```

This will take several minutes on a 4-core ARM box — it's compiling, not
hanging.

✅ **Test it:**
```bash
~/llama.cpp/build/bin/llama-server --version
```

This produces `~/llama.cpp/build/bin/llama-server`, used by both models below.

### Inky (qwen35-tiny) — auxiliary tasks + health beacon

Download it into `~/models/`. Ubuntu 24.04's system Python blocks unmanaged
`pip install`s (PEP 668), so install the Hugging Face CLI with the escape
hatch this box actually used:

```bash
pip install -U huggingface_hub --break-system-packages
```

Then pull it — this box's own shell history confirms it came from
`unsloth`'s GGUF releases:

```bash
hf download unsloth/Qwen3.5-0.8B-GGUF Qwen3.5-0.8B-Q4_K_M.gguf --local-dir ~/models
```

(This box also has a few unrelated larger GGUFs — a `Qwen_Qwen3.5-9B`, some
Gemma/Qwen2.5 files, llama.cpp's own vocab test fixtures — sitting in
`~/models/` from earlier experiments. None of them are required for
anything this guide sets up; don't download them unless you have your own
reason to.)

✅ **Test it:**
```bash
ls -lh ~/models/Qwen3.5-0.8B-Q4_K_M.gguf   # a few hundred MB, not 0 bytes
```

Run it bound to localhost only (never expose this port publicly):

```bash
~/llama.cpp/build/bin/llama-server \
  -m ~/models/Qwen3.5-0.8B-Q4_K_M.gguf \
  --host 127.0.0.1 --port 45072 --alias Inky \
  --threads 4 --ctx-size 10240 --n-gpu-layers 0 \
  --cache-type-k q4_0 --cache-type-v q4_0
```

✅ **Test it:**
```bash
curl -s http://127.0.0.1:45072/health   # {"status":"ok"}
```

In practice this box runs it as a persistent background process rather than
inline in a terminal — wrap it in its own systemd user unit (same pattern
as §5; this box's actual unit is named `llama-qwen35-tiny.service`) if you
want it to survive reboots.

### gemma-4-E2B — the real fallback, behind a router

Unlike Inky, gemma isn't manually downloaded into `~/models/` — it's
fetched automatically by llama-server's own `--hf-repo` flag on first load,
from `google/gemma-4-E2B-it-qat-q4_0-gguf` on Hugging Face (cached under
`~/.cache/huggingface/hub/`). You don't need a separate download step for
it, just the flag in the launch command below.

It runs behind a **router** — `llama-server`'s own multi-model mode, which
can load more than one preset on demand — rather than a single dedicated
process like Inky's. Create `~/.config/systemd/user/llama-router.service`:

```ini
[Unit]
Description=llama.cpp router server (multi-model)
After=network.target

[Service]
Type=simple
WorkingDirectory=/home/huey/llama.cpp/build/bin
ExecStart=/home/huey/llama.cpp/build/bin/llama-server \
  --host 127.0.0.1 \
  --port 8080 \
  --jinja \
  -fa on \
  -t 4 \
  -ngl 0 \
  -c 65536 \
  --cache-type-k q4_0 \
  --cache-type-v q4_0 \
  --models-preset /home/huey/llama-presets.ini \
  --models-max 2 \
  --models-autoload \
  --timeout 3600

Restart=always
RestartSec=10

[Install]
WantedBy=default.target
```

`--models-preset` points at `~/llama-presets.ini`, which is where each
model's individual flags live (not the systemd unit — the router itself
stays generic). This box's actual file:

```ini
; ~/llama-presets.ini
; Use hyphenated keys matching CLI long names (no leading --)

[qwen35-tiny]
model    = /home/huey/models/Qwen3.5-0.8B-Q4_K_M.gguf
ctx-size = 4096
threads  = 4
reasoning-budget = 0

; gemma-4-E2B is discovered from the HF cache; this section only overrides
; its defaults. Hermes sends no temperature, so the server default applies —
; ai-gym found gemma fixes the simulated incident 3/3 at 0.3 vs 1/3 at 1.0.
; reasoning off: Hermes sends no thinking toggle, and thinking cost ~50 s per reply.
; load-on-startup: keep gemma resident so a failover never pays the ~16 s cold load.
; parallel = 1: see the cache-eviction incident above — do not remove this.
[google/gemma-4-E2B-it-qat-q4_0-gguf:IT]
temp = 0.3
reasoning = off
load-on-startup = true
parallel = 1
```

(A `[qwen35-fast]` section pointing at a deleted 4B model file is also
still in this box's actual `llama-presets.ini` — dead weight left over from
the 09-15 retirement, harmless, not reproduced here.)

```bash
systemctl --user daemon-reload
systemctl --user enable --now llama-router.service
```

✅ **Test it:**
```bash
systemctl --user status llama-router.service   # active (running)
# wait ~15-20s for load-on-startup to finish, then:
curl -s http://127.0.0.1:8080/v1/models | python3 -m json.tool | grep '"id"'
```

### Keeping gemma healthy: `fallback_guard.sh` + `fallback_warm.sh`

A plain systemd unit isn't enough — the 09-15 and 09-18 incidents above
both happened *underneath* services that reported themselves as healthy.
This box runs two scripts, sourced from `jibjabjog/hermes-config` and
registered as the `fallback-guard` cron job (§7 has the clone command and
exact `hermes cron create`), that actually verify the fallback works rather
than just that a port is open:

- **`fallback_guard.sh`** (run every 5 min via cron, §7) checks: is the
  router running (starts it if not); does the model actually answer a real
  completion, not just `/health` (restarts the router once if not); does
  `config.yaml`'s `fallback_model` still point at gemma (drift check —
  exactly what caught nothing back on 09-15, since nothing was checking at
  all); and, only when healthy and idle, triggers the warm-up below. Silent
  on Telegram when healthy; only speaks up on a state change.
- **`fallback_warm.sh`** rebuilds the exact system prompt + tool schemas
  Hermes would really send on a Telegram failover (reading the live
  session's stored prompt, offline — nothing is sent to Telegram) and feeds
  it to gemma as a 1-token completion, so the ~19k-token prefill is already
  cached *before* a real failover ever needs it. Runs detached so a cold
  prefill (~17-20 min) doesn't block the cron job.

✅ **Test it:**
```bash
~/.hermes/scripts/fallback_guard.sh -v   # prints a status line either way
tail -20 ~/.hermes/logs/fallback_guard.log
```
For genuine proof the cache is working (not just that the scripts run
without error), send the same payload twice and check the second call's
cache hit:
```bash
curl -s http://127.0.0.1:8080/v1/chat/completions -H "Content-Type: application/json" \
  --data-binary @~/.hermes/cache/warm_telegram.json \
  | python3 -c "import json,sys; t=json.load(sys.stdin)['timings']; print('cache_n:', t['cache_n'], '/', t.get('prompt_n'))"
```
A near-full cache hit (most of the prompt's tokens, not 0) confirms it —
this is the exact test that caught the 09-18→22 incident.

## 7. Hermes's internal scheduler — don't confuse it with `crontab`

**Important gotcha found while writing this guide:** Hermes's own
self-documentation (`huey-origins` README) lists these as "cron jobs" —
described in a way that reads like standard OS cron. **They are not in the
OS crontab.** Checking `crontab -l` on this box shows only two unrelated
jobs (`rat-backup.sh`, for a different backup system entirely).

Hermes's jobs actually live in its own internal scheduler, state at
`~/.hermes/cron/jobs.json`, driven by a ticker process inside the gateway —
not `cron(8)`. You manage these through Hermes itself (its `/cron` or
scheduler skill/command), not by editing `crontab -e`. If you're rebuilding
this and reach for `crontab -e` expecting to find Hermes's jobs, you won't —
that was a documentation imprecision from Hermes's self-audit, not a real
discrepancy in the running system once you know where to look.

The four Hermes-managed jobs on this box, for reference:

| Job | Schedule | Script | Delivery |
|---|---|---|---|
| Freerouter | `0 6 * * *` | `freerouter_failover.sh` | local (log file) |
| local-llama-ping | `*/15 * * * *` | `local_llama_ping.sh` | Telegram |
| hermes-backup | `0 3 * * *` | `backup_hermes.sh` | local |
| fallback-guard | `*/5 * * * *` | `fallback_guard.sh` | Telegram |

`fallback-guard` is the newest (added 2026-09-18 alongside the gemma
fallback, §6) — it's what actually keeps gemma healthy and its cache warm,
not just a health check.

Scripts live in `~/.hermes/scripts/` (see the script-provenance note below —
they're not part of the upstream `hermes-agent` clone). Register the jobs
through Hermes's own CLI (`hermes cron create`, aliased `add`) once the
scripts are in place — don't hand-edit `jobs.json` directly. The exact
commands that reproduce this box's four jobs:

```bash
hermes cron create "0 6 * * *" --name "Freerouter" \
  --script freerouter_failover.sh --no-agent --deliver local

hermes cron create "*/15 * * * *" --name "local-llama-ping" \
  --script local_llama_ping.sh --no-agent --deliver telegram

hermes cron create "0 3 * * *" --name "hermes-backup" \
  --script backup_hermes.sh --no-agent --deliver local

hermes cron create "*/5 * * * *" --name "fallback-guard" \
  --script fallback_guard.sh --no-agent --deliver telegram
```

`--no-agent` matters here: it means the script's stdout is delivered as-is
without ever routing through the LLM (a "classic watchdog pattern" per
`hermes cron create --help`) — appropriate for jobs that are pure
shell/health-check logic, not reasoning tasks.

✅ **Test it:**
```bash
hermes cron list                    # all 4 present, state "scheduled"
hermes cron run <job-id>            # force one to run now, don't wait for its schedule
hermes cron runs <job-id>           # check the result of that forced run
```

**Script provenance — these aren't upstream Hermes files.**
`freerouter_failover.sh`, `backup_hermes.sh`, `fallback_guard.sh`, and
`fallback_warm.sh` are all bespoke scripts written for this deployment;
their actual source is kept in the sibling repo
[`jibjabjog/hermes-config`](https://github.com/jibjabjog/hermes-config),
which also documents the required env vars
(`OPENROUTER_API_KEY`, `TELEGRAM_BOT_TOKEN`, `TELEGRAM_HOME_CHANNEL`) and has
its own troubleshooting section. Clone it and copy the scripts in:

```bash
git clone https://github.com/jibjabjog/hermes-config.git ~/.hermes-config
cp ~/.hermes-config/freerouter_failover.sh ~/.hermes/scripts/
cp ~/.hermes-config/backup_hermes.sh ~/.hermes/scripts/
cp ~/.hermes-config/fallback_guard.sh ~/.hermes/scripts/
cp ~/.hermes-config/fallback_warm.sh ~/.hermes/scripts/
chmod +x ~/.hermes/scripts/freerouter_failover.sh ~/.hermes/scripts/backup_hermes.sh \
  ~/.hermes/scripts/fallback_guard.sh ~/.hermes/scripts/fallback_warm.sh
```

`local_llama_ping.sh` is **not** in that repo — it isn't tracked anywhere
public as of this writing. Its actual behavior (read directly off this box):
a real `POST` to `127.0.0.1:45072/v1/chat/completions` (a live completion
request, not just a `/health` check — a stronger signal that inference
itself works, not just that the process is up), appending `"local llama
ok"` or `"local llama DOWN"` with a timestamp to
`~/.hermes/logs/local_llama_ping.log` depending on the result.

**This script was actually broken by the 2026-09-15 router retirement (§6)
until caught and fixed the same day** — it originally pointed at
`127.0.0.1:8080` with `"model":"qwen35-fast"` (the retired router/4B tier),
so the moment that router was stopped, every run started logging `DOWN`
even though Inky itself was perfectly healthy on `45072`. A monitoring
script that still checks a deliberately-retired endpoint will confidently
report the wrong thing forever — worth remembering any time you retire a
component that something else might be quietly depending on. Treat this
script as a real gap in this guide's reproducibility (recreate it by hand,
pointed at `45072`/`qwen35-tiny` as shown above), not an oversight to skip
past.

## 8. Integrations

### GitHub
Already covered in §3 — `gh auth login` as the bot account. Hermes's GitHub
skill/plugin (`evey-github`) then shells out to `gh` for repo/PR/issue
operations.

### Google Workspace
1. Create a project at [console.cloud.google.com](https://console.cloud.google.com)
   (top bar → project dropdown → "New Project"), then enable the Workspace
   APIs you need (**APIs & Services → Library** — search for Gmail API,
   Calendar API, Drive API, etc., enable each one individually).
2. **APIs & Services → OAuth consent screen** — set it up before creating
   credentials, or credential creation will nag you to. Choose **External**
   user type unless you have a Google Workspace organization. **The common
   gotcha:** a freshly-created consent screen defaults to "Testing" mode,
   which only allows sign-ins from email addresses you've explicitly added
   as test users — add your own Google account under **Audience → Test
   users**, or OAuth will fail with an "access blocked" error even though
   everything else is configured correctly.
3. **APIs & Services → Credentials → Create Credentials → OAuth client ID**
   — type **Desktop app**. Download the resulting JSON and save it as
   `~/.hermes/google_client_secret.json`.
4. Run through Hermes's `google-workspace` skill's OAuth flow; on success it
   writes `google_token.json` alongside it. Both files are secrets — never
   commit or copy them.
5. **Known local patch, worth doing proactively:** the stock
   `google-workspace` `SKILL.md` invokes a bare `python` to run its scripts.
   On this box bare `python` resolves to the *system* interpreter, which
   doesn't have the Google client libraries installed (only Hermes's venv
   does — see §4). The fix is editing that skill's `SKILL.md` to call
   `~/.hermes/hermes-agent/venv/bin/python` explicitly. A working copy of
   that edit is kept at `~/hermes-gws-SKILL.working.md` as a reference/backup,
   since a future `hermes update` can silently revert the live skill file
   back to upstream and reintroduce the bug.

✅ **Test it:**
```bash
~/.hermes/hermes-agent/venv/bin/python -c "import google.auth" && echo "libs present"
```
If this errors but the same line works when you drop `~/.hermes/hermes-agent/venv/bin/`
(i.e. it only works with the *system* Python), that's the exact symptom of
the patch above not being applied.

### Telegram
1. In Telegram, message **@BotFather**, send `/newbot`, and follow the
   prompts (choose a display name and a unique `_bot`-suffixed username).
   BotFather replies with a bot token — put it in `~/.hermes/.env` as
   `TELEGRAM_BOT_TOKEN` (env var name per this box's own backup docs,
   `jibjabjog/hermes-config`, see §12).
2. You also need a **chat ID** to act as the delivery target — message your
   new bot once, then message **@userinfobot** (or hit
   `https://api.telegram.org/bot<token>/getUpdates` and read the `chat.id`
   field back) to find your numeric ID. Set it as `TELEGRAM_HOME_CHANNEL` in
   `.env`.
3. This box uses that single "home channel" as the default delivery target
   for cron job notifications and alerts (e.g. the local-llama-ping health
   check posts failures here).

✅ **Test it:**
```bash
curl -s "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/getMe"
```
Returns `{"ok":true,"result":{...your bot's username...}}` if the token is
valid — a read-only check, doesn't message anyone. Saving an actual message
delivery test for §13, once a real cron job can trigger it.

### Voice messages: STT/TTS over Telegram

Separate from the bot token/chat ID above, but worth setting up
deliberately rather than leaving to defaults: this box transcribes incoming
Telegram voice notes and can speak its replies back — both **fully local**,
no cloud STT/TTS API key or cost involved.

**Speech-to-text** (`stt:` in `config.yaml`):
```yaml
stt:
  enabled: true
  provider: local
  local:
    model: tiny
    language: en
```
`provider: local` uses `faster-whisper` (already in Hermes's venv — see
§4). This box uses the `tiny` Whisper model: fastest and least accurate of
the five sizes (`tiny`/`base`/`small`/`medium`/`large-v3`), a deliberate
trade-off for CPU-only hardware where a bigger model would mean a
noticeably slower transcription. `language: en` skips auto-detection,
which otherwise frequently misidentifies short or accented clips — set it
to your own language code (or `""` for auto-detect) if you're not speaking
English to it. Voice notes get transcribed automatically; no extra
per-message setup needed beyond this config being present.

**Text-to-speech** (`tts:` in `config.yaml`):
```yaml
tts:
  provider: piper
voice:
  auto_tts: true
```
`piper` is a fully local, open-source TTS engine — again, no API key. Its
voice models (`.onnx` files) live in `~/.hermes/cache/piper-voices/`; this
box has `en_US-lessac` downloaded. `voice.auto_tts: true` means every text
reply is *also* spoken back as a voice note automatically, not just
transcribed replies to voice input — if you only want spoken replies when
the user sent voice, that's a different (undocumented on this box) knob to
look for, not the default here.

✅ **Test it:**
```bash
echo "test" | ~/.hermes/hermes-agent/venv/bin/piper \
  --model ~/.hermes/cache/piper-voices/en_US-lessac-low.onnx \
  --output_file /tmp/piper-test.wav
file /tmp/piper-test.wav   # should say "WAVE audio", not empty/error
```
That confirms Piper itself works, independent of Telegram. For the full
loop, send your bot a voice message on Telegram and confirm two things
come back: a text transcript, and a spoken voice-note reply.

## 9. Freerouter / failover scripting

`~/.hermes/scripts/freerouter.py` checks OpenRouter reachability/quota daily
and rotates `model.default`/vision/etc among available free models.
`freerouter_failover.sh` wraps that with a health check on `qwen35-tiny`
(Inky) specifically and a Telegram notification either way — it's about
Freerouter's own daily model rotation, not the primary `fallback_model`.

**Don't confuse this with `fallback-guard` (§6/§7).** Two separate
mechanisms, easy to conflate since both are about "the local model":
Freerouter/`freerouter_failover.sh` runs once a day, checks Inky, and
rotates which *OpenRouter* model is primary — it doesn't touch
`fallback_model` at all as of the 2026-09-15 fix (earlier versions used
`set_fallback_model.py` to flip `fallback_model` between two local tiers on
every run; that machinery is gone). `fallback-guard`/`fallback_guard.sh`
runs every 5 minutes, checks gemma specifically, and is what actually keeps
the real `fallback_model` (§6) healthy and its cache warm. `model_manager.py`
reads/writes Hermes's model-selection state files
(`~/.hermes/.model_fallback.json`, `.model_selection.json`) for Freerouter's
own rotation — also unrelated to `fallback_model`.
`freerouter_failover.sh`'s source is in `jibjabjog/hermes-config` (§7);
`local_llama_ping.sh` isn't tracked in any repo found on this box as of this
writing — treat that one as a reproducibility gap.

✅ **Test it:**
```bash
~/.hermes/scripts/freerouter_failover.sh dry   # the script's own test mode
```
Dry mode skips the live OpenRouter check and the gateway restart, so it's
safe to run any time — good for confirming the script and its dependencies
(qwen35-tiny reachable, Telegram configured) are wired up correctly before
trusting it to run unattended at 06:00.

## 10. Skills and plugins

Two separate extension mechanisms are in play, don't conflate them:

- **Skills** (`~/.hermes/skills/`) — markdown-driven capability packs (~25
  active on this box: github, email, research, devops, etc.), plus a handful
  symlinked in from `~/.agents/skills/` for dev-workflow skills (TDD,
  writing-plans, parallel-agent dispatching) shared with this Claude Code
  setup.
- **Plugins** (`~/.hermes/plugins/`) — the `evey-*` suite, 30 Python plugins
  giving Hermes autonomy/memory/scheduling/wallet/sandboxing capabilities
  (`evey-autonomy`, `evey-wallet`, `evey-sandbox`, `evey-memory-adaptive`,
  etc.). This is a much higher-stakes surface than skills — some of these
  (autonomy, wallet, sandbox) grant real-world agency and should get a
  dedicated security review before being trusted on a rebuild, not just
  installed and forgotten.

Both directories are populated by cloning/copying the relevant sources in —
this guide doesn't enumerate each one's origin since that's Hermes-specific
packaging, not host setup.

✅ **Test it:**
```bash
hermes skills list
hermes plugins list
```
Whatever you've installed so far should show up here — useful as a running
checklist while you work through recreating this box's set, and as a sanity
check after any future `hermes update` (§'s Known version drift below) that
a plugin didn't silently get disabled.

## 11. Self-hosted Supabase over Tailscale

A separate concern from Hermes itself, but integrated with it. The whole
point of putting Supabase behind Tailscale is reaching it privately from
*your other devices* — so this section covers both sides: the server, and
getting your own laptop/phone onto the same network. Skipping the second
half is the single most likely way to finish this section and then be
unable to reach anything.

### Create a Tailscale account

Sign up at [tailscale.com](https://tailscale.com) (the free **Personal**
plan covers up to 100 devices/3 users — plenty for this). You can sign in
with an existing Google, GitHub, or Microsoft account rather than creating
a new password. This account is what defines your **tailnet** — a private
mesh network only your logged-in devices can join.

### Connect the server

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```
`tailscale up` prints a login URL — open it, sign in with the account you
just created, and the command returns once the box has joined your tailnet.

✅ **Test it:**
```bash
tailscale status   # this box listed as "self" (or similar)
tailscale ip -4    # the tailnet IP other devices will use to reach it
```

### Connect your own devices

This is the part that's easy to skip and then wonder why nothing is
reachable. Install Tailscale and log into the **same account** on whatever
you want to reach this box from:

- **macOS / Windows:** download the app from
  [tailscale.com/download](https://tailscale.com/download), install it,
  sign in. It runs as a menu bar / system tray app — no command line
  needed.
- **iOS / Android:** install "Tailscale" from the App Store / Play Store,
  sign in.
- **Another Linux machine:** same one-liner as the server —
  `curl -fsSL https://tailscale.com/install.sh | sh && sudo tailscale up`.

✅ **Test it (from the device you just connected, not from this box):**
```bash
tailscale status   # should now list BOTH this box and the device you're on
ping <this-box's-tailscale-ip>   # from tailscale ip -4 above, e.g. 100.x.x.x
```
If `tailscale status` on your laptop/phone doesn't show this box, double
check both are signed into the same Tailscale account — a common mistake is
accidentally creating a second account rather than signing into the first.

### Set up Supabase

1. Set up self-hosted Supabase via Docker Compose (see
   `jibjabjog/self-hosted-supabase-tailscale` for the detailed guide this box
   itself produced, including first-setup failure modes).
2. Bind Supabase's exposed ports to the Tailscale interface / rely on
   Tailscale ACLs so it's reachable only at its tailnet IP
   (`100.124.0.62` on this box), never on the public internet.

   ✅ **Test it (from the device you connected above, not from this box):**
   ```bash
   curl -s http://<this-box's-tailscale-ip>:<supabase-port>/rest/v1/
   ```
   Should get a response from Supabase's REST endpoint. Also worth
   confirming it *fails* from a device **not** on the tailnet (e.g. your
   phone with WiFi/mobile data but Tailscale turned off) — that's the actual
   security property you're relying on.

This stack broke once during a Hermes update and was repaired — if you hit
phantom-directory or demo-JWT issues on first bring-up, that's a known
failure class here, not a sign of a bad install.

## 12. Backups

`backup_hermes.sh` (source in `jibjabjog/hermes-config`, §7; run daily by
Hermes's own scheduler, §7) syncs `~/.hermes/` config, scripts, and presets
to a dedicated GitHub backup repo — `jibjabjog/hermes-config` itself on this
box, not a separate `hermes-backup` repo despite that name appearing in
older internal notes. Set up a similar repo and point the script at it.
Before your first backup push, sanity-check that secrets are actually
excluded — `redact_config_secrets.py` in `~/.hermes/scripts/` exists
specifically to strip secrets from `config.yaml` before it's backed up; make
sure it's actually wired into the backup path rather than assumed. Also
double check the repo's actual GitHub visibility before relying on it as a
secrets boundary — `jibjabjog/hermes-config`'s own README describes itself
as "private," but it's visible as a **public** repo on this box; redaction
is the thing actually protecting you here, not repo visibility.

✅ **Test it:**
```bash
~/.hermes/scripts/backup_hermes.sh    # run it once by hand, don't wait for 03:00
tail -20 ~/.hermes/logs/backup.log
```
Then actually open the pushed `config.yaml` in the backup repo on GitHub and
confirm the secret-shaped fields read as redacted placeholders, not real
values — don't just trust that the script ran without error.

## 13. Verifying the rebuild

```bash
systemctl --user status hermes-gateway.service   # active (running)
systemctl --user status llama-router.service     # active (running) -- gemma
curl -s localhost:45072/health                   # Inky (auxiliary tasks + health beacon)
curl -s localhost:8080/v1/models                 # the router -- gemma should be listed
hermes fallback list                             # confirms gemma is the live fallback target
gh auth status                                    # bot account logged in
tailscale status                                  # this box + Supabase reachable
hermes cron list                                  # all 4 jobs present, "enabled"
hermes cron run <job-id>                          # force one job now, then check its log
```

Beyond process/service checks, confirm the integrations actually work end to
end, not just that credentials are present:
- **Google Workspace:** trigger any `google-workspace` skill action (e.g.
  list recent Calendar events) and confirm it succeeds rather than failing
  with an import error — that specific failure mode means the venv-python
  patch from §8 didn't take.
- **Telegram:** force-run `local-llama-ping` (`hermes cron run <job-id>`) and
  confirm a message actually arrives in the chat you set as
  `TELEGRAM_HOME_CHANNEL`.
- **The gemma fallback, for real:** don't just trust `hermes fallback list`
  — §6 and §9's own history is a warning that "looks configured" and "works"
  are different things on this box specifically. Confirm the cache is
  actually being reused (the two-command test at the end of §6), not just
  that the process is up.

---

## Known version drift (as of this write-up)

Installed Hermes is **0.20.5**, ~71 commits behind the `main` branch's
**0.20.6** (released 2026-08-27). None of what follows is a bug in the
rebuild above — it's just what this specific deployment does and doesn't
use, broken out by *why* rather than dumped in one list, since that list
turned out to have two outright errors in it (corrected below).

**Two corrections to an earlier version of this section** — checked
against the live config and process state rather than assumed:
- **Voice (STT/TTS) is fully configured, not absent.** Speech-to-text
  (`faster-whisper`, local) and text-to-speech (Piper, local) are both
  installed and wired up — see §8's Telegram section below for the actual
  setup and a functional test. This deployment talks back.
- **Checkpoints v2 is enabled and in active use, not "not referenced."**
  `checkpoints.enabled: true` in `config.yaml`, and
  `~/.hermes/checkpoints/store` is a real, actively-updated shadow git repo
  — Hermes snapshots the working directory before `write_file`/`patch`/
  `terminal` calls so `/rollback` has something to restore. Run `hermes
  checkpoints status` to see its size and what's tracked.

**Genuinely not configured here** — available upstream, not turned on:
- **Bot Mode** — named specialist bots with persistent chats/routines/group
  chats/`@mentions`. This box runs one plain Hermes profile, not a bot
  roster.
- **Mixture-of-Agents (MoA) preset** — `hermes moa` lets you define a named
  preset that fans one prompt out to several reference models and
  aggregates the result; none configured here (and would be a poor fit for
  a CPU-only box regardless — each MoA call is *N* model calls, not one).
- **Subscription Proxy / Nous Portal** — a paid, zero-config alternative to
  this box's per-service setup (OpenRouter key, `gh` auth, Google OAuth,
  etc.): one OAuth login covers a model provider plus web search, image
  generation, TTS, and browser automation together. Not used here because
  the per-service path this guide walks through is free.
- **External memory providers (Honcho, Mem0, and six others)** — plugin
  backends for cross-session user modeling beyond Hermes's built-in
  `MEMORY.md`/`USER.md` system. `memory.provider: ''` here — built-in
  memory only.
- **Image *generation*** (FAL.ai-backed, text-to-image output) — a separate
  tool from the vision *input* this box does have configured
  (`auxiliary.vision.model`, used to actually look at images sent to it).
  Generation isn't enabled; understanding images sent to the bot already
  works.
- **Multi-agent Kanban orchestration** — a board UI for fanning a task out
  across multiple agent workers. `delegation.orchestrator_enabled: true` is
  set (the underlying single-level delegation feature works), but nothing
  populates `~/.hermes/kanban.db` — this deployment hasn't set up
  multi-agent task boards.

**Not applicable to a headless server, not really "missing":**
- **Wake word** ("Hey Hermes" hands-free voice trigger) and **Pets/Petdex**
  (animated mascots) — both are CLI/TUI/desktop-app UI features. There's no
  local microphone or window to speak to or show a mascot in on a
  server you only reach over SSH/Telegram.
- **The desktop app itself** — same reason.
- **`/loop` recurring loops** — worth clarifying rather than filing next to
  the others: `/loop` and this box's cron jobs (§7) aren't really
  alternatives for the same job. `/loop` is *session-scoped* — "keep
  polling this while my current conversation is open" — and the docs
  themselves say to use cron instead for anything that needs to run
  unattended, overnight, surviving restarts. This box's Freerouter/
  llama-ping/backup jobs are exactly that kind of unattended work, so cron
  was always the right tool here, not a workaround for a missing feature.

---

## Why "Huey"?

![Huey, a three-legged maintenance drone, walking through the cargo hold of the Valley Forge](huey2.webp)

Not a Hermes feature — just the naming. The bot account, the user account,
this whole box, all take their name from one of the drone robots in
*Silent Running*.

**Silent Running (1972)**, directed by Douglas Trumbull: in a future where
Earth's forests have died out, botanist Freeman Lowell tends the last
surviving forest ecosystems, preserved inside geodesic domes aboard the
space freighter *Valley Forge*. When orders arrive to jettison and destroy
the domes so the fleet can return to commercial service, Lowell rebels —
he kills his crewmates to save the forest and flees into the rings of
Saturn, with only three drone robots for company.

> Huey is one of those three drones aboard the *Valley Forge* — alongside
> Dewey and Louie — originally built for simple maintenance and surgical
> work. An accident costs him a leg partway through, and he's left walking
> on two rather than three or four, giving him a distinct, slightly
> lopsided gait (the pose in the still above). Reprogrammed by Lowell to
> tend the forest and even play poker, Huey becomes his only real
> companion in total isolation — a mute, faceless machine that somehow
> reads as more alive than anyone else left in the story.
