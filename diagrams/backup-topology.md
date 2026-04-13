# 3-Layer Backup Topology

Embed in `04-BACKUP-AND-RECOVERY.md` after the "Backup Philosophy" section.

```mermaid
graph TB
    subgraph L1["Layer 1 — Daily/Weekly Snapshots (on-host)"]
        PAI_SNAP["{PAI-machine}<br/><i>daily LaunchAgent</i><br/>tar → {PAI-backups-dir}/<br/>PAI-SNAPSHOT-YYYY-MM-DD.tar.gz<br/>(7-day retention)"]
        OC_SNAP["{OpenClaw-machine}<br/><i>weekly LaunchAgent<br/>(com.{principal-lowercase}.openclaw-backup)</i><br/>openclaw backup create --verify<br/>→ ~/Backups/<br/>(28-day retention)"]
    end

    subgraph L2["Layer 2 — Distributed Copies (Syncthing)"]
        SVC_BACKUPS["{services-machine}<br/>{services-backups-dir}<br/><i>send-only mirror</i>"]
        PAI_OC_MIRROR["{PAI-machine}<br/>{Sync-OpenClaw-dir}/backups/<br/><i>one-way Syncthing share<br/>from {OpenClaw-machine}</i>"]
    end

    subgraph L3["Layer 3 — Full System + Offsite"]
        TM["Time Machine<br/>(encrypted portable drive)<br/>full {PAI-machine} OS + state"]
        OFFSITE["Monthly offsite rotation<br/>(safe / friend's house /<br/>safety deposit box)"]
    end

    PAI_SNAP -->|"Syncthing"| SVC_BACKUPS
    OC_SNAP -->|"Syncthing"| PAI_OC_MIRROR
    PAI_SNAP -->|"file-level included"| TM
    TM -->|"physical rotation"| OFFSITE

    PAI_FW["{PAI-framework-dir}<br/>(read-only mirror)"]
    PAI_FW -->|"Syncthing"| SVC_BACKUPS

    PAI_VERIFY{{"openclaw backup verify<br/>+ tar tzf integrity check"}}
    PAI_OC_MIRROR -.->|"manual integrity test"| PAI_VERIFY
    SVC_BACKUPS -.->|"manual integrity test"| PAI_VERIFY

    style PAI_SNAP fill:#1E40AF,stroke:#3B82F6,color:#F9FAFB
    style OC_SNAP fill:#7C2D12,stroke:#F97316,color:#F9FAFB
    style SVC_BACKUPS fill:#1A3D1A,stroke:#22C55E,color:#F9FAFB
    style PAI_OC_MIRROR fill:#1A3D1A,stroke:#22C55E,color:#F9FAFB
    style PAI_FW fill:#374151,stroke:#9CA3AF,color:#F9FAFB
    style TM fill:#3D1A5B,stroke:#A855F7,color:#F9FAFB
    style OFFSITE fill:#5B4A1A,stroke:#F59E0B,color:#F9FAFB
    style PAI_VERIFY fill:#1F2937,stroke:#6B7280,color:#F9FAFB
```

**Reading notes:**
- **Layer 1** is on-host: snapshots are created on the source machine and stored locally first. PAI gets a daily tarball; OpenClaw gets a weekly verified archive via the native `openclaw backup create --verify` command. No hand-rolled `tar -czf` — the OpenClaw command knows about config, workspace, identity, and plugin state and excludes the right things.
- **Layer 2** is distributed: Syncthing one-way mirrors push the snapshots to other hosts within minutes of creation. `{PAI-backups-dir}` and `{PAI-framework-dir}` mirror to `{services-machine}`. OpenClaw's `~/Backups/` mirrors to `{Sync-OpenClaw-dir}/backups/` on `{PAI-machine}`. These are **send-only** — receivers never write back.
- **Layer 3** is full-system + offsite: Time Machine to an encrypted portable drive captures the OS + applications + keychain. Once a month, the drive goes offsite. This survives a house fire that takes out the LAN.
- **Verification matters more than frequency.** `openclaw backup verify <archive>` re-checks an existing archive; `tar tzf` on PAI snapshots tests integrity. A silent backup failure is worse than no backup at all.
