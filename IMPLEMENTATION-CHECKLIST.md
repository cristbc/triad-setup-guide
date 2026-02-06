# Implementation Checklist

A step-by-step guide for a receiving PAI agent (or human) to stand up the full triad. Each phase has a verification gate — don't proceed until the gate passes.

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
  - [ ] Install and start voice server
  - [ ] Verify voice server responds: test with its documented endpoint or test command
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

- [ ] Install inter-agent channel tool on `{PAI-machine}`
- [ ] Configure gateway token in channel tool
- [ ] Configure gateway token in OpenClaw gateway
- [ ] Test Channel 1 (Direct): Send message to `{OpenClaw-agent}`, get response
- [ ] Test Channel 2 (Telegram relay): Send message via `{OpenClaw-agent}` to `{principal}`'s phone
- [ ] Test Channel 3 (Mobile): Message `{OpenClaw-agent}` directly from Telegram
- [ ] Test Channel 4 (File exchange): Place file in Syncthing folder, verify arrival
- [ ] Test fallback: Disconnect LAN, verify Tailscale fallback works
- [ ] Test file-drop fallback: Stop gateway, verify file-based communication works

**Gate:** All 4 channels operational. Fallback chain (LAN → Tailscale → file drop) tested.

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

## Phase 7: Validation and Hardening

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
| Backup script | LaunchAgent or systemd timer | 04 |
| Channel tool | `{PAI-dir}/tools/` | 05 |
| Gateway token | `{OpenClaw-config-dir}/` and channel tool | 05 |
| Soul documents | `{soul-docs-dir}/` on `{OpenClaw-machine}` | 07 |
| OpenClaw service | `~/.config/systemd/user/` on `{OpenClaw-machine}` | 07 |

---

*Created by Verbum, Ben's DA*
