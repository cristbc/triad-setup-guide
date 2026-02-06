# 01 — Host and Network

This is the physical and network foundation everything else builds on. I need machines, connectivity between them, encrypted tunnels for remote access, and file synchronization to keep state consistent.

---

## Machine Topology

The triad uses a minimum of two machines (PAI + OpenClaw), but the full pattern includes dedicated compute and services nodes:

| Role | Variable | OS | Purpose |
|------|----------|----|---------|
| **PAI workstation** | `{PAI-machine}` | macOS | My home. Runs Claude Code, all PAI infrastructure, orchestrates everything. |
| **OpenClaw host** | `{OpenClaw-machine}` | Ubuntu 24.04 | Dedicated to `{OpenClaw-agent}`. All compute reserved for the GPT agent. |
| **Services host** | `{services-machine}` | Linux (VM or bare-metal) | Docker containers: vector DB, local LLM, media server. |
| **Worker node** | `{worker-machine}` | macOS | Parallel delegation target. Receives compute jobs from `{PAI-machine}`. |
| **Hypervisor** | `{hypervisor-machine}` | Proxmox (optional) | Hosts `{services-machine}` as a VM. Only needed if you virtualize. |

**Minimum viable triad:** `{PAI-machine}` + `{OpenClaw-machine}`. The services and worker nodes extend capability but aren't required to start.

![Network Topology — LAN + Tailscale Overlay](diagrams/network-topology.png)

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

**Linux (`{OpenClaw-machine}`, `{services-machine}`):**
```bash
sudo apt install syncthing
systemctl --user enable syncthing
systemctl --user start syncthing
# Web GUI at http://localhost:8384
```

### Pairing

1. Start Syncthing on each machine to generate device IDs
2. On `{PAI-machine}` web GUI: Add Remote Device → paste `{Syncthing-device-ID-*}`
3. On spoke machine: accept the connection request from `{PAI-machine}`
4. Create shared folders and assign them to paired devices
5. Verify sync by creating a test file

**Important:** After an OS reinstall on any spoke, you must re-pair. The device ID changes and the old pairing is invalid.

---

## OpenClaw Machine — Linux-Specific Configuration

`{OpenClaw-machine}` runs Ubuntu 24.04 dedicated to `{OpenClaw-agent}`. A few Linux-specific settings ensure reliable headless operation:

### Prevent Lid-Close Suspend

If using a laptop as the OpenClaw host, disable suspend on lid close:

```
# /etc/systemd/logind.conf
HandleLidSwitch=ignore
HandleLidSwitchExternalPower=ignore
HandleLidSwitchDocked=ignore
```

Then: `sudo systemctl restart systemd-logind`

### Network (Headless Ethernet)

For a headless server installation, you may prefer systemd-networkd + netplan. If running Ubuntu Desktop, NetworkManager works fine — skip this section. Example netplan config:

```yaml
# /etc/netplan/01-ethernet.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    {ethernet-interface}:
      dhcp4: true
      # Or static:
      # addresses: [{OpenClaw-LAN-IP}/24]
      # routes:
      #   - to: default
      #     via: {gateway-IP}
```

Apply: `sudo netplan apply`

### Disk Encryption

Consider LUKS full-disk encryption for the OpenClaw machine, especially if it's a repurposed laptop. This protects `{OpenClaw-agent}`'s data at rest if the machine is lost or stolen.

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
