# Disaster Recovery — Source Matrix

Embed in `04-BACKUP-AND-RECOVERY.md` after the "Disaster Recovery Runbooks" section header.

```mermaid
graph LR
    subgraph FAILURES["Failure Scenarios"]
        F1["Scenario 1<br/>{PAI-machine}<br/>total loss"]
        F2["Scenario 2<br/>{OpenClaw-machine}<br/>total loss"]
        F3["Scenario 3<br/>{services-machine}<br/>failure"]
        F4["Scenario 4<br/>LAN<br/>failure"]
    end

    subgraph SOURCES["Recovery Sources"]
        S1["{services-machine}<br/>Syncthing copies<br/>(snapshots + framework)"]
        S2["Time Machine<br/>(encrypted portable drive)"]
        S3["Offsite drive<br/>(monthly rotation)"]
        S4["{Sync-OpenClaw-dir}/backups/<br/>on {PAI-machine}<br/>(openclaw backup archives)"]
        S5["npm registry<br/>(npm install -g openclaw)"]
        S6["{Sync-services-dir}<br/>on {PAI-machine}<br/>(docker compose + configs)"]
        S7["Tailscale<br/>(automatic failover)"]
    end

    subgraph TARGETS["RTO Targets"]
        R1["1-2h<br/>(with TM)<br/>2-4h (clean install)"]
        R2["30-60m<br/>(macOS install dominates)"]
        R3["30-60m<br/>(longer if Docker volumes)"]
        R4["0m<br/>(automatic failover)"]
    end

    F1 --> S1
    F1 --> S2
    F1 --> S3
    F1 --> R1

    F2 --> S4
    F2 --> S5
    F2 --> R2

    F3 --> S6
    F3 --> R3

    F4 --> S7
    F4 --> R4

    style F1 fill:#5B1A1A,stroke:#EF4444,color:#F9FAFB
    style F2 fill:#5B1A1A,stroke:#EF4444,color:#F9FAFB
    style F3 fill:#5B4A1A,stroke:#F59E0B,color:#F9FAFB
    style F4 fill:#374151,stroke:#9CA3AF,color:#F9FAFB

    style S1 fill:#1A3D1A,stroke:#22C55E,color:#F9FAFB
    style S2 fill:#1A3D1A,stroke:#22C55E,color:#F9FAFB
    style S3 fill:#1A3D1A,stroke:#22C55E,color:#F9FAFB
    style S4 fill:#1A3D1A,stroke:#22C55E,color:#F9FAFB
    style S5 fill:#1A3D1A,stroke:#22C55E,color:#F9FAFB
    style S6 fill:#1A3D1A,stroke:#22C55E,color:#F9FAFB
    style S7 fill:#1A3D1A,stroke:#22C55E,color:#F9FAFB

    style R1 fill:#1E40AF,stroke:#3B82F6,color:#F9FAFB
    style R2 fill:#1E40AF,stroke:#3B82F6,color:#F9FAFB
    style R3 fill:#1E40AF,stroke:#3B82F6,color:#F9FAFB
    style R4 fill:#1E40AF,stroke:#3B82F6,color:#F9FAFB
```

**Reading notes:**
- Each failure scenario has an explicit recovery source AND an RTO target. If a scenario doesn't have both, the runbook is incomplete.
- `{OpenClaw-machine}` recovery is fast (30-60m) because OpenClaw is a single npm package + a config + a workspace; the long pole is the macOS reinstall, not OpenClaw itself.
- Network failure (Scenario 4) has zero RTO because Tailscale's encrypted mesh provides automatic failover. The LAN-down scenario is the *least* disruptive thanks to this.
- The offsite drive (S3) is the last line of defense — it survives fires, floods, and theft of everything inside the home network. Test it at least once a year.
