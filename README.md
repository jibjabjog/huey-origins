# Setting Up This Box From Scratch — A Beginner's Guide

**Purpose:** a from-nothing-to-here walkthrough for rebuilding this exact
environment: a headless Hermes Agent deployment on an Oracle Cloud ARM64 VM,
with a local CPU LLM failover, GitHub/Google/Telegram integrations, and a
self-hosted Supabase over Tailscale.

**Audience:** someone who has never touched this box before. Every step
assumes a fresh `huey` user account and an empty home directory.

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

## 0. What you end up with

- Oracle Cloud "Always Free" ARM64 VM, Ubuntu 24.04, 4 OCPU / 24 GB RAM, no GPU
- Hermes Agent (Nous Research) running as a `systemd --user` background service
- A local llama.cpp build providing a CPU-only fallback model when OpenRouter
  is unreachable or rate-limited
- GitHub, Google Workspace, and Telegram wired into Hermes
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
   - Add your SSH public key at creation time.
3. Open the security list / NSG for at least SSH (22). Everything else
   (Hermes's gateway, Supabase, llama-server) will be bound to `127.0.0.1` or
   reached over Tailscale, not exposed publicly — don't open extra ports.
4. SSH in as `ubuntu` (or your chosen default user), and create the `huey`
   user if it isn't the default:
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
personal one — keeps its commits/PRs attributable and scoped.

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
Hermes falls back to a model running locally on CPU. This box runs exactly
**one** local model for this — `qwen35-tiny` ("Inky", `Qwen3.5-0.8B-Q4_K_M.gguf`)
— and it's the target for `fallback_model` *and* every `auxiliary.*`
sub-config in `~/.hermes/config.yaml` that needs a cheap local model
(compression, skills_hub, approval, mcp, title_generation, triage_specifier,
kanban_decomposer, profile_describer, curator, web_extract, session_search).

**This box used to run a second, larger local tier** (a 4B model,
`qwen35-fast`, behind a multi-model router on port 8080) as the "everyday"
fallback, with Inky reserved as a deeper emergency-only backup. Don't
reproduce that design — it's why this section only describes one model now:
- The router setup drifted out of sync with `config.yaml` over time:
  `fallback_model` ended up pointing at a port nothing listened on anymore,
  and 10 of 11 `auxiliary.*` sub-configs had a placeholder API key pointed
  at the *real* OpenRouter cloud instead of the local router — silently
  broken for weeks before anyone noticed, confirmed via daily config
  backups in `jibjabjog/hermes-config`.
- Running a 4B model with any real intent is also a bad fit for a CPU-only
  box like this one — it sat there costing ~5.7GB RSS while mostly not even
  being reachable correctly.
- The fix (2026-09-15): retire the second tier entirely, repoint everything
  at Inky, delete the 4B model file. One model, one config value
  (`http://127.0.0.1:45072/v1`) referenced everywhere it's needed — much
  harder for this kind of drift to happen unnoticed again.

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

This produces `~/llama.cpp/build/bin/llama-server`.

Download the one GGUF-quantized model you need into `~/models/`. Ubuntu
24.04's system Python blocks unmanaged `pip install`s (PEP 668), so install
the Hugging Face CLI with the escape hatch this box actually used:

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
as §5, and this box's actual unit is named `llama-qwen35-tiny.service`) if
you want it to survive reboots. This is the *only* local `llama-server`
process this box runs — there's no separate router or second model to
stand up.

## 7. Hermes's internal scheduler — don't confuse it with `crontab`

**Important gotcha found while writing this guide:** Hermes's own
self-documentation (`huey-origins` README) lists three "cron jobs" —
Freerouter (daily 06:00), local-llama-ping (every 15 min), hermes-backup
(daily 03:00) — described in a way that reads like standard OS cron. **They
are not in the OS crontab.** Checking `crontab -l` on this box shows only two
unrelated jobs (`rat-backup.sh`, for a different backup system entirely).

Hermes's jobs actually live in its own internal scheduler, state at
`~/.hermes/cron/jobs.json`, driven by a ticker process inside the gateway —
not `cron(8)`. You manage these through Hermes itself (its `/cron` or
scheduler skill/command), not by editing `crontab -e`. If you're rebuilding
this and reach for `crontab -e` expecting to find Hermes's jobs, you won't —
that was a documentation imprecision from Hermes's self-audit, not a real
discrepancy in the running system once you know where to look.

The three Hermes-managed jobs on this box, for reference:

| Job | Schedule | Script | Delivery |
|---|---|---|---|
| Freerouter | `0 6 * * *` | `freerouter_failover.sh` | local (log file) |
| local-llama-ping | `*/15 * * * *` | `local_llama_ping.sh` | Telegram |
| hermes-backup | `0 3 * * *` | `backup_hermes.sh` | local |

Scripts live in `~/.hermes/scripts/` (see the script-provenance note below —
they're not part of the upstream `hermes-agent` clone). Register the jobs
through Hermes's own CLI (`hermes cron create`, aliased `add`) once the
scripts are in place — don't hand-edit `jobs.json` directly. The exact
commands that reproduce this box's three jobs:

```bash
hermes cron create "0 6 * * *" --name "Freerouter" \
  --script freerouter_failover.sh --no-agent --deliver local

hermes cron create "*/15 * * * *" --name "local-llama-ping" \
  --script local_llama_ping.sh --no-agent --deliver telegram

hermes cron create "0 3 * * *" --name "hermes-backup" \
  --script backup_hermes.sh --no-agent --deliver local
```

`--no-agent` matters here: it means the script's stdout is delivered as-is
without ever routing through the LLM (a "classic watchdog pattern" per
`hermes cron create --help`) — appropriate for jobs that are pure
shell/health-check logic, not reasoning tasks.

✅ **Test it:**
```bash
hermes cron list                    # all 3 present, state "scheduled"
hermes cron run <job-id>            # force one to run now, don't wait for its schedule
hermes cron runs <job-id>           # check the result of that forced run
```

**Script provenance — these aren't upstream Hermes files.**
`freerouter_failover.sh` and `backup_hermes.sh` are bespoke scripts written
for this deployment; their actual source is kept in the sibling repo
[`jibjabjog/hermes-config`](https://github.com/jibjabjog/hermes-config),
which also documents the required env vars
(`OPENROUTER_API_KEY`, `TELEGRAM_BOT_TOKEN`, `TELEGRAM_HOME_CHANNEL`) and has
its own troubleshooting section. Clone it and copy the scripts in:

```bash
git clone https://github.com/jibjabjog/hermes-config.git ~/.hermes-config
cp ~/.hermes-config/freerouter_failover.sh ~/.hermes/scripts/
cp ~/.hermes-config/backup_hermes.sh ~/.hermes/scripts/
chmod +x ~/.hermes/scripts/freerouter_failover.sh ~/.hermes/scripts/backup_hermes.sh
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
1. Create a Google Cloud project, enable the Workspace APIs you need (Gmail,
   Calendar, Drive, etc.).
2. Create OAuth 2.0 credentials, download as `google_client_secret.json` into
   `~/.hermes/`.
3. Run through Hermes's `google-workspace` skill's OAuth flow; on success it
   writes `google_token.json` alongside it. Both files are secrets — never
   commit or copy them.
4. **Known local patch, worth doing proactively:** the stock
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

## 9. Freerouter / failover scripting

`~/.hermes/scripts/freerouter.py` checks OpenRouter reachability/quota daily
and rotates `model.default`/vision/etc among available free models.
`freerouter_failover.sh` wraps that with a health check on `qwen35-tiny`
and a Telegram notification either way. As of the 2026-09-15 fix (§6),
**it no longer touches `fallback_model`** — that's now permanently pinned
to `qwen35-tiny`/Inky regardless of Freerouter's daily result, so there's
nothing left to swap. Earlier versions of this script (and this guide) used
`set_fallback_model.py` to flip `fallback_model` between a `qwen35-fast`
tier and `qwen35-tiny` on every run; that machinery is gone along with the
second tier. `model_manager.py` still reads/writes Hermes's model-selection
state files (`~/.hermes/.model_fallback.json`, `.model_selection.json`) for
Freerouter's own `model.default` rotation, unrelated to the fallback fix.
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

A separate concern from Hermes itself, but integrated with it. Broad shape:

1. Install Tailscale, join this box to your tailnet:
   ```bash
   curl -fsSL https://tailscale.com/install.sh | sh
   sudo tailscale up
   ```
   `tailscale up` prints a login URL — open it, authenticate, and the
   command returns once the box has joined your tailnet.

   ✅ **Test it:**
   ```bash
   tailscale status   # this box listed, plus any other devices on your tailnet
   tailscale ip -4    # the tailnet IP other devices will use to reach it
   ```
2. Set up self-hosted Supabase via Docker Compose (see
   `jibjabjog/self-hosted-supabase-tailscale` for the detailed guide this box
   itself produced, including first-setup failure modes).
3. Bind Supabase's exposed ports to the Tailscale interface / rely on
   Tailscale ACLs so it's reachable only at its tailnet IP
   (`100.124.0.62` on this box), never on the public internet.

   ✅ **Test it (from another device on the same tailnet, not from this
   box):**
   ```bash
   curl -s http://<this-box's-tailscale-ip>:<supabase-port>/rest/v1/
   ```
   Should get a response from Supabase's REST endpoint. Also worth
   confirming it *fails* from a device **not** on the tailnet — that's the
   actual security property you're relying on.

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
curl -s localhost:45072/health                   # the local fallback model (Inky)
hermes fallback list                             # confirms qwen35-tiny is the live fallback target
gh auth status                                    # bot account logged in
tailscale status                                  # this box + Supabase reachable
hermes cron list                                  # all 3 jobs present, "enabled"
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

---

## Known version drift (as of this write-up)

Installed Hermes is **0.20.5**, ~71 commits behind the `main` branch's
**0.20.6** (released 2026-08-27). Features documented upstream but not
present in this build: Bot Mode, wake-word voice, Pets/Petdex, an active
Mixture-of-Agents preset, the desktop app (N/A on a headless server anyway),
Subscription Proxy/Nous Portal, a configured TTS pipeline (Whisper STT is
present; TTS isn't), `/loop` recurring loops (this box uses its own cron
ticker instead, §7), external memory providers (Honcho/Mem0 — only built-in
SQLite memory is used), Checkpoints v2/rollback, image generation input, and
multi-agent Kanban orchestration (the DB table exists but nothing populates
it). None of these are bugs in the rebuild above — they're just not part of
what this deployment currently uses.
