# Four Communication Channels — Map

Embed in `05-COMMUNICATION-PROTOCOLS.md` after the "Channel Architecture" section header.

```mermaid
graph LR
    PAI["{PAI-agent}<br/>(Claude Code on<br/>{PAI-machine})"]
    OC["{OpenClaw-agent}<br/>(OpenClaw on<br/>{OpenClaw-machine})"]
    PRIN["{principal}<br/>(human)"]

    subgraph C1["Channel 1 — Direct Inter-Agent (gateway via SSH→loopback)"]
        GW["Loopback gateway<br/>ws://127.0.0.1:{gateway-port}"]
    end

    subgraph C2["Channel 2 — Telegram (via {OpenClaw-agent})"]
        T_RELAY["Relay topic<br/>{Telegram-Topic-Relay}<br/><i>sacred</i>"]
        T_OPS["Ops topic<br/>{Telegram-Topic-Ops}"]
        T_ALERTS["Alerts topic<br/>{Telegram-Topic-Alerts}"]
    end

    subgraph C3["Channel 3 — Mobile DM"]
        DM["DM with<br/>{telegram-bot}<br/>(dmPolicy=pairing)"]
    end

    subgraph C4["Channel 4 — Syncthing File Exchange"]
        WS["{Agent-workspace-dir}<br/>(comms/inbox, wip, drafts...)"]
    end

    PAI <-->|"SSH tunnel"| GW
    GW <-->|"loopback"| OC

    PAI -->|"--topic relay"| T_RELAY
    PAI -->|"--topic ops"| T_OPS
    PAI -->|"--topic alerts"| T_ALERTS
    OC --> T_RELAY
    OC --> T_OPS
    OC --> T_ALERTS

    T_RELAY -.->|"phone read"| PRIN
    T_OPS -.->|"phone read"| PRIN
    T_ALERTS -.->|"phone read"| PRIN

    PRIN <-->|"DM"| DM
    DM <--> OC

    PAI <-->|"local Syncthing"| WS
    WS <-->|"remote Syncthing"| OC

    PRIN -.->|"human-gated relay<br/>[ORPHU→VERBUM]<br/>paste back to Claude Code"| PAI

    style PAI fill:#1E40AF,stroke:#3B82F6,color:#F9FAFB,stroke-width:2px
    style OC fill:#7C2D12,stroke:#F97316,color:#F9FAFB,stroke-width:2px
    style PRIN fill:#374151,stroke:#9CA3AF,color:#F9FAFB,stroke-width:2px
    style GW fill:#1A3D1A,stroke:#22C55E,color:#F9FAFB
    style WS fill:#1A3D1A,stroke:#22C55E,color:#F9FAFB
    style T_RELAY fill:#5B1A1A,stroke:#EF4444,color:#F9FAFB
    style T_OPS fill:#3D1A5B,stroke:#A855F7,color:#F9FAFB
    style T_ALERTS fill:#5B4A1A,stroke:#F59E0B,color:#F9FAFB
    style DM fill:#1E3A5F,stroke:#3B82F6,color:#F9FAFB
```

**Reading notes:**
- **Channel 1** is silent (`{principal}` doesn't see it) and synchronous. The gateway is loopback-only — every connection from `{PAI-machine}` is *over* an SSH tunnel into the agent user's session.
- **Channel 2** is `{PAI-agent}` → Telegram via `{OpenClaw-agent}` as a relay. Three topics, strict separation: Relay (escalations only, sacred), Ops (routine status, mostly silent), Alerts (failures + backup results).
- **Channel 3** is `{principal}` ↔ `{OpenClaw-agent}` directly. `dmPolicy=pairing` means only `{Principal-Telegram-User-ID}` can DM the bot.
- **Channel 4** is the Syncthing-mirrored agent workspace. Briefs go in `comms/inbox/`, work artifacts in `wip/`. This is the durable backbone of file-first tasking — see `08-TASKING-PROTOCOL.md`.
- The dotted line back from `{principal}` to `{PAI-agent}` is the **human-gated relay**: there's no automated `{OpenClaw-agent}` → `{PAI-agent}` path. `{principal}` reads, copies, pastes. Eliminates prompt injection from the Telegram side.
