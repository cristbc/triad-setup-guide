# Implementation Checklist

A step-by-step guide for a receiving PAI agent (or human) to stand up the full triad. Each phase has a verification gate — don't proceed until the gate passes. The checklist has 8 phases plus a prerequisite phase.

**Note:** This checklist covers the minimum viable triad (2 agents). To add more agents, repeat Phases 5-7 for each additional agent — each new agent needs its own communication channel skill, workspace layout, identity configuration, and tasking protocol setup. The reference triad runs every host on macOS; if you're targeting a Linux/VPS host see the "Alternate hosts" sidebar in `07-OPENCLAW-APPENDIX.md`.

---

## Phase 0: Prerequisites

**Read:** `INDEX.md`, `VARIABLES.md`, `SECRETS-SETUP.md`

- [ ] Read and understand the guide structure (`INDEX.md`)
- [ ] Copy `VARIABLES.md` and fill in every variable for your environment
- [ ] Generate all credentials per `SECRETS-SETUP.md`
- [ ] Store all credentials in password manager
- [ ] Verify: Every variable in `VARIABLES.md` has a value assigned

**Gate:** All variables defined, all credentials generated and stored.

---

## Phase 1: Machines and Network

**Read:** `01-HOST-AND-NETWORK.md`

- [ ] Identify or provision all machines (minimum: `{PAI-machine}` + `{OpenClaw-machine}`)
- [ ] Connect all machines to LAN (Ethernet preferred)
- [ ] Assign or reserve IP addresses for each machine
- [ ] Install Tailscale on all machines, authenticate to same account
- [ ] Enable Tailscale SSH on machines needing remote access
- [ ] Generate and distribute SSH keys per `SECRETS-SETUP.md`
- [ ] Configure `~/.ssh/config` on `{PAI-machine}` with host aliases
- [ ] Restrict automation SSH key to `{LAN-subnet}` in authorized_keys
- [ ] Install Syncthing on all machines
- [ ] Create shared folders per hub-and-spoke topology
- [ ] Pair all Syncthing devices through `{PAI-machine}` hub
- [ ] Configure `{OpenClaw-machine}` for headless operation (lid-close, network, encryption)

**Gate:** `ssh {OpenClaw-machine}` works from `{PAI-machine}`. Syncthing test file appears on all paired machines. Tailscale SSH works from outside LAN.

---

## Phase 2: PAI Configuration

**Read:** `02-PAI-CUSTOMIZATION-LAYER.md`

- [ ] Install Claude Code on `{PAI-machine}`
- [ ] Install PAI framework from `{PAI-repo}`
- [ ] Configure `settings.json`:
  - [ ] `daidentity` block (name, voice, catchphrase)
  - [ ] `principal` block (name, timezone)
  - [ ] `env` paths (`PAI_DIR`, `PROJECTS_DIR`)
  - [ ] `permissions` (allow, deny, ask lists)
  - [ ] `contextFiles` (SKILL.md, steering rules, identity)
- [ ] Set up hook system:
  - [ ] SecurityValidator on PreToolUse (Bash, Edit, Write, Read)
  - [ ] FormatReminder on UserPromptSubmit
  - [ ] Rating capture hooks
  - [ ] Session lifecycle hooks (start, end, stop)
- [ ] Configure voice server:
  - [ ] Install and start your voice server (whatever implementation you've chosen)
  - [ ] Verify it responds using its own documented test command — not a hard-coded endpoint, since voice server interfaces vary by implementation
- [ ] Create DAIDENTITY.md with your DA's personality and voice conventions
- [ ] Create or customize AI Steering Rules (SYSTEM + USER)
- [ ] Set up memory directories (WORK/, LEARNING/, STATE/, RESEARCH/)
- [ ] Verify skills load correctly (start a session, check context injection)

**Gate:** New Claude Code session starts, context loads, voice greeting plays, SecurityValidator blocks a test dangerous command, FormatReminder enforces Algorithm format.

---

## Phase 3: Security

**Read:** `03-SECURITY-MODEL.md`

- [ ] Review stock security patterns from PAI repo
- [ ] Copy patterns to USER configuration and customize for your threat model
- [ ] Verify BLOCK level: test with a destructive command (should be blocked)
- [ ] Verify CONFIRM level: test with force push (should prompt for approval)
- [ ] Verify ALERT level: test with an unusual but allowed operation
- [ ] Confirm security event logging: check `MEMORY/SECURITY/` for log entries
- [ ] Review `settings.json` ask patterns — add any that concern you
- [ ] Verify no credentials in git: `git status` shows no secret files tracked

**Gate:** Security validator catches test cases at all three levels. Event logs are being written. No credentials exposed.

---

## Phase 4: Backup and Recovery

**Read:** `04-BACKUP-AND-RECOVERY.md`

- [ ] Create backup script for daily PAI snapshots
- [ ] Set up LaunchAgent (macOS) or systemd timer for automated daily backups
- [ ] Verify daily snapshot creates correctly: `ls {PAI-backups-dir}/`
- [ ] Confirm Syncthing mirrors backups to `{services-machine}` (if applicable)
- [ ] Set up full system backup (Time Machine or equivalent)
- [ ] Create OpenClaw backup script on `{OpenClaw-machine}`
- [ ] Verify OpenClaw backup creates and syncs to `{PAI-machine}`
- [ ] **DR drill:** Test one recovery scenario from `04-BACKUP-AND-RECOVERY.md` runbooks

**Gate:** Daily snapshot runs automatically. Backup appears on `{services-machine}`. OpenClaw backup accessible from `{PAI-machine}`. At least one DR scenario tested successfully.

---

## Phase 5: Communication

**Read:** `05-COMMUNICATION-PROTOCOLS.md`

- [ ] Install inter-agent channel tool on `{PAI-machine}` (adapt the OrphuChannel reference skill for your `{OpenClaw-agent}`)
- [ ] Configure gateway token in channel tool
- [ ] Configure gateway token in OpenClaw gateway (`~/.openclaw/openclaw.json`)
- [ ] Configure SSH alias for `{OpenClaw-agent-lowercase}@{OpenClaw-machine}` with LAN→Tailscale fallback
- [ ] Test Channel 1 (Direct): `{InterAgent-Tool} wake` returns healthy from `{OpenClaw-agent}`
- [ ] Test Channel 1: `{InterAgent-Tool} send "ping"` returns a real reply
- [ ] Verify gateway is loopback-only: `lsof -nP -iTCP:{gateway-port}` shows `127.0.0.1`
- [ ] Test Channel 2 (Telegram routing): `{InterAgent-Tool} telegram --topic ops "test"` arrives in the **Ops** topic only
- [ ] Verify topic separation: routine status → Ops, escalations → Relay, failures → Alerts
- [ ] Test Channel 3 (Mobile): `{principal}` DMs `{telegram-bot}` from phone, gets a response
- [ ] Test `dmPolicy=pairing`: a non-allowlisted Telegram account is rejected
- [ ] Test Channel 4 (File exchange): write a file to `{Sync-OpenClaw-dir}/comms/inbox/`, verify it appears at `{Agent-comms-inbox}/` on `{OpenClaw-machine}`
- [ ] Test Channel 4 reverse: `{OpenClaw-agent}` produces a file in `{Agent-wip-dir}/test/`, verify it appears on `{PAI-machine}` via Syncthing
- [ ] Test LAN→Tailscale fallback: temporarily block LAN, confirm SSH falls through to Tailscale
- [ ] Test file-drop degraded mode: stop the gateway, write a brief to `{Agent-comms-inbox}/`, confirm the next heartbeat picks it up

**Gate:** All 4 channels operational. SSH→loopback-WebSocket pattern verified. Topic routing correct. Fallback chain (LAN → Tailscale → file drop) tested.

---

## Phase 6: Multi-Agent Coordination

**Read:** `06-MULTI-AGENT-COORDINATION.md`

- [ ] Test task delegation: `{PAI-agent}` delegates a task to `{OpenClaw-agent}`
- [ ] Set up GitHub collaboration (if using):
  - [ ] Create shared private repo under `{github-username}`
  - [ ] Authorize SSH keys on both machines
  - [ ] Test: both agents can clone, push, pull
  - [ ] Configure branch protection on `main`
- [ ] Set up worker system (if using `{worker-machine}`):
  - [ ] Install CLI backends on `{worker-machine}`
  - [ ] Create job dispatch script on `{PAI-machine}`
  - [ ] Create worker launch script on `{worker-machine}`
  - [ ] Test: dispatch job, verify output returns
- [ ] Verify soul document access: `{PAI-agent}` can read `{OpenClaw-agent}`'s identity files
- [ ] Test end-to-end: `{principal}` directs `{PAI-agent}` → `{PAI-agent}` delegates to `{OpenClaw-agent}` → result returns

**Gate:** A real task flows through the full triad: `{principal}` asks, `{PAI-agent}` coordinates, `{OpenClaw-agent}` contributes, result delivered.

---

## Phase 7: Tasking Protocol and Heartbeat

**Read:** `08-TASKING-PROTOCOL.md`

- [ ] Create the workspace directory layout on `{OpenClaw-machine}`: `{Agent-comms-inbox}/`, `{Agent-comms-archive}/`, `{Agent-wip-dir}/`, `drafts/`, `notes/`, `memory/`
- [ ] Write initial `{Agent-workspace-dir}/CURRENT-TASK.md` with status `IDLE` and the required fields (status, owner, assigned_by, brief, last_checkpoint_utc, next_action, artifacts_path)
- [ ] Write initial `{Agent-workspace-dir}/HEARTBEAT.md` as a checklist (max ~60 lines — procedure, not knowledge base)
- [ ] Configure `agents.defaults.heartbeat` in `~/.openclaw/openclaw.json` with `interval`, `model`, `session: main`
- [ ] Wait for one heartbeat cycle and confirm it runs without errors (check gateway logs)
- [ ] Confirm heartbeat is silent when nothing is noteworthy (no spurious posts)
- [ ] Test file-first tasking: `{PAI-agent}` writes a brief to `{Agent-comms-inbox}/{PAI-AGENT-NAME}-TO-{OPENCLAW-AGENT-NAME}-test.md`, sends a wake signal via `{InterAgent-Tool} send`, confirm `{OpenClaw-agent}` reads the brief, transitions CURRENT-TASK.md to `IN_PROGRESS`, produces an artifact in `{Agent-wip-dir}/test/`, and archives the brief
- [ ] Test PRIORITY interrupt: send a HIGH/URGENT brief while a NORMAL task is in progress, confirm the agent pauses the original (sets `## Paused Task` block in CURRENT-TASK.md), works the interrupt, then resumes
- [ ] Test resume loop detection: stall a task and confirm the heartbeat detects it after 2 cycles and either produces output or escalates to Relay (do NOT post "Recovered from restart" without producing output)
- [ ] Verify the paste-ready relay format: paste a `[{OPENCLAW-AGENT-NAME}→{PAI-AGENT-NAME}] test` message into a Claude Code session and confirm `{PAI-agent}` responds with the two-zone format automatically
- [ ] Set up an external dead-man's-switch that alerts if no heartbeat ack arrives within 2× `{Heartbeat-Interval}` (silent failure mitigation)

**Gate:** A file-first brief flows end-to-end (write → wake → read → work → archive). A priority interrupt pauses and resumes correctly. The heartbeat fires on schedule and stays silent unless something's noteworthy.

---

## Phase 8: Validation and Hardening

**Read:** All files (review pass)

- [ ] Walk the full `INDEX.md` dependency map — does the flow make sense?
- [ ] Run a "day in the life" scenario: morning session, delegate work, check results, mobile query
- [ ] Review security logs after a full day of operation
- [ ] Verify backup ran automatically overnight
- [ ] Check Syncthing sync status across all machines
- [ ] Review and tune security patterns based on observed false positives/negatives
- [ ] Document any environment-specific tweaks in your `VARIABLES.md` comments
- [ ] Schedule credential rotation reminders (quarterly)

**Gate:** Full triad operates for 24+ hours without intervention. Backups, security, communication, and coordination all working.

---

## Quick Reference: What Goes Where

| What | Where | Reference |
|------|-------|-----------|
| Template values | `VARIABLES.md` (your copy) | VARIABLES.md |
| Credentials | Password manager only | SECRETS-SETUP.md |
| SSH config | `~/.ssh/config` on `{PAI-machine}` | 01 |
| Syncthing folders | Per hub-and-spoke topology | 01 |
| settings.json | `{PAI-dir}/settings.json` | 02 |
| Security patterns | `{PAI-dir}/skills/PAI/USER/PAISECURITYSYSTEM/` | 03 |
| PAI backup | macOS LaunchAgent → daily snapshot to `{PAI-backups-dir}` | 04 |
| OpenClaw backup | `openclaw backup create --verify` → weekly LaunchAgent on `{OpenClaw-machine}` | 04 |
| Channel tool | A skill on `{PAI-machine}` (adapted from OrphuChannel reference) | 05 |
| Gateway token | `~/.openclaw/openclaw.json` on `{OpenClaw-machine}` and the channel tool's secret store on `{PAI-machine}` | 05 / 07 |
| Telegram topic IDs | Stored as `{Telegram-Topic-Relay/Ops/Alerts}` in your VARIABLES.md (private) | 05 |
| Identity files (SOUL/IDENTITY/MEMORY/USER) | Inside `{Agent-workspace-dir}/` on `{OpenClaw-machine}` | 07 |
| Tasking inbox / archive / wip | `{Agent-comms-inbox}/`, `{Agent-comms-archive}/`, `{Agent-wip-dir}/` | 05 / 08 |
| OpenClaw binary | `/opt/homebrew/lib/node_modules/openclaw/` (managed by `admin` user) | 07 |
| OpenClaw gateway service | LaunchAgent at `~/Library/LaunchAgents/ai.openclaw.gateway.plist` | 07 |
| Heartbeat config | `~/.openclaw/openclaw.json → agents.defaults.heartbeat` | 07 |
| Heartbeat checklist | `{Agent-workspace-dir}/HEARTBEAT.md` | 07 / 08 |
| Current task state | `{Agent-workspace-dir}/CURRENT-TASK.md` | 08 |

---

*Created by Verbum, Ben's DA*
