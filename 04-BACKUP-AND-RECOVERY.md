# 04 — Backup and Recovery

My infrastructure spans multiple machines, and any one of them could die at any time. A fried SSD, a spilled coffee, a power surge — I need to survive all of it without losing meaningful state. This document describes the 3-layer backup topology I maintain and the disaster recovery runbooks for getting back to operational after every failure scenario.

---

## Backup Philosophy

I follow a modified 3-2-1 rule:

- **3 copies** of everything critical (local, distributed, offsite)
- **2 different media types** (SSD/disk + portable drive)
- **1 offsite** copy (physically separated from the home network)

The key principle: **no human remembering to back up.** Every backup that requires a human to initiate will eventually be skipped. My snapshots run on a timer. My distributed copies sync automatically. The only manual step is the monthly offsite rotation, and even that has a calendar reminder.

What I protect, in priority order:

1. `{PAI-dir}` — My entire operational state: memory, skills, hooks, settings, history
2. `{PAI-framework-dir}` — The PAI framework source and tools
3. `{OpenClaw-config-dir}` and `{soul-docs-dir}` — `{OpenClaw-agent}`'s configuration and identity
4. Service configurations — Docker compose files, volumes, environment configs

![3-Layer Backup Topology](diagrams/backup-topology.png)

---

## Layer 1: Daily Automated Snapshots

A LaunchAgent on `{PAI-machine}` fires daily and creates a compressed archive of `{PAI-dir}`. No human involvement. If `{PAI-machine}` is asleep at the scheduled time, macOS runs the job on next wake.

### What Gets Archived

The snapshot captures `{PAI-dir}` in its entirety: memory system, skills, hooks, settings, history, and any state files. This is a complete point-in-time image of my operational state.

### Naming Convention

```
PAI-SNAPSHOT-YYYY-MM-DD.tar.gz
```

Archives land in `{PAI-backups-dir}`. I retain 7 days of snapshots. A cleanup step in the same script removes anything older, so the directory never grows unbounded.

### LaunchAgent Configuration

On macOS, I use a LaunchAgent plist to schedule the daily job:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.pai.daily-backup</string>
    <key>ProgramArguments</key>
    <array>
        <string>/bin/bash</string>
        <string>{PAI-dir}/tools/backup-snapshot.sh</string>
    </array>
    <key>StartCalendarInterval</key>
    <dict>
        <key>Hour</key>
        <integer>3</integer>
        <key>Minute</key>
        <integer>0</integer>
    </dict>
    <key>StandardOutPath</key>
    <string>{PAI-backups-dir}/snapshot.log</string>
    <key>StandardErrorPath</key>
    <string>{PAI-backups-dir}/snapshot-error.log</string>
</dict>
</plist>
```

The plist lives at `~/Library/LaunchAgents/com.pai.daily-backup.plist`. Load it with:

```bash
launchctl load ~/Library/LaunchAgents/com.pai.daily-backup.plist
```

### Backup Script Logic

The script itself is straightforward:

1. Create timestamp: `YYYY-MM-DD`
2. `tar czf {PAI-backups-dir}/PAI-SNAPSHOT-$DATE.tar.gz -C ~ {PAI-dir-basename}`
3. Delete any `PAI-SNAPSHOT-*.tar.gz` files older than 7 days
4. Log success or failure

On Linux machines (if used as the PAI host instead of macOS), replace the LaunchAgent with a systemd timer:

```ini
# ~/.config/systemd/user/pai-backup.timer
[Unit]
Description=Daily PAI backup

[Timer]
OnCalendar=*-*-* 03:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

```ini
# ~/.config/systemd/user/pai-backup.service
[Unit]
Description=PAI backup snapshot

[Service]
Type=oneshot
ExecStart={PAI-dir}/tools/backup-snapshot.sh
```

Enable with: `systemctl --user enable --now pai-backup.timer`

---

## Layer 2: Distributed Copies via Syncthing

Layer 1 creates the snapshots. Layer 2 distributes them so they survive the loss of `{PAI-machine}` itself.

### Automatic Distribution

`{PAI-backups-dir}` is a Syncthing shared folder configured as **send-only** from `{PAI-machine}` to `{services-machine}`. Syncthing writes to `{services-backups-dir}` on the receiving end (this may differ from `{PAI-backups-dir}` — define both in your VARIABLES.md). Every time a new snapshot appears, Syncthing pushes it automatically. No cron, no rsync — it just happens.

Additionally, `{PAI-framework-dir}` syncs to `{services-machine}` as a read-only mirror. This means `{services-machine}` always has both:
- The last 7 daily snapshots of my operational state
- A current copy of the PAI framework source

If `{PAI-machine}` is destroyed, `{services-machine}` has everything I need to rebuild.

### Physical Media Backups

For true resilience beyond the network:

| Medium | Frequency | Method | Location |
|--------|-----------|--------|----------|
| USB drive (home) | Daily | `rsync` from `{PAI-backups-dir}` | Plugged into `{PAI-machine}` or `{services-machine}` |
| Portable drive | Weekly/intermittent | Manual `rsync` or Time Machine | Taken offsite periodically |

The USB drive gives me a third copy on different physical media. The portable drive provides the offsite component. Together with the Syncthing distribution, I have copies on 3 different physical devices in at least 2 locations.

---

## Layer 3: Full System Backup

Snapshots protect PAI state. But `{PAI-machine}` is more than PAI — it has OS configuration, application state, SSH keys, browser profiles, development tools, and everything else that makes it a working machine. Layer 3 captures all of it.

### Time Machine (macOS)

I run Time Machine to an encrypted portable drive. This provides:
- Full OS restoration (boot from recovery, restore everything)
- File-level recovery (grab a single file from any point in time)
- Application state I can't easily recreate (preferences, licenses, keychains)

**Encryption is mandatory.** The backup drive contains the full contents of my machine, including credentials stored in the keychain.

### Monthly Offsite Rotation

Once a month, I take the encrypted portable drive to a physically separate location — a safe, a trusted friend's house, a bank deposit box. The key insight: if my home has a fire, flood, or burglary, all network-connected backups in the house are gone. The offsite copy survives that.

Rotation cycle:
1. Run a full Time Machine backup to the portable drive
2. Verify the backup completed successfully
3. Take the drive offsite
4. Bring the previous offsite drive home (if rotating two drives)

---

## Disaster Recovery Runbooks

![Disaster Recovery Source Matrix](diagrams/dr-recovery-matrix.png)

Four scenarios, ordered by likelihood. Each includes the steps to get back to operational and the expected recovery time.

---

### Scenario 1: `{PAI-machine}` Failure (Total Loss)

**Situation:** My primary workstation is dead. Hardware failure, theft, catastrophic damage. I've lost everything on that machine.

**Recovery source:** `{services-machine}` (Syncthing copies) + Time Machine drive

**Steps:**

1. **Acquire replacement hardware**
   - Any macOS machine will work. Same model preferred but not required.

2. **Restore macOS from Time Machine**
   - Boot into recovery mode, select Time Machine backup
   - This restores the full OS, applications, SSH keys, system configuration
   - If no Time Machine available, do a clean macOS install and proceed to step 3

3. **Install Claude Code**
   - Download and install Claude Code
   - Authenticate with Anthropic account
   - This gives me a fresh `{PAI-dir}` skeleton

4. **Restore PAI state from backup**
   - Pull the latest snapshot from `{services-machine}`:
     ```
     scp {services-machine}:{services-backups-dir}/PAI-SNAPSHOT-latest.tar.gz ~/
     ```
   - Or retrieve from the offsite drive if `{services-machine}` is also down
   - Extract over the fresh `{PAI-dir}`:
     ```
     tar xzf PAI-SNAPSHOT-latest.tar.gz -C ~
     ```

5. **Restore PAI framework**
   - Copy from `{services-machine}` Syncthing mirror or clone from `{PAI-repo}`

6. **Re-pair Syncthing**
   - Install Syncthing on the new machine
   - The new machine has a new device ID — old pairings are invalid
   - Add `{Syncthing-device-ID-OpenClaw}`, `{Syncthing-device-ID-services}`, `{Syncthing-device-ID-worker}` as remote devices
   - Recreate shared folders per the topology in `01-HOST-AND-NETWORK.md`

7. **Restore Tailscale**
   - `tailscale up --ssh` and authenticate
   - The old device will appear as offline in the Tailscale admin — remove it
   - Update any references to the old Tailscale IP if it changed

8. **Verify operational state**
   - All hooks fire correctly
   - Skills load and execute
   - Voice server starts and speaks
   - Inter-agent channel tool reaches `{OpenClaw-agent}`
   - Syncthing shows all folders in sync
   - Daily backup LaunchAgent is loaded

**RTO target:** 1-2 hours (with Time Machine), 2-4 hours (clean install + snapshot restore)

---

### Scenario 2: `{OpenClaw-machine}` Failure (Total Loss)

**Situation:** The machine running `{OpenClaw-agent}` is dead. I've lost the operating system, all local data, and the agent's configuration.

**Recovery source:** `{Sync-OpenClaw-dir}/backups/` on `{PAI-machine}` + OpenClaw source repository

**Steps:**

1. **Fresh OS install**
   - Install Ubuntu 24.04 on replacement hardware
   - During install: set hostname to `{OpenClaw-machine}`, create user `{principal-lowercase}`

2. **Run post-install script**
   - Install essential packages: `build-essential`, `curl`, `git`, `tmux`, `jq`
   - Install and authenticate Tailscale: `tailscale up --ssh`
   - Install Syncthing: `sudo apt install syncthing && systemctl --user enable --now syncthing`
   - Configure lid-close behavior (if laptop) per `01-HOST-AND-NETWORK.md`

3. **Restore from backup archive**
   - My OpenClaw backup script stores timestamped archives in `{Sync-OpenClaw-dir}/backups/` which syncs to `{PAI-machine}`
   - Copy the latest archive to the new machine:
     ```
     scp {PAI-machine}:{Sync-OpenClaw-dir}/backups/openclaw-backup-latest.tar.gz ~/
     ```
   - Extract to home directory

4. **Reinstall OpenClaw from source**
   - Clone the OpenClaw repository to `{OpenClaw-install-dir}`
   - Install dependencies per OpenClaw documentation
   - Copy restored configuration into `{OpenClaw-config-dir}`

5. **Restore systemd services**
   - Copy restored service files to `~/.config/systemd/user/`
   - Reload: `systemctl --user daemon-reload`
   - Enable and start: gateway, Telegram bot, scheduled tasks
   - Verify each service is active: `systemctl --user status <service>`

6. **Restore soul documents**
   - Copy restored soul docs to `{soul-docs-dir}`
   - These define `{OpenClaw-agent}`'s identity and personality — without them, the agent works but isn't *itself*

7. **Restore SSH authorized keys**
   - Copy `{SSH-automation-key}.pub` from `{PAI-machine}` to `~/.ssh/authorized_keys` on the new machine
   - Restrict to LAN: `from="{LAN-subnet}" ssh-ed25519 AAAA... homelab-automation`

8. **Re-pair Syncthing with `{PAI-machine}`**
   - New device ID generated on fresh install
   - Add `{PAI-machine}` as remote device, share `{Sync-OpenClaw-dir}` folder
   - Accept pairing on `{PAI-machine}` side
   - Remove the old device entry on `{PAI-machine}`

9. **Verify operational state**
   - Gateway responds to WebSocket connections
   - Telegram bot receives and responds to messages
   - Scheduled tasks fire on time
   - `{PAI-agent}` can reach `{OpenClaw-agent}` via inter-agent channel
   - Syncthing shows folder in sync

**RTO target:** 30-45 minutes

---

### Scenario 3: `{services-machine}` Failure

**Situation:** The Docker services host is down. This affects backup storage, containerized services (vector DB, local LLM, media server), but not core agent operations.

**Recovery source:** `{Sync-services-dir}` on `{PAI-machine}` + container image registries

**Steps:**

1. **Recreate the machine**
   - If virtualized: create a new VM on `{hypervisor-machine}` with the same specs
   - If bare-metal: fresh Linux install
   - Install Docker and Docker Compose

2. **Restore Docker Compose files**
   - Copy from `{Sync-services-dir}` on `{PAI-machine}`:
     ```
     scp -r {PAI-machine}:{Sync-services-dir}/docker/ ~/docker/
     ```
   - These contain the compose files and environment configurations for all services

3. **Pull container images and restore volumes**
   - `docker compose pull` in each service directory
   - Restore volume data from backup archives in `{Sync-services-dir}/backups/` if available
   - For services without volume backups, they'll start fresh (acceptable for caches, not for persistent data)

4. **Install and configure Syncthing**
   - Install Syncthing, start the service
   - Pair with `{PAI-machine}` (new device ID)
   - Recreate shared folders: `{Sync-services-dir}`, PAI framework mirror, `{PAI-backups-dir}` mirror

5. **Install Tailscale**
   - `tailscale up --ssh` and authenticate
   - Remove old device from Tailscale admin

6. **Start services**
   - `docker compose up -d` for each service stack
   - Verify each service is healthy: `docker compose ps`

7. **Verify backup flow**
   - Confirm `{PAI-backups-dir}` snapshots are syncing to the new machine
   - Confirm PAI framework mirror is current

**RTO target:** 30-60 minutes (longer if large Docker volumes need restoration)

**Impact during downtime:** Core agent operations are unaffected. I lose backup redundancy (Layer 2 partially degraded), containerized services (vector DB queries, local LLM inference), and media serving. These are important but not critical to the triad's communication and coordination.

---

### Scenario 4: Network Failure (LAN Down)

**Situation:** The local network is down — router failure, switch failure, ISP outage. Machines are physically fine but can't reach each other over LAN.

**This is the least disruptive failure because of how the network is layered.**

**What still works:**

- **Tailscale** provides encrypted mesh connectivity independent of the LAN. As long as machines have any internet connectivity (Wi-Fi hotspot, cellular tethering), they can reach each other via Tailscale IPs.
- **SSH via Tailscale:** `ssh {principal-lowercase}@{OpenClaw-TS-IP}` works when LAN SSH doesn't.
- **Syncthing pauses and resumes automatically.** When LAN connectivity returns, Syncthing detects the peers and syncs any changes that accumulated during the outage. No manual intervention.
- **Inter-agent communication falls back to Tailscale addresses.** My channel tool tries `{OpenClaw-LAN-IP}` first, then falls back to `{OpenClaw-TS-IP}`. If Tailscale is up, the agents keep talking.

**What degrades:**

- Latency increases (Tailscale routes through relay servers if no direct path)
- Syncthing sync may pause until connectivity returns (it buffers gracefully)
- Large file transfers are slower over Tailscale relay vs. Gigabit LAN

**What to do:**

1. Verify Tailscale connectivity: `tailscale status`
2. If Tailscale is up, continue working normally — just slower
3. If Tailscale is also down (no internet at all), work locally on `{PAI-machine}` and sync later
4. When LAN returns, verify Syncthing has caught up: check web GUI at `http://localhost:8384`

**RTO target:** 0 minutes (automatic failover to Tailscale) or "wait for LAN to return" if no internet

---

## OpenClaw Backup Specifics

`{OpenClaw-agent}`'s state is spread across several locations on `{OpenClaw-machine}`. My backup script captures all of them into a single timestamped archive:

### What the Backup Script Captures

| Component | Path | Why It Matters |
|-----------|------|---------------|
| OpenClaw configuration | `{OpenClaw-config-dir}` | Agent settings, gateway config, API keys |
| systemd service files | `~/.config/systemd/user/` | Service definitions for gateway, bot, tasks |
| SSH authorized keys | `~/.ssh/authorized_keys` | Allows `{PAI-machine}` to connect |
| Soul documents | `{soul-docs-dir}` | Agent identity, personality, behavioral rules |
| Scheduled tasks | Crontab or systemd timers | Recurring jobs the agent runs |

### Archive Format

```
openclaw-backup-YYYY-MM-DD-HHMMSS.tar.gz
```

### Where It Goes

The archive is written to `{Sync-OpenClaw-dir}/backups/` on `{OpenClaw-machine}`. Because `{Sync-OpenClaw-dir}` is a bidirectional Syncthing folder, the archive automatically syncs to `{PAI-machine}`. This means I always have a copy of `{OpenClaw-agent}`'s backup on my own machine, even if `{OpenClaw-machine}` dies.

### When to Run

- **Before any OpenClaw update** (new version, config change)
- **Before any OS change** (system update, package upgrade)
- **Periodically** via cron or systemd timer (daily recommended)
- **Before hardware changes** (moving to new machine, disk swap)

---

## Verification

After setting up the backup infrastructure, confirm:

- [ ] LaunchAgent (or systemd timer) is loaded and scheduled
- [ ] Manual snapshot test: run the backup script, verify archive appears in `{PAI-backups-dir}`
- [ ] Verify archive integrity: `tar tzf PAI-SNAPSHOT-YYYY-MM-DD.tar.gz | head`
- [ ] Syncthing shows `{PAI-backups-dir}` folder syncing to `{services-machine}`
- [ ] Snapshot appears on `{services-machine}` within minutes of creation
- [ ] 7-day retention working: after 8+ days, oldest snapshot is removed
- [ ] OpenClaw backup script runs and produces archive in `{Sync-OpenClaw-dir}/backups/`
- [ ] OpenClaw backup syncs to `{PAI-machine}` via Syncthing
- [ ] Time Machine running and encrypted to portable drive
- [ ] Test Time Machine restore: recover a single file to verify the backup is valid
- [ ] Offsite drive rotation schedule set (monthly calendar reminder)
- [ ] Each disaster recovery runbook reviewed and understood by `{principal}`

---

*Created by Verbum, Ben's DA*
