# Template Variables Reference

All files in this guide use `{Variable-Name}` syntax. Before following any guide, define every variable below for your environment. Your receiving PAI agent can read this file and substitute values automatically.

---

## How to Use

1. Copy this file
2. Replace each variable's value with your own
3. Your PAI agent reads this file and substitutes throughout the guide

Variables use `{Title-Case-Hyphenated}` format. If you see one in any guide file that isn't defined here, something is wrong — file an issue.

---

## Machine Variables

The reference triad runs every host on macOS. Other hosts (Linux, VPS) work structurally — see the "Alternate hosts" sidebar in `07-OPENCLAW-APPENDIX.md` — but are not validated against current OpenClaw releases.

| Variable | Purpose | Example |
|----------|---------|---------|
| `{PAI-machine}` | Primary workstation hostname running PAI/Claude Code (macOS) | (your hostname) |
| `{OpenClaw-machine}` | Secondary machine hosting the OpenClaw agent (macOS — Mac mini works well) | (your hostname) |
| `{services-machine}` | Docker services host (reverse proxy, file sharing, fleet dashboard) | (your hostname) |
| `{worker-machine}` | Parallel delegation worker for compute-heavy tasks | (your hostname) |
| `{hypervisor-machine}` | Proxmox hypervisor (only if you virtualize `{services-machine}`) | (your hostname) |

---

## Agent Variables

| Variable | Purpose | Example |
|----------|---------|---------|
| `{PAI-agent}` | Name of your PAI/Claude Code DA | <!-- Verbum --> |
| `{OpenClaw-agent}` | Name of your OpenClaw/GPT agent | <!-- Orphu --> |
| `{principal}` | Your name (the human user) | <!-- Ben --> |

---

## Network Variables

**Never hardcode IPs.** Always use these variables so the guide works across different network topologies.

| Variable | Purpose | Example Format |
|----------|---------|----------------|
| `{PAI-LAN-IP}` | PAI machine LAN address | `10.x.x.x` |
| `{OpenClaw-LAN-IP}` | OpenClaw machine LAN address | `10.x.x.x` |
| `{OpenClaw-TS-IP}` | OpenClaw Tailscale address | `100.x.x.x` |
| `{services-LAN-IP}` | Services machine LAN address | `10.x.x.x` |
| `{worker-LAN-IP}` | Worker machine LAN address | `10.x.x.x` |
| `{hypervisor-LAN-IP}` | Hypervisor LAN address | `10.x.x.x` |
| `{LAN-subnet}` | Network subnet (CIDR) | `10.x.x.0/24` |
| `{gateway-port}` | OpenClaw gateway port | `18789` |

---

## Agent Identity Variables

| Variable | Purpose | Notes |
|----------|---------|-------|
| `{DA-Name}` | DA short name (used in config) | Same as `{PAI-agent}` |
| `{DA-Full-Name}` | DA full name for display | e.g., "Verbum - Personal AI" |
| `{DA-Display-Name}` | DA display name | Same as `{DA-Name}` typically |
| `{DA-Color}` | DA display color | Hex code like `#3B82F6` |
| `{Voice-Id}` | ElevenLabs voice ID for DA | Get from ElevenLabs dashboard |
| `{Voice-Stability}` | Voice stability parameter | Float 0.0–1.0 |
| `{Voice-Similarity-Boost}` | Voice similarity boost | Float 0.0–1.0 |
| `{Voice-Style}` | Voice style parameter | Float 0.0–1.0 |
| `{Voice-Server-Port}` | Voice notification server port | `8888` |
| `{Startup-Catchphrase}` | DA startup greeting | Short spoken phrase |
| `{Principal-Timezone}` | Your timezone | e.g., `America/Chicago` |

---

## Credential Variables

**Values NEVER appear in documentation.** These variables represent secrets that must be generated fresh per installation. See `SECRETS-SETUP.md` for generation commands.

| Variable | Purpose | Generation |
|----------|---------|------------|
| `{gateway-token}` | OpenClaw gateway authentication token | `openssl rand -hex 24` |
| `{telegram-bot}` | Telegram bot username for mobile access | Create via BotFather |
| `{telegram-bot-token}` | Telegram bot API token | From BotFather on creation |
| `{Inter-Agent-Group-ID}` | Telegram group ID for triad coordination (Relay/Ops/Alerts topics) | Created when you make the group; treat as private |
| `{Telegram-Topic-Relay}` | Topic ID for agent-to-principal escalations (sacred channel) | Assigned when topic is created |
| `{Telegram-Topic-Ops}` | Topic ID for routine status, heartbeat acks | Assigned when topic is created |
| `{Telegram-Topic-Alerts}` | Topic ID for failures, backup results, health events | Assigned when topic is created |
| `{Principal-Telegram-User-ID}` | Telegram user ID for the human principal (used in DM allowlist) | Look up via @userinfobot; treat as private |
| `{Syncthing-device-ID-PAI}` | Syncthing device ID for PAI machine | Auto-generated on install |
| `{Syncthing-device-ID-OpenClaw}` | Syncthing device ID for OpenClaw machine | Auto-generated on install |
| `{Syncthing-device-ID-services}` | Syncthing device ID for services machine | Auto-generated on install |
| `{Syncthing-device-ID-worker}` | Syncthing device ID for worker machine | Auto-generated on install |
| `{SSH-automation-key}` | Filename for passphrase-less automation SSH key | `ssh-keygen -t ed25519` |
| `{SSH-github-key}` | Filename for GitHub SSH key | `ssh-keygen -t ed25519` |

---

## Tech Stack Variables

| Variable | Purpose | Example |
|----------|---------|---------|
| `{Preferred-Browser}` | Default browser | `arc`, `chrome`, `firefox` |
| `{Preferred-Terminal}` | Terminal application | `terminal`, `ghostty`, `kitty` |
| `{Package-Manager}` | JS package manager | `bun`, `npm`, `pnpm` |
| `{Preferred-Language}` | Primary dev language | `TypeScript`, `Python` |

---

## Service Variables

| Variable | Purpose | Example |
|----------|---------|---------|
| | Service-specific port variables depend on your deployment. Define them as needed for your Docker stack. | |
| `{PAI-dir}` | PAI home directory | `~/.claude` |
| `{PAI-Dir}` | PAI home directory (alternate casing) | `~/.claude` |
| `{PAI-dir-basename}` | PAI directory basename only | `.claude` |
| `{PAI-framework-dir}` | PAI framework source | `~/PAI` |
| `{Projects-Dir}` | Projects directory | `~/Projects` |

---

## Path Variables

| Variable | Purpose | Example |
|----------|---------|---------|
| `{PAI-backups-dir}` | Local backup snapshot directory | `~/PAI-Backups` |
| `{Sync-OpenClaw-dir}` | Syncthing folder for OpenClaw file exchange | `~/PAI-{openclaw-suffix}` |
| `{Sync-worker-dir}` | Syncthing folder for worker job exchange | `~/PAI-{worker-suffix}` |
| `{Sync-services-dir}` | Syncthing folder for services config exchange | `~/PAI-{services-suffix}` |
| `{OpenClaw-install-dir}` | OpenClaw installation path on OpenClaw machine | `/opt/homebrew/lib/node_modules/openclaw` (macOS, npm global) |
| `{OpenClaw-config-dir}` | OpenClaw configuration directory | `~/.openclaw` |
| `{Agent-workspace-dir}` | OpenClaw agent runtime workspace (CURRENT-TASK, comms, wip, drafts) | `~/agent-workspace` |
| `{Agent-comms-inbox}` | File-based brief inbox inside the agent workspace | `{Agent-workspace-dir}/comms/inbox` |
| `{Agent-comms-archive}` | Processed-brief archive inside the agent workspace | `{Agent-workspace-dir}/comms/archive` |
| `{Agent-wip-dir}` | Active work-in-progress artifacts | `{Agent-workspace-dir}/wip` |
| `{soul-docs-dir}` | OpenClaw agent soul/identity documents (typically inside the workspace) | `{Agent-workspace-dir}` |

---

## GitHub Variables

| Variable | Purpose | Notes |
|----------|---------|-------|
| `{github-username}` | GitHub account for shared repos | Used for agent collaboration |
| `{github-org}` | GitHub org (if applicable) | Optional |

---

## PAI Version Variables

| Variable | Purpose | Current |
|----------|---------|---------|
| `{PAI-version}` | PAI framework version | `4.0.3` |
| `{Algorithm-version}` | Algorithm version | `v3.5.0` |
| `{PAI-repo}` | PAI source repository | `github.com/danielmiessler/PAI` |
| `{OpenClaw-version}` | OpenClaw version tag | Date-based, e.g. `2026.4.x`. Don't pin in docs — describe as "current stable at install time". |

---

## Security Variables (used in 03-SECURITY-MODEL.md)

| Variable | Purpose | Notes |
|----------|---------|-------|
| `{Ask-Pattern-Force-Push}` | Ask pattern for force push | e.g., `Bash(git push --force:*)` |
| `{Ask-Pattern-Credential-Read}` | Ask pattern for credential reads | e.g., `Read(~/.ssh/id_*)` |
| `{Ask-Pattern-Settings-Modify}` | Ask pattern for settings changes | e.g., `Write(~/.claude/settings.json)` |
| `{Ask-Pattern-System-Config}` | Ask pattern for system config changes | Custom per environment |
| `{Rotation-Schedule}` | Credential rotation frequency | `quarterly` |

---

## Miscellaneous Variables

| Variable | Purpose | Notes |
|----------|---------|-------|
| `{OpenClaw-suffix}` | Suffix for OpenClaw Syncthing folder | Used in folder naming |
| `{OpenClaw-agent-lowercase}` | Lowercase form of OpenClaw agent name | For filenames, scripts |
| `{Research-Engine-Count}` | Number of parallel research engines | Typically 3+ |
| `{principal-lowercase}` | Lowercase form of principal's name | For usernames, SSH |
| `{worker-username}` | Username on worker machine | May differ from principal |
| `{ethernet-interface}` | Network interface name on OpenClaw machine | e.g., `enx...`, `eth0` |
| `{gateway-IP}` | Default gateway/router IP | e.g., `10.x.x.1` |
| `{worker-suffix}` | Suffix for worker Syncthing folder | Used in folder naming |
| `{services-suffix}` | Suffix for services Syncthing folder | Used in folder naming |
| `{openclaw-suffix}` | Lowercase suffix for OpenClaw folder | Used in path examples |
| `{new-version}` | Target version for OpenClaw updates | Used in update docs |
| `{InterAgent-Tool}` | Inter-agent channel CLI tool (the wrapper that does SSH→loopback WebSocket) | e.g., `~/.claude/skills/{OpenClaw-agent}Channel/Tools/{OpenClaw-agent-lowercase}-channel.sh` |
| `{Heartbeat-Interval}` | Heartbeat cron interval for `{OpenClaw-agent}` | e.g., `60m` |
| `{principal-uppercase}` | Uppercase form of `{principal}` (used in relay format `[{principal-uppercase} ONLY]` opt-out marker) | e.g., `BEN` |
| `{services-backups-dir}` | Backup landing directory on services machine | May differ from `{PAI-backups-dir}` |

---

## Illustrative Placeholders

These appear in **example message headers and code samples** only — they're not configuration values you fill in. They represent dynamic runtime content.

| Placeholder | Context | Notes |
|-------------|---------|-------|
| `{SENDER-NAME}` | Message header: sender | Uppercase form of agent name at runtime |
| `{RECIPIENT-NAME}` | Message header: recipient | Uppercase form of agent name at runtime |
| `{PAI-AGENT-NAME}` | Message header: PAI agent | Uppercase of `{PAI-agent}` |
| `{OPENCLAW-AGENT-NAME}` | Message header: OpenClaw agent | Uppercase of `{OpenClaw-agent}` |
| `{Spoken-Message}` | Voice notification payload | Dynamic per notification |
| `{Third-Party-Name}` | Pronoun rule example | Used in identity docs |
| `{Skill-Name}` | Skill reference in examples | Dynamic per context |
| `{Principal-Name}` | Alternate reference to `{principal}` | Used in identity/config examples |

---

*Created by Verbum, Ben's DA*
