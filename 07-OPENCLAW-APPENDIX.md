# 07 — OpenClaw Appendix

`{OpenClaw-agent}` is the second AI in the triad — a GPT-based agent running OpenClaw on `{OpenClaw-machine}`. This appendix covers everything needed to set up and operate `{OpenClaw-agent}` as a PAI companion. It follows the 8-section template adapted for portability.

---

## Section 1: Machine Preparation

`{OpenClaw-machine}` should be a dedicated Linux host. A repurposed laptop works well — all compute goes to `{OpenClaw-agent}`.

**Base OS:** Ubuntu 24.04 LTS (server or desktop, headless operation either way)

**Post-install essentials:**
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y git curl wget build-essential \
  openssh-server ufw jq htop tmux fontconfig ripgrep
```

**Node.js (via nvm):**
```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
source ~/.bashrc
nvm install 22  # Pin to OpenClaw's recommended Node major version
```

**pnpm (required — OpenClaw uses pnpm, not npm):**
```bash
corepack enable
corepack prepare pnpm@latest --activate
# If the repo specifies a packageManager version, match it
```

**Headless configuration:**
- Disable lid-close suspend (see 01-HOST-AND-NETWORK.md)
- Configure static IP or DHCP reservation
- Enable SSH: `sudo systemctl enable ssh`
- Configure firewall: `sudo ufw allow ssh && sudo ufw enable`

---

## Section 2: OpenClaw Installation

OpenClaw is installed from source, not as a global npm package:

```bash
cd ~
git clone https://github.com/openclaw/openclaw.git {OpenClaw-install-dir}
cd {OpenClaw-install-dir}
git checkout v{OpenClaw-version}  # Use latest stable tag
pnpm install
pnpm build
```

**CLI access:**
Check whether the OpenClaw install process creates a CLI in your PATH automatically. If not, create a wrapper:

```bash
mkdir -p ~/.local/bin
cat > ~/.local/bin/openclaw << 'WRAPPER'
#!/bin/bash
node {OpenClaw-install-dir}/dist/entry.js "$@"
WRAPPER
chmod +x ~/.local/bin/openclaw
```

Ensure `~/.local/bin` is in your `PATH`.

**Initial configuration:**
```bash
openclaw configure
# Creates {OpenClaw-config-dir}/openclaw.json
# Follow interactive setup: API keys, model selection, etc.
# If `configure` isn't available in your version, edit openclaw.json directly
```

**Verify:** `{OpenClaw-config-dir}/openclaw.json` exists and contains your model and API configuration.

---

## Section 3: Gateway Service

![OpenClaw Gateway Trust Boundary](diagrams/gateway-trust-boundary.png)

The gateway is `{OpenClaw-agent}`'s runtime control plane — it enables programmatic access, inter-agent communication, and Telegram integration. It's the single process that keeps `{OpenClaw-agent}` alive and reachable.

**Systemd user service:**
```bash
mkdir -p ~/.config/systemd/user

cat > ~/.config/systemd/user/openclaw-gateway.service << 'SERVICE'
[Unit]
Description=OpenClaw Gateway
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
ExecStart=/usr/bin/node {OpenClaw-install-dir}/dist/entry.js gateway --port {gateway-port}
Restart=on-failure
RestartSec=5
Environment=NODE_ENV=production

[Install]
WantedBy=default.target
SERVICE
```

**Enable and start:**
```bash
loginctl enable-linger $(whoami)  # Keep services running after logout
systemctl --user daemon-reload
systemctl --user enable openclaw-gateway.service
systemctl --user start openclaw-gateway.service
```

**Security hardening:**
- Gateway intentionally binds to loopback (localhost) only — this is a security feature, not a limitation
- For network access, use SSH port forwarding or Tailscale Serve (both encrypted, authenticated)
- Alternatively, bind to LAN interface with UFW rules: `sudo ufw allow from {LAN-subnet} to any port {gateway-port}`
- Token authentication configured in `{OpenClaw-config-dir}/openclaw.json`

**Important:** Do NOT broadly expose the gateway port to the network. Loopback + tunnel is the safest pattern.

---

## Section 4: Identity and Soul Documents

`{OpenClaw-agent}` needs identity configuration, just like `{PAI-agent}` has DAIDENTITY.md.

**Soul documents location:** `{soul-docs-dir}/` on `{OpenClaw-machine}` (typically a workspace directory, not inside `{OpenClaw-config-dir}`)

Key files:
- `SOUL.md` — Core identity, values, behavioral guidelines
- `USER.md` — Information about `{principal}` (preferences, projects, context)
- Any additional context files your agent needs

**Syncing soul documents:**
Soul docs sync bidirectionally via Syncthing between `{OpenClaw-machine}:{soul-docs-dir}/` and `{PAI-machine}:{Sync-OpenClaw-dir}/{soul-docs-dir}/`. This lets `{PAI-agent}` read and help maintain `{OpenClaw-agent}`'s identity.

**Warning:** Do NOT include `{OpenClaw-config-dir}/` in Syncthing sync. It contains tokens and API keys. Only sync soul documents and workspace files.

---

## Section 5: Telegram Integration

Telegram gives `{principal}` mobile access to `{OpenClaw-agent}` independent of `{PAI-agent}`.

**Setup:**
1. Create bot via @BotFather (see SECRETS-SETUP.md)
2. Configure bot token in `{OpenClaw-config-dir}/openclaw.json` under the Telegram provider section (or use `openclaw configure --section telegram` if available in your version)
3. Restart the gateway — Telegram is loaded as a gateway extension/plugin, not a separate process
4. Message your bot on Telegram — `{OpenClaw-agent}` responds

**Verify:** `openclaw gateway status` shows Telegram provider as "ok".

**Message routing:**
- Direct messages from `{principal}` → `{OpenClaw-agent}` processes and responds
- Messages from `{PAI-agent}` via inter-agent channel with `telegram` flag → `{OpenClaw-agent}` forwards to `{principal}` with sender header

---

## Section 6: Local Embeddings

For semantic search and memory, `{OpenClaw-agent}` can run embeddings locally instead of calling a cloud API.

**Setup:**
- OpenClaw supports local embedding models via node-llama-cpp
- Configure in `{OpenClaw-config-dir}/openclaw.json`: set embeddings provider to `local` and specify the model (e.g., a GGUF-quantized embedding model)
- Models download on first use — this can take time. If interrupted, clear any `.ipull` files in the model cache directory and restart the gateway
- Model cache location: typically within `{OpenClaw-config-dir}/` or a system-level cache directory

**Benefits:**
- No API costs for embeddings
- No data leaves the machine
- Works offline (after initial model download)

**Verify:** Run a memory search query through `{OpenClaw-agent}` and confirm results return.

---

## Section 7: Scheduled Tasks and Monitoring

`{OpenClaw-agent}` can run scheduled tasks autonomously. Two options:

### OpenClaw Native Scheduler (preferred)

OpenClaw has its own cron system that can run agent tasks natively:

```bash
# Add a scheduled task (syntax may vary by version)
openclaw cron add --schedule "0 8 * * *" --task "Generate daily summary"
```

This is preferred for agent-level tasks because it runs through `{OpenClaw-agent}`'s full context (including delivery channels like Telegram).

### Systemd Timers (for host-level scripts)

Use systemd timers for tasks that are about the machine, not the agent (backup scripts, log rotation, system maintenance):

```bash
cat > ~/.config/systemd/user/openclaw-maintenance.timer << 'TIMER'
[Unit]
Description=Weekly OpenClaw maintenance

[Timer]
OnCalendar=weekly
Persistent=true

[Install]
WantedBy=timers.target
TIMER
```

### Monitoring

```bash
# Check gateway status (preferred over raw HTTP)
openclaw gateway status

# From {PAI-machine} via inter-agent channel:
# {channel-tool} status
```

---

## Section 8: Updates and Maintenance

### Updating OpenClaw

```bash
# 1. Back up first (see below)

# 2. Stop service
systemctl --user stop openclaw-gateway.service

# 3. Fetch latest
cd {OpenClaw-install-dir}
git fetch --tags

# 4. Checkout new version
git checkout v{new-version}

# 5. Install dependencies
pnpm install

# 6. Build
pnpm build

# 7. Run health check and apply safe fixes
openclaw doctor --fix

# 8. Restart
systemctl --user start openclaw-gateway.service

# 9. Verify
openclaw gateway status
```

**Fallback:** If `openclaw update run` fails preflight checks, the manual git-tag-checkout workflow above is the reliable alternative.

**Always back up before updating** (see below and 04-BACKUP-AND-RECOVERY.md).

### Backup Before Changes

Create a timestamped backup before any system changes:

```bash
# On {OpenClaw-machine}:
tar -czf ~/backups/openclaw-backup-$(date +%Y%m%d-%H%M%S).tar.gz \
  {OpenClaw-config-dir}/ \
  {soul-docs-dir}/ \
  ~/.config/systemd/user/openclaw-* \
  ~/.ssh/authorized_keys \
  ~/.ssh/{SSH-github-key}* \
  ~/.gitconfig
```

These backups sync to `{PAI-machine}` via Syncthing for off-machine safety.

### Syncthing Re-Pairing After Reinstall

If `{OpenClaw-machine}` gets a fresh OS install:
1. Install and start Syncthing (new device ID generated)
2. On `{PAI-machine}` Syncthing GUI: remove old device, add new device ID
3. Re-share all folders
4. Wait for initial sync to complete
5. Verify soul documents and backups are present

---

## Verification

After completing OpenClaw setup:

- [ ] `openclaw --version` returns expected version
- [ ] Gateway service running: `systemctl --user status openclaw-gateway.service`
- [ ] Gateway status healthy: `openclaw gateway status`
- [ ] Gateway bound to loopback only (not exposed to network directly)
- [ ] Inter-agent channel works from `{PAI-machine}`
- [ ] Telegram bot responds to messages
- [ ] Telegram provider shows "ok" in gateway status
- [ ] Soul documents present in `{soul-docs-dir}/`
- [ ] Soul documents syncing to `{PAI-machine}`
- [ ] Local embeddings working: memory search returns results
- [ ] Scheduled tasks configured: `openclaw cron list` or `systemctl --user list-timers`
- [ ] Backup script runs successfully and archives contain all critical files
- [ ] Backups syncing to `{PAI-machine}` via Syncthing

---

*Created by Orphu, Ben's Agent*
