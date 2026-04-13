# Triad Entity Relationship and Delegation Flow

Embed in `06-MULTI-AGENT-COORDINATION.md` after "The Triad Pattern" section.

```mermaid
graph TB
    PRIN["{principal}<br/>(human)<br/><i>T0 — decision authority</i>"]

    subgraph PAI_HOST["{PAI-machine}"]
        VERBUM["{PAI-agent}<br/>(Claude Code)<br/><i>T1 — orchestrator</i>"]
        WS_LOCAL["{Sync-OpenClaw-dir}/<br/>(Syncthing local view)"]
    end

    subgraph OC_HOST["{OpenClaw-machine}"]
        ORPHU["{OpenClaw-agent}<br/>(OpenClaw daemon)<br/><i>T1 — autonomous worker</i>"]
        WS_REMOTE["{Agent-workspace-dir}/<br/>(real workspace)"]
        INBOX["comms/inbox/<br/>{PAI-AGENT-NAME}-TO-{OPENCLAW-AGENT-NAME}-&lt;slug&gt;.md"]
        WIP["wip/&lt;slug&gt;/<br/>(active artifacts)"]
        DRAFTS["drafts/<br/>(completed work)"]
        CT["CURRENT-TASK.md<br/>(slim, ~50 lines max)"]
        HB["HEARTBEAT.md<br/>(checklist, ~60 lines max)"]

        WS_REMOTE --- INBOX
        WS_REMOTE --- WIP
        WS_REMOTE --- DRAFTS
        WS_REMOTE --- CT
        WS_REMOTE --- HB
    end

    PRIN -->|"directs"| VERBUM
    PRIN -.->|"DM via {telegram-bot}<br/>(Channel 3, dmPolicy=pairing)"| ORPHU

    VERBUM ==>|"1. write brief<br/>(durable, file-first)"| INBOX
    VERBUM -->|"2. wake signal<br/>(ephemeral, gateway)"| ORPHU
    ORPHU -->|"3. read brief"| INBOX
    ORPHU -->|"4. update status"| CT
    ORPHU ==>|"5. produce artifacts"| WIP
    ORPHU -->|"6. archive brief<br/>after processing"| DRAFTS
    ORPHU -.->|"7. ack to Relay topic<br/>(Channel 2)"| PRIN

    WS_LOCAL <==>|"Syncthing<br/>(bidirectional)"| WS_REMOTE
    VERBUM ---|"reads via<br/>local view"| WS_LOCAL

    HEARTBEAT_FIRES{"Heartbeat<br/>fires every<br/>{Heartbeat-Interval}"}
    HEARTBEAT_FIRES -->|"reads"| HB
    HEARTBEAT_FIRES -->|"intake scan"| INBOX
    HEARTBEAT_FIRES -.->|"silent if<br/>nothing noteworthy"| ORPHU

    style PRIN fill:#374151,stroke:#9CA3AF,color:#F9FAFB,stroke-width:2px
    style VERBUM fill:#1E40AF,stroke:#3B82F6,color:#F9FAFB,stroke-width:2px
    style ORPHU fill:#7C2D12,stroke:#F97316,color:#F9FAFB,stroke-width:2px
    style INBOX fill:#5B4A1A,stroke:#F59E0B,color:#F9FAFB
    style WIP fill:#1A3D1A,stroke:#22C55E,color:#F9FAFB
    style DRAFTS fill:#1A3D1A,stroke:#22C55E,color:#F9FAFB
    style CT fill:#3D1A5B,stroke:#A855F7,color:#F9FAFB
    style HB fill:#3D1A5B,stroke:#A855F7,color:#F9FAFB
    style WS_LOCAL fill:#1F2937,stroke:#6B7280,color:#F9FAFB
    style WS_REMOTE fill:#1F2937,stroke:#6B7280,color:#F9FAFB
    style HEARTBEAT_FIRES fill:#5B1A1A,stroke:#EF4444,color:#F9FAFB
```

**Reading notes:**
- **Thick arrows = file-first tasking flow.** The brief lives in `comms/inbox/` (durable). The wake signal is ephemeral. The artifacts live in `wip/`. After processing, the brief is archived. This is the entire shape of a delegation.
- **`{OpenClaw-agent}`'s workspace appears in two places** — physically on `{OpenClaw-machine}` at `{Agent-workspace-dir}/`, and as a Syncthing-mirrored view at `{Sync-OpenClaw-dir}/` on `{PAI-machine}`. `{PAI-agent}` reads and writes through the local view; Syncthing handles the rest.
- **The heartbeat is for health, not tasking.** It reads `HEARTBEAT.md` (the checklist), scans `comms/inbox/` (intake), and posts to Ops only if something noteworthy happened. Silence is the default. Stalls are detected by comparing `last_checkpoint_utc` across cycles — see `08-TASKING-PROTOCOL.md`.
- **`{principal}` can DM `{OpenClaw-agent}` directly** without going through `{PAI-agent}`. That's Channel 3 (dotted line). `{principal}` is T0 and always has the option.
- The relationship between `{PAI-agent}` and `{OpenClaw-agent}` is **peer-to-peer with `{principal}` as the shared parent**, not a strict hierarchy. Either agent can initiate, propose, ask, or escalate.
