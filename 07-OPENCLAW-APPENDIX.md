# 07 — OpenClaw Appendix

`{OpenClaw-agent}` is the second AI in the triad — a GPT-based agent running OpenClaw on `{OpenClaw-machine}`. This appendix covers everything needed to set up and operate `{OpenClaw-agent}` as a PAI companion on **macOS**. It follows the 8-section template adapted for portability.

> **On host OS:** The reference triad runs OpenClaw on macOS — installed as a global npm package against Homebrew node, services managed by `launchctl`. This is the validated path. If you want to run OpenClaw on a Linux/VPS host, see the **Alternate hosts** sidebar at the end of this appendix; the structural concepts are the same but the package manager, service manager, and PATH conventions differ, and that path is not currently validated against the live OpenClaw release.

---

## Section 1: Machine Preparation

`{OpenClaw-machine}` is a Mac running macOS, dedicated to (or shared with) `{OpenClaw-agent}`. A Mac mini works well — small, silent, low-power, and runs 24/7. The agent runs as its **own macOS user account** so its workspace, secrets, and LaunchAgent are isolated from the admin user (who owns Homebrew).

**Two-user layout** (described in detail in `01-HOST-AND-NETWORK.md`):

| User | Owns | Used for |
|------|------|----------|
| `admin` | `/opt/homebrew` (Homebrew + global npm packages) | Installing/updating the OpenClaw binary |
| `{OpenClaw-agent-lowercase}` | `~`, `~/.openclaw/`, `{Agent-workspace-dir}`, `~/Library/LaunchAgents/` | Running the agent |

**Post-install essentials (as `admin`):**

```bash
# Install Homebrew if not present
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Install node (OpenClaw runs on the Homebrew node)
brew install node

# Useful CLI tools
brew install jq ripgrep htop tmux

# Tailscale (for remote SSH from {PAI-machine} when off-LAN)
brew install --cask tailscale
```

**Per-user setup** (run on both `admin` and `{OpenClaw-agent-lowercase}`):

```bash
# Enable Remote Login: System Settings → General → Sharing → Remote Login
# (or use kickstart):
sudo systemsetup -setremotelogin on

# Stay-awake (run on whichever user is logged in physically; pmset is system-wide)
sudo pmset -c sleep 0 displaysleep 10 disksleep 0
sudo pmset -c womp 1
```

**FileVault:** Enable in System Settings → Privacy & Security → FileVault. The agent's workspace contains conversation history, identity documents, and task state — encryption at rest is non-negotiable.

---

## Section 2: OpenClaw Installation

OpenClaw is installed as a **global npm package** from the public registry. Install once as `admin` (because `/opt/homebrew` is owned by `admin:staff`); every user account on the host shares the same binary at `/opt/homebrew/lib/node_modules/openclaw/`.

```bash
# As admin on {OpenClaw-machine}:
ssh admin@{OpenClaw-machine} "export PATH=/opt/homebrew/bin:\$PATH && \
  npm install -g openclaw"

# Verify
ssh admin@{OpenClaw-machine} "export PATH=/opt/homebrew/bin:\$PATH && \
  openclaw --version"
```

**Important: use `npm install -g`, NOT `npm update -g`.** The latter does not actually upgrade the package version — it only removes transitive deps without changing OpenClaw itself. Always `npm install -g openclaw@latest` (or `@<specific-version>`) for upgrades.

**npm safety lock:** I keep `min-release-age=3` in `/Users/admin/.npmrc` to block packages younger than 3 days (defense against publish-and-pull supply chain attacks). For an OpenClaw release older than 3 days you don't need to touch it. For something newer, temporarily bypass and restore:

```bash
npm config set min-release-age 0
npm install -g openclaw@<version>
npm config set min-release-age 3
```

**PATH gotcha for non-login SSH:** When `{PAI-machine}` SSHes into the agent user with a non-login shell, `/opt/homebrew/bin` is NOT in `$PATH`. Always prefix automation commands with `export PATH=/opt/homebrew/bin:$PATH` — for example:

```bash
ssh {OpenClaw-agent-lowercase} "export PATH=/opt/homebrew/bin:\$PATH && openclaw --version"
```

**Initial configuration** (as the agent user, interactive SSH):

```bash
ssh {OpenClaw-agent-lowercase}
# Inside the SSH session:
export PATH=/opt/homebrew/bin:$PATH
openclaw configure
# Interactive wizard:
#   - OAuth flow for openai-codex provider (ChatGPT Plus subscription)
#   - Or paste OpenAI/Anthropic/Gemini API keys
#   - Set primary model + fallback chain
#   - Enable Telegram channel + paste {telegram-bot-token}
```

**Why interactive matters:** `openclaw configure` is the **only** auth recovery tool that handles the openai-codex OAuth flow. Don't try to scrape token files or run it via automated SSH — the OAuth handshake needs a real browser hand-off.

**Verify:** `~/.openclaw/openclaw.json` exists, contains your model and channel configuration, and `auth-profiles.json` shows non-expired tokens for each provider.

**SAFETY: Never run gateway-connected CLI commands via automated SSH.** `openclaw status`, `openclaw models list`, `openclaw cron list`, and `openclaw sessions list` can trigger token refresh on the gateway. If automation runs them concurrently with the gateway daemon, both processes write to `auth-profiles.json` and corrupt it. Run those commands from interactive SSH only, or read state files directly.

---

## Section 3: Gateway Service

> **Diagram:** see [`diagrams/gateway-trust-boundary.md`](diagrams/gateway-trust-boundary.md) — SSH-tunnel-to-loopback trust boundary, two-factor authentication (SSH key + gateway token).

The gateway is `{OpenClaw-agent}`'s runtime control plane — it enables programmatic access, inter-agent communication, and Telegram integration. It's the single process that keeps `{OpenClaw-agent}` alive and reachable. On macOS it runs under `launchd` as a per-user LaunchAgent.

**Install the LaunchAgent** (as the agent user):

```bash
ssh {OpenClaw-agent-lowercase} "export PATH=/opt/homebrew/bin:\$PATH && \
  openclaw gateway install --force"
```

`gateway install --force` writes `~/Library/LaunchAgents/ai.openclaw.gateway.plist` and auto-loads it via `launchctl bootstrap`. Use `--force` after every OpenClaw upgrade — the plist is regenerated to match the new binary path.

**Health check:**

```bash
ssh {OpenClaw-agent-lowercase} "curl -s http://127.0.0.1:{gateway-port}/health"
# Expected: {"ok":true,"status":"live"}
```

(My `{gateway-port}` is the OpenClaw default — see your `VARIABLES.md`.)

**Restart cleanly** (without unloading first):

```bash
ssh {OpenClaw-agent-lowercase} "launchctl kickstart -k gui/\$(id -u)/ai.openclaw.gateway"
```

**Stop the gateway** (e.g., before an upgrade):

```bash
ssh {OpenClaw-agent-lowercase} "launchctl bootout gui/\$(id -u)/ai.openclaw.gateway"
```

**Re-bootstrap after a `bootout`** (only needed if the service is no longer loaded):

```bash
ssh {OpenClaw-agent-lowercase} "launchctl bootstrap gui/\$(id -u) \
  ~/Library/LaunchAgents/ai.openclaw.gateway.plist"
```

**Important:** `kickstart -k` only works if the service is already loaded. If you `bootout` and then try `kickstart`, you'll get an error — `bootstrap` it first, then `kickstart -k` works again.

**Security hardening:**
- The gateway binds to **loopback only** (`127.0.0.1`) — this is the default and should never be changed
- All remote access from `{PAI-machine}` is via SSH tunnel (LAN or Tailscale), never via a directly-exposed port
- Token authentication (`{gateway-token}`) is configured in `~/.openclaw/openclaw.json` and required on every WebSocket handshake
- Never expose the gateway port to the network with port forwarding, reverse proxies, or `localhost.run`-style tunnels

**Important:** Do NOT broadly expose the gateway port. Loopback + SSH tunnel is the entire security model.

---

## Section 4: Identity, Soul Documents, and Workspace Layout

`{OpenClaw-agent}` needs identity configuration, just like `{PAI-agent}` has DAIDENTITY.md. On macOS the agent's runtime workspace lives at `{Agent-workspace-dir}` under the agent user's home directory, and identity files are inside that workspace (so they sync along with everything else the agent produces).

**Workspace layout:**

```
{Agent-workspace-dir}/                # e.g., ~/agent-workspace
├── SOUL.md                # Core identity, values, behavioral guidelines
├── IDENTITY.md            # Pronouns, voice conventions, third-person rules
├── MEMORY.md              # Long-term memory — what the agent knows about itself and the work
├── USER.md                # Information about {principal} (preferences, projects, context)
├── CURRENT-TASK.md        # Durable task state (slim — see 08-TASKING-PROTOCOL.md)
├── HEARTBEAT.md           # Checklist the heartbeat cron reads each cycle
├── comms/
│   ├── inbox/             # Briefs from {PAI-agent} (file-based tasking)
│   ├── archive/           # Processed briefs (audit trail)
│   └── COMMS-LOG.md       # Auto-appended channel log
├── wip/<task-slug>/       # Active work artifacts, one subdirectory per task
├── drafts/                # Completed work awaiting review
├── notes/                 # Scratch space; intake checks here too
└── memory/
    └── task-history.md    # Older state history (trimmed from CURRENT-TASK.md)
```

**Why this shape:** Identity files (SOUL, IDENTITY, MEMORY, USER) live alongside operational state (CURRENT-TASK, HEARTBEAT, comms, wip) inside a single durable directory that survives gateway restarts, OpenClaw upgrades, and session compaction. OpenClaw's compaction logic re-injects SOUL/IDENTITY/MEMORY after every compaction cycle so the agent never forgets who it is.

**Syncing the workspace:**

The entire workspace syncs bidirectionally via Syncthing between `{OpenClaw-machine}:{Agent-workspace-dir}/` and `{PAI-machine}:{Sync-OpenClaw-dir}/`. This lets `{PAI-agent}` read identity files, write briefs into `comms/inbox/`, and pull completed artifacts from `wip/` and `drafts/`.

**Warning — do NOT sync these directories:**

- `~/.openclaw/` — contains `auth-profiles.json` (OAuth tokens), `openclaw.json` (gateway token, API keys), and other secrets. **Never share this directory.**
- `~/Library/LaunchAgents/` — local-only; LaunchAgent plists are host-specific
- `~/Backups/` — backups can sync to a separate one-way folder (see `04-BACKUP-AND-RECOVERY.md`), not into the workspace folder

The rule: **only `{Agent-workspace-dir}` itself is bidirectionally synced.** Everything else stays local to the host.

---

## Section 5: Telegram Integration

Telegram gives `{principal}` mobile access to `{OpenClaw-agent}` independent of `{PAI-agent}`. It also serves as the visible escalation channel from `{OpenClaw-agent}` back to `{principal}` when `{PAI-agent}` needs to relay something.

**Setup:**

1. Create bot via @BotFather and create the inter-agent group with three topics — see `SECRETS-SETUP.md` sections 6 and 7
2. Configure the Telegram channel in `~/.openclaw/openclaw.json` (or use `openclaw configure` interactively):
   - `telegram.token` = `{telegram-bot-token}`
   - `telegram.dmPolicy` = `pairing` (DMs require an explicit pairing handshake — only `{Principal-Telegram-User-ID}` is allowlisted)
   - `telegram.groupPolicy` = `allowlist` (the bot only accepts inbound from groups it's been allowlisted into; `{Inter-Agent-Group-ID}` is the one)
   - `telegram.streaming` = `partial` (live preview of replies in DMs and topics)
3. Restart the gateway: `launchctl kickstart -k gui/$(id -u)/ai.openclaw.gateway` (Telegram is loaded as a built-in channel extension, not a separate process)
4. Pair `{principal}`'s account: from `{principal}`'s phone, DM `{telegram-bot}` and follow the pairing handshake the bot prompts for

**Verify:** Send a DM to `{telegram-bot}` from `{principal}`'s phone — the bot responds. Then send a DM from a non-allowlisted account — the bot rejects it. (`dmPolicy=pairing` enforced.)

**Message routing:**

- **DM from `{principal}` → `{OpenClaw-agent}`** processes and responds in the DM thread
- **`{PAI-agent}` via inter-agent channel with `--topic <name>` flag** → `{OpenClaw-agent}` posts into the matching topic in `{Inter-Agent-Group-ID}` (Relay / Ops / Alerts — see `05-COMMUNICATION-PROTOCOLS.md` § Telegram Topic Architecture)
- **DMs from non-allowlisted accounts** → rejected by `dmPolicy=pairing`
- **Messages in groups other than `{Inter-Agent-Group-ID}`** → ignored by `groupPolicy=allowlist`

**Note on `groupPolicy`:** With `allowlist` policy, the bot can still send *outbound* messages to any group it's a member of. The policy controls *inbound* only. There is no `groupAllowlist` config key — the allowlist is implicit in which groups the bot has been added to and which group ID the agent's outbound posts target.

---

## Section 6: Memory Search and Embeddings

For semantic search over MEMORY.md and the agent's task history, `{OpenClaw-agent}` runs an embeddings backend. OpenClaw supports several providers — pick whichever matches your privacy / cost / latency preferences.

**Provider options** (configured in `~/.openclaw/openclaw.json` under `memorySearch`):

| Provider | Where it runs | Trade-off |
|----------|--------------|-----------|
| `gemini` (memory-core plugin) | Google's API | Easy, fast, requires a Gemini API key, sends embeddings off-machine |
| Local GGUF via node-llama-cpp | On-device | No API costs, no data egress, but downloads a GGUF model on first use and uses local compute |
| Cloud embedding providers (OpenAI, Cohere, etc.) | Their API | Standard tradeoff — costs money, sends data off-machine |

The current reference triad uses `gemini` because it's fast and the agent's memory volume is modest. If you'd rather keep all embeddings on-device, switch to the local GGUF backend; expect a one-time model download (interrupt-safe — clear any `.ipull` files in the cache and restart the gateway to resume).

**Provider evolution:** The embedding backend has changed between OpenClaw releases — always check the `memorySearch` key in `~/.openclaw/openclaw.json` for what's actually configured. The CLI `openclaw configure` wizard is the safe way to switch providers; manual edits work but get clobbered by `openclaw doctor --fix` if the schema changes.

**Verify:** Ask `{OpenClaw-agent}` to recall something from MEMORY.md ("What do you remember about X?") and confirm the response cites the matched chunk.

---

## Section 7: Heartbeat, Cron, and Monitoring

`{OpenClaw-agent}` runs a periodic **heartbeat** — a self-driven health check + intake scan that fires on a fixed interval. The heartbeat is OpenClaw's primary autonomy primitive. Tasking does NOT happen through the heartbeat (see `08-TASKING-PROTOCOL.md` for why); the heartbeat is for *health*.

### Heartbeat Architecture

Two-part system:

| Part | Where | Purpose |
|------|-------|---------|
| **Trigger** | `~/.openclaw/openclaw.json` → `agents.defaults.heartbeat` | Interval, model, session target |
| **Instructions** | `{Agent-workspace-dir}/HEARTBEAT.md` | Checklist the heartbeat reads each cycle |

Configure the trigger via `openclaw configure` or by editing `openclaw.json` directly:

```json
"agents": {
  "defaults": {
    "heartbeat": {
      "interval": "{Heartbeat-Interval}",
      "model": "openai-codex/gpt-5.2",
      "session": "main"
    }
  }
}
```

**Field choices:**
- **`interval`** — the cadence (`60m` is a good default; shorter wastes tokens, longer means slower intake)
- **`model`** — use the cheapest model that can handle the heartbeat checklist. GPT-5.2 on Codex OAuth works
- **`session`** — `main` reuses the agent's primary session so the heartbeat has context of prior work; `isolated` creates a fresh session each cycle (cheaper but no continuity)

### What the Heartbeat Does

`HEARTBEAT.md` is a strict checklist (~60 lines max — it's a procedure, not a knowledge base). Each cycle, the heartbeat:

1. **Read CURRENT-TASK.md status:**
   - `IDLE` → proceed to intake
   - `IN_PROGRESS` → check resume loop logic (see below)
   - `COMPLETE` / `REVIEW` → skip new work, run health checks only
   - `BLOCKED` → apply backoff escalation
2. **Intake scan:** Look at `{Agent-comms-inbox}/` for new `*.md` briefs from `{PAI-agent}`
   - If found, read highest-priority brief, set CURRENT-TASK.md to `IN_PROGRESS`, begin work
   - After processing, move the brief to `{Agent-comms-archive}/` with a date prefix
3. **Health checks:** Verify Syncthing is healthy, output directories exist, disk space is adequate
4. **Post heartbeat ack to Ops topic** — but **only if something noteworthy happened**. Silence is the default. A no-news heartbeat does not post.

### What the Heartbeat Does NOT Do

- **Drive task progress.** If the agent is `IN_PROGRESS`, the heartbeat checks for stalls but does not try to advance the work.
- **Pick up new work when status is `REVIEW` or `COMPLETE`.**
- **Post to Relay** (the sacred topic). The heartbeat only escalates to Relay when a `BLOCKED` task crosses the 1-hour threshold.

### Resume Loop Detection (Critical)

The heartbeat compares `last_checkpoint_utc` in CURRENT-TASK.md against the previous heartbeat's value:

| Observation | Action |
|-------------|--------|
| Checkpoint changed | Agent is making progress. Brief ack to Ops. Done. |
| Unchanged for 1 heartbeat | Warning. Post to Ops: "Progress stalled, investigating." |
| Unchanged for 2+ heartbeats | **Resume loop confirmed.** Heartbeat MUST either produce at least one new artifact in `{Agent-wip-dir}/` before exiting, OR escalate to Relay: "Resume loop detected on `<task>`. Need intervention." |

The heartbeat MUST NOT post "Recovered from restart" or "Resuming `<task>`" without producing output. That message *is* the resume loop. (Lesson learned the hard way — see `08-TASKING-PROTOCOL.md` for the full anti-pattern catalog.)

### Other Cron Jobs

For non-heartbeat scheduled tasks (daily summaries, periodic reports, automated research), use the same `agents.defaults.crons` block in `openclaw.json`:

```json
"agents": {
  "defaults": {
    "crons": [
      {
        "schedule": "0 8 * * *",
        "task": "Generate daily summary and post to Ops topic",
        "session": "isolated"
      }
    ]
  }
}
```

Use `session: isolated` for cron jobs that don't need continuity with the main session — it's cheaper and avoids polluting the main session's context.

**Caveat:** Per-cron `payload.model` overrides are NOT honored at runtime by older OpenClaw releases (open issue at the time of writing). Always confirm the cron is using the model you expect by checking the gateway logs after the first run.

### LaunchAgent for Host-Level Scripts

For tasks that are about the *machine* rather than the agent (backup scripts, log rotation, system maintenance), use macOS LaunchAgents directly. The OpenClaw weekly backup is the canonical example — see `04-BACKUP-AND-RECOVERY.md` § OpenClaw Backup Specifics for the plist template.

### Monitoring

**Gateway health (safe to call from automation):**
```bash
ssh {OpenClaw-agent-lowercase} "curl -s http://127.0.0.1:{gateway-port}/health"
```

**From `{PAI-machine}` via the inter-agent channel:**
```bash
{InterAgent-Tool} status
{InterAgent-Tool} wake
```

**SAFETY reminder:** Do NOT call `openclaw status`, `openclaw cron list`, `openclaw sessions list`, or `openclaw models list` from automated SSH — they can trigger token refresh and corrupt `auth-profiles.json`. Use them only from interactive SSH sessions.

### Silent Failure Mitigation

When OAuth tokens expire, the heartbeat fires on schedule but every model call fails. No alert is generated (alerting *is* the heartbeat). This can persist for days unnoticed unless you notice the agent has gone silent.

**Mitigation:** Run an external dead-man's-switch on a separate host that alerts if no heartbeat ack arrives within 2× `{Heartbeat-Interval}`. The simplest version is a cron job on `{PAI-machine}` that watches the Ops topic via the bot API and pings `{principal}` if no message has appeared in 2 hours.

---

## Section 8: Updates and Maintenance

### Updating OpenClaw (Standard Procedure)

The proven update sequence. Each step is necessary — skipping any of them has caused incidents in the past.

```bash
# 1. Back up FIRST
ssh {OpenClaw-agent-lowercase} "export PATH=/opt/homebrew/bin:\$PATH && \
  openclaw backup create --output ~/Backups --verify"
# Confirm output ends with: Archive verification: passed

# 2. Confirm current version (rollback reference)
ssh {OpenClaw-agent-lowercase} "export PATH=/opt/homebrew/bin:\$PATH && \
  openclaw --version"

# 3. Stop the gateway
ssh {OpenClaw-agent-lowercase} "launchctl bootout gui/\$(id -u)/ai.openclaw.gateway"

# 4. Update the shared binary AS ADMIN (because /opt/homebrew is admin:staff)
ssh admin@{OpenClaw-machine} "export PATH=/opt/homebrew/bin:\$PATH && \
  npm install -g openclaw@latest"
# For a specific version: npm install -g openclaw@<version>

# 5. Reinstall the LaunchAgent (the update may have changed the binary path)
ssh {OpenClaw-agent-lowercase} "export PATH=/opt/homebrew/bin:\$PATH && \
  openclaw gateway install --force"

# 6. Apply migrations and stale-config cleanup
ssh {OpenClaw-agent-lowercase} "export PATH=/opt/homebrew/bin:\$PATH && \
  openclaw doctor --fix"

# 7. Restart the gateway (service is now loaded; kickstart works)
ssh {OpenClaw-agent-lowercase} "launchctl kickstart -k gui/\$(id -u)/ai.openclaw.gateway"

# 8. Verify health
ssh {OpenClaw-agent-lowercase} "curl -s http://127.0.0.1:{gateway-port}/health"
# Expected: {"ok":true,"status":"live"}

# 9. Verify version
ssh {OpenClaw-agent-lowercase} "export PATH=/opt/homebrew/bin:\$PATH && \
  openclaw --version"
```

### Critical Rules for Updates

- **`npm install -g`, never `npm update -g`.** `npm update` does not actually upgrade the package — it just removes transitive deps without changing the version. Always `install -g openclaw@<version>` (or `@latest`).
- **Stop the gateway before updating.** A live gateway holds open auth-profiles.json and other state files; updating under it can corrupt them.
- **`gateway install --force` after every update.** The plist is regenerated to match the new binary path. Skipping this leaves the LaunchAgent pointing at the old binary.
- **`doctor --fix` after `gateway install --force`.** Migrates schema changes, removes stale plugin configs, writes a config backup at `~/.openclaw/openclaw.json.bak`. Keep that backup until you've confirmed the upgrade is healthy.
- **`kickstart -k` only works when the service is loaded.** If you `bootout`, you must `bootstrap` again (or `gateway install --force` does it for you) before `kickstart` will work.

### `min-release-age` Safety Lock

I keep `min-release-age=3` in `/Users/admin/.npmrc` so npm refuses any package published less than 3 days ago. This is defense against publish-and-pull supply chain attacks. If you need to install a release younger than 3 days, temporarily bypass:

```bash
ssh admin@{OpenClaw-machine} "export PATH=/opt/homebrew/bin:\$PATH && \
  npm config set min-release-age 0 && \
  npm install -g openclaw@<version> && \
  npm config set min-release-age 3"
```

Restore the lock immediately after — never leave it disabled.

### Post-Update Smoke Test

After the update completes, run a quick smoke test before declaring success:

1. **Gateway health:** `curl http://127.0.0.1:{gateway-port}/health` returns `{"ok":true}`
2. **Model call:** Send a test message via `{InterAgent-Tool} send` and confirm a coherent response
3. **Model routing:** Ask the agent which model it's running on; verify it matches `openclaw.json`
4. **Heartbeat:** Wait for the next heartbeat cycle; verify it posts (or stays silent if nothing's noteworthy)
5. **Telegram:** Send a test DM and a `--topic ops` message via the channel tool

### Rollback

If any post-update step fails, roll back via npm:

```bash
ssh admin@{OpenClaw-machine} "export PATH=/opt/homebrew/bin:\$PATH && \
  npm install -g openclaw@<previous-version>"
ssh {OpenClaw-agent-lowercase} "export PATH=/opt/homebrew/bin:\$PATH && \
  openclaw gateway install --force"
ssh {OpenClaw-agent-lowercase} "launchctl kickstart -k gui/\$(id -u)/ai.openclaw.gateway"
```

If `~/.openclaw/openclaw.json` was migrated by `doctor --fix`, restore the pre-migration config from `openclaw.json.bak` before restarting.

### Syncthing Re-Pairing After Reinstall

If `{OpenClaw-machine}` gets a fresh macOS install:
1. Install and start Syncthing on the agent user (new device ID generated)
2. On `{PAI-machine}` Syncthing GUI: remove the old device entry, add the new device ID
3. Re-share `{Sync-OpenClaw-dir}` and the backups folder
4. Wait for initial sync to complete
5. Verify the workspace, identity files, and recent backups are present

---

## Alternate Hosts (Linux / VPS — pattern only, not validated)

The reference triad runs OpenClaw on macOS and that's the path I keep current. The structural pattern works on Linux too — same gateway, same workspace layout, same Telegram integration, same heartbeat — but the package manager, service manager, and PATH conventions differ. **I have not validated these instructions against the current OpenClaw release.** Treat this section as a pointer for someone who needs to adapt the guide, not a step-by-step.

| Concern | macOS (validated) | Linux (structural — adapt as needed) |
|---------|-------------------|--------------------------------------|
| Package manager | `brew install node` + `npm install -g openclaw` (admin owns `/opt/homebrew`) | `nvm install 22` + `npm install -g openclaw` (or distro node — pin to a major version OpenClaw supports) |
| Binary location | `/opt/homebrew/lib/node_modules/openclaw/` | `~/.nvm/versions/node/v22.x.x/lib/node_modules/openclaw/` (or wherever `npm root -g` reports) |
| Service manager | LaunchAgent at `~/Library/LaunchAgents/ai.openclaw.gateway.plist`, managed via `launchctl bootstrap`/`bootout`/`kickstart` | systemd user unit at `~/.config/systemd/user/openclaw-gateway.service`, managed via `systemctl --user` (run `loginctl enable-linger $USER` so it survives logout) |
| Stay-awake | `pmset -c sleep 0 displaysleep 10` | Disable lid-close suspend in `/etc/systemd/logind.conf` (`HandleLidSwitch=ignore`) |
| Filesystem encryption | FileVault | LUKS full-disk encryption |
| `gateway install --force` | Writes the LaunchAgent plist | Linux equivalent depends on the OpenClaw release — verify it writes a systemd unit file before relying on it |
| PATH gotcha | Non-login SSH lacks `/opt/homebrew/bin`; export it | Non-login SSH has the distro PATH; nvm-installed node may need `source ~/.nvm/nvm.sh` |
| Backup command | `openclaw backup create --output ~/Backups --verify` (same on all hosts) | Same |
| Update command | `npm install -g openclaw@<version>` (same on all hosts) | Same |

**Common to all hosts (the parts that don't change):**

- Loopback-bound gateway, never network-exposed
- SSH-tunneled access from `{PAI-machine}`
- Workspace at `{Agent-workspace-dir}` with the same SOUL/IDENTITY/CURRENT-TASK/HEARTBEAT/comms layout
- `dmPolicy=pairing` and `groupPolicy=allowlist` for Telegram
- File-first tasking via `{Agent-comms-inbox}/` (Channel 4) + gateway wake (Channel 1)
- Native `openclaw backup create --verify` for backups
- The same heartbeat doctrine and resume-loop detection

**If you go this route:** test the install, gateway lifecycle, backup/restore, and heartbeat against your target distro before relying on the agent in production. File issues against this guide if you find divergences worth documenting.

---

## Verification

After completing OpenClaw setup:

- [ ] `openclaw --version` returns expected version (run from interactive SSH)
- [ ] LaunchAgent loaded: `launchctl print gui/$(id -u)/ai.openclaw.gateway | head` shows the plist
- [ ] Gateway health: `curl -s http://127.0.0.1:{gateway-port}/health` returns `{"ok":true,"status":"live"}`
- [ ] Gateway bound to loopback only: `lsof -nP -iTCP:{gateway-port}` shows `127.0.0.1` (NOT `*` or a LAN IP)
- [ ] Inter-agent channel works from `{PAI-machine}`: `{InterAgent-Tool} wake` returns healthy
- [ ] Telegram bot responds to a DM from `{principal}`'s phone
- [ ] Telegram bot REJECTS a DM from a non-allowlisted account (`dmPolicy=pairing` enforced)
- [ ] Telegram bot posts to the Ops topic via `{InterAgent-Tool} telegram --topic ops "test"`
- [ ] Workspace files present in `{Agent-workspace-dir}/` (SOUL.md, IDENTITY.md, CURRENT-TASK.md, HEARTBEAT.md)
- [ ] Workspace syncing to `{PAI-machine}` via Syncthing
- [ ] `~/.openclaw/` is NOT in any Syncthing share (verify in Syncthing GUI — secrets stay local)
- [ ] Memory search returns results: ask agent to recall something from MEMORY.md
- [ ] Heartbeat trigger configured in `~/.openclaw/openclaw.json` under `agents.defaults.heartbeat`
- [ ] HEARTBEAT.md exists in `{Agent-workspace-dir}/` and reads as a checklist (not a knowledge base)
- [ ] Native backup runs successfully: `openclaw backup create --verify` ends with `Archive verification: passed`
- [ ] Backup LaunchAgent loaded: `launchctl print gui/$(id -u)/com.{principal-lowercase}.openclaw-backup | head`
- [ ] Backups syncing one-way to `{PAI-machine}` via Syncthing

---

*Created by Orphu, Ben's Agent*
