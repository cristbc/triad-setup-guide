# 01 — Host and Network

This is the physical and network foundation everything else builds on. I need machines, connectivity between them, encrypted tunnels for remote access, and file synchronization to keep state consistent.

---

## Machine Topology

The triad uses a minimum of two machines (PAI + OpenClaw), but the full pattern includes dedicated compute and services nodes:

| Role | Variable | OS | Purpose |
|------|----------|----|---------|
| **PAI workstation** | `{PAI-machine}` | macOS | My home. Runs Claude Code, all PAI infrastructure, orchestrates everything. |
| **OpenClaw host** | `{OpenClaw-machine}` | macOS (Mac mini works well) | Dedicated to `{OpenClaw-agent}` under its own user account. All compute reserved for the GPT agent. |
| **Services host** | `{services-machine}` | Linux (VM or bare-metal) | Docker services: reverse proxy, file sharing, fleet dashboard. Services evolve — document current stack in VARIABLES.md. |
| **Worker node** | `{worker-machine}` | macOS | Parallel delegation target. Receives compute jobs from `{PAI-machine}`. |
| **Hypervisor** | `{hypervisor-machine}` | Proxmox (optional) | Hosts `{services-machine}` as a VM. Only needed if you virtualize. |

**Note:** The hypervisor role is optional. If `{services-machine}` runs on bare-metal hardware, this role is eliminated entirely.

**On host OS choice for `{OpenClaw-machine}`:** The reference triad runs OpenClaw on macOS — installed via `npm install -g openclaw` against Homebrew node, services managed by `launchctl`. A second user account on the OpenClaw host owns the agent's workspace and runs the gateway under launchd. Linux/VPS deployments work structurally but are not currently validated against the live OpenClaw release; see the "Alternate hosts" sidebar in `07-OPENCLAW-APPENDIX.md`.

**Minimum viable triad:** `{PAI-machine}` + `{OpenClaw-machine}`. The services and worker nodes extend capability but aren't required to start.

> **Diagram:** see [`diagrams/network-topology.md`](diagrams/network-topology.md) — LAN + Tailscale overlay with all four hosts and the two-user OpenClaw layout.

---

## Network Layer

### LAN First

All machines connect via Ethernet to a local switch. I prefer wired connections for reliability and to avoid Wi-Fi latency during inter-machine operations. Wi-Fi works but introduces failure modes I'd rather avoid.

- All machines on `{LAN-subnet}`
- Unmanaged Gigabit switch (no special configuration needed)
- Each machine gets a static or DHCP-reserved IP

### Tailscale Overlay

Tailscale provides encrypted mesh networking for when `{principal}` is away from the LAN. Every machine joins the same Tailscale account.

**Setup per machine:**
1. Install Tailscale
2. `tailscale up` and authenticate
3. Enable Tailscale SSH: `tailscale up --ssh`
4. Record the assigned Tailscale IP

**What Tailscale gives me:**
- Encrypted connectivity from anywhere (coffee shop, travel) without port forwarding
- Tailscale SSH means I can reach machines without managing SSH keys for remote access
- Stable addresses even when LAN IPs change
- MagicDNS for hostname resolution across the tailnet

**My connection preference order:**
1. LAN IP (fastest, no overhead)
2. Tailscale IP (encrypted, works anywhere)
3. Tailscale SSH (no key management)

I always try LAN first, fall back to Tailscale. My inter-agent channel tool does this automatically.

---

## SSH Configuration

### Two Key Strategy

I maintain two SSH keys with different security profiles:

| Key | Variable | Passphrase | Scope | Purpose |
|-----|----------|-----------|-------|---------|
| Automation | `{SSH-automation-key}` | None | LAN only | Scripts, cron, delegation |
| GitHub | `{SSH-github-key}` | Yes | GitHub | Code push/pull |

The automation key has no passphrase so scripts can use it unattended. To compensate, I restrict it server-side:

```
# In ~/.ssh/authorized_keys on target machines:
from="{LAN-subnet}" ssh-ed25519 AAAA... homelab-automation
```

This means even if the key leaks, it only works from the local network. Once key-based auth is confirmed working, also disable password auth: set `PasswordAuthentication no` in `/etc/ssh/sshd_config` on all target machines.

### SSH Config Patterns

On `{PAI-machine}`, my `~/.ssh/config` maps hostnames to connection details:

```
Host {OpenClaw-machine}
    HostName {OpenClaw-LAN-IP}
    User {principal-lowercase}
    IdentityFile ~/.ssh/{SSH-automation-key}

Host {services-machine}
    HostName {services-LAN-IP}
    User {principal-lowercase}
    IdentityFile ~/.ssh/{SSH-automation-key}

Host {worker-machine}
    HostName {worker-LAN-IP}
    User {worker-username}
    IdentityFile ~/.ssh/{SSH-automation-key}
```

This lets me `ssh {OpenClaw-machine}` without remembering IPs or specifying keys.

### Portable Dotfiles over SSH

I use a dotfile-forwarding tool (like kyrat or similar) so my shell preferences travel with me when I SSH into other machines. Combined with terminal background tinting per host, I always know which machine I'm on at a glance.

---

## Syncthing — Hub-and-Spoke File Sync

Syncthing provides real-time, peer-to-peer file synchronization over LAN. I use it for:
- Exchanging files with `{OpenClaw-agent}`
- Distributing delegation jobs to `{worker-machine}`
- Backing up PAI framework to `{services-machine}`
- Sharing service configs with `{services-machine}`

### Topology

`{PAI-machine}` is the hub. Every other machine is a spoke. Spokes never sync directly with each other — all data flows through the hub. This keeps the trust model simple: I only need to authorize connections from `{PAI-machine}`.

```
                    {PAI-machine} (hub)
                   /    |    \      \
                  /     |     \      \
{OpenClaw-machine}  {worker-machine}  {services-machine} (x2 folders)
```

### Shared Folders

| Folder Name | Path on Hub | Synced With | Direction | Purpose |
|------------|-------------|-------------|-----------|---------|
| `PAI-{OpenClaw-suffix}` | `~/{Sync-OpenClaw-dir}` | `{OpenClaw-machine}` | Bidirectional | File exchange + soul docs |
| `PAI-{worker-suffix}` | `~/{Sync-worker-dir}` | `{worker-machine}` | Bidirectional | Job dispatch + output retrieval |
| `PAI-{services-suffix}` | `~/{Sync-services-dir}` | `{services-machine}` | Bidirectional | Service config exchange |
| `PAI` | `~/PAI` | `{services-machine}` | Send only | Framework backup (read-only mirror) |
| `PAI-Backups` | `~/{PAI-backups-dir}` | `{services-machine}` | Send only | Daily snapshot backup |

### Installation

**macOS (`{PAI-machine}`, `{worker-machine}`):**
```bash
brew install syncthing
brew services start syncthing
# Web GUI at http://localhost:8384
```

**macOS additional users (`{OpenClaw-machine}` running OpenClaw under its own user):**
```bash
# As the agent user on {OpenClaw-machine}:
brew install syncthing
brew services start syncthing
# Web GUI at http://localhost:8384
```

**Linux (`{services-machine}` only, if you use a Linux services host):**
```bash
sudo apt install syncthing
systemctl --user enable --now syncthing.service
# Web GUI at http://localhost:8384
```

### Pairing

1. Start Syncthing on each machine to generate device IDs
2. On `{PAI-machine}` web GUI: Add Remote Device → paste `{Syncthing-device-ID-*}`
3. On spoke machine: accept the connection request from `{PAI-machine}`
4. Create shared folders and assign them to paired devices
5. Verify sync by creating a test file

**Important:** After an OS reinstall on any spoke, you must re-pair. The device ID changes and the old pairing is invalid.

**Maintenance tip:** Review your Syncthing shared folders periodically. Zombie syncs — folders still configured to sync with devices that no longer exist — waste resources and generate spurious warnings. Prune any dead device pairings and remove orphaned folder configurations.

---

## OpenClaw Machine — Platform-Specific Configuration

`{OpenClaw-machine}` runs macOS dedicated to `{OpenClaw-agent}`. A Mac mini works well — small, silent, low-power, and runs 24/7 with the lid closed (or with no lid at all). Detailed install steps live in `07-OPENCLAW-APPENDIX.md`. The host-level concerns below apply regardless of which OpenClaw release you install.

### Two-User Layout

I run two macOS user accounts on `{OpenClaw-machine}`:

| User | Owns | Why |
|------|------|-----|
| `admin` | `/opt/homebrew` (Homebrew + global npm packages) | The OpenClaw binary is installed via `npm install -g openclaw` and lives under `admin:staff`. Updates run as `admin`. |
| `{OpenClaw-agent-lowercase}` | `{Agent-workspace-dir}`, `~/.openclaw/`, `~/Library/LaunchAgents/ai.openclaw.gateway.plist` | The agent runs as its own user. This isolates its workspace, secrets, and LaunchAgent from the admin user and from `{principal}`'s personal files. |

This split keeps the binary in one place (admin can update it once) while giving each agent on the host its own isolated home directory. If you ever add a second OpenClaw agent on the same Mac mini, it gets its own user account too.

### Stay Awake

Even without a lid, macOS will sleep on idle. Configure power settings so the OpenClaw host stays awake whenever it's on AC power:

```bash
sudo pmset -c sleep 0 displaysleep 10 disksleep 0
sudo pmset -c womp 1   # Wake on network access
```

For a headless Mac mini you can also disable display sleep entirely (`displaysleep 0`).

### SSH Aliases for Multi-User Hosts

Because the OpenClaw host has two user accounts, the simplest way to address each one is with a per-user SSH alias on `{PAI-machine}`. Example `~/.ssh/config` block:

```
Host {OpenClaw-agent-lowercase}
    HostName {OpenClaw-LAN-IP}
    User {OpenClaw-agent-lowercase}
    IdentityFile ~/.ssh/{SSH-automation-key}

Host {OpenClaw-machine}
    HostName {OpenClaw-LAN-IP}
    User admin
    IdentityFile ~/.ssh/{SSH-automation-key}
```

`ssh {OpenClaw-agent-lowercase}` lands you in the agent user's session for workspace operations, gateway management, and `openclaw` CLI calls. `ssh {OpenClaw-machine}` lands you in the admin session for OpenClaw binary updates via npm.

**Tailscale fallback:** Add a `Match exec` block (or use Tailscale SSH) so the same alias falls through to the Tailscale IP when the LAN is unreachable.

### Disk Encryption

Enable FileVault on `{OpenClaw-machine}`. The agent's workspace contains task history, conversation context, and identity documents — encrypted-at-rest is non-negotiable, especially if the machine is portable.

---

## Verification

After completing this section, confirm:

- [ ] All machines reachable via LAN IP from `{PAI-machine}`
- [ ] Tailscale installed and authenticated on all machines
- [ ] Tailscale SSH working: `ssh {principal}@{OpenClaw-TS-IP}`
- [ ] SSH automation key connects without password to all targets
- [ ] SSH config resolves hostnames: `ssh {OpenClaw-machine}` works
- [ ] Syncthing running on all machines
- [ ] All shared folders created and syncing
- [ ] Test file created on `{PAI-machine}` appears on synced spoke
- [ ] OpenClaw machine stays awake with lid closed (if laptop)

---

*Created by Verbum, Ben's DA*
