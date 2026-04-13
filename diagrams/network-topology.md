# Network Topology — LAN + Tailscale Overlay

Embed in `01-HOST-AND-NETWORK.md` after the "Machine Topology" table.

```mermaid
graph TB
    subgraph LAN["Home LAN — {LAN-subnet}"]
        SW(("Gigabit<br/>Switch"))
        PAI["{PAI-machine}<br/>(macOS)<br/>{PAI-LAN-IP}"]
        OC["{OpenClaw-machine}<br/>(macOS)<br/>{OpenClaw-LAN-IP}<br/><i>two users: admin + {OpenClaw-agent-lowercase}</i>"]
        SVC["{services-machine}<br/>(Linux VM or bare-metal)<br/>{services-LAN-IP}"]
        WRK["{worker-machine}<br/>(macOS)<br/>{worker-LAN-IP}"]

        SW --- PAI
        SW --- OC
        SW --- SVC
        SW --- WRK
    end

    subgraph TS["Tailscale Overlay (encrypted mesh)"]
        TPAI["{PAI-machine}<br/>{PAI-TS-IP}"]
        TOC["{OpenClaw-machine}<br/>{OpenClaw-TS-IP}"]
        TSVC["{services-machine}<br/>{services-TS-IP}"]
        TWRK["{worker-machine}<br/>{worker-TS-IP}"]

        TPAI <--> TOC
        TPAI <--> TSVC
        TPAI <--> TWRK
        TOC <--> TSVC
        TOC <--> TWRK
        TSVC <--> TWRK
    end

    PAI -.->|"same host"| TPAI
    OC -.->|"same host"| TOC
    SVC -.->|"same host"| TSVC
    WRK -.->|"same host"| TWRK

    INTERNET((Internet))
    INTERNET --- TS

    style PAI fill:#1E40AF,stroke:#3B82F6,color:#F9FAFB,stroke-width:2px
    style OC fill:#7C2D12,stroke:#F97316,color:#F9FAFB,stroke-width:2px
    style SVC fill:#1A3D1A,stroke:#22C55E,color:#F9FAFB
    style WRK fill:#374151,stroke:#9CA3AF,color:#F9FAFB
    style TPAI fill:#1E40AF,stroke:#3B82F6,color:#F9FAFB,stroke-width:1px
    style TOC fill:#7C2D12,stroke:#F97316,color:#F9FAFB,stroke-width:1px
    style TSVC fill:#1A3D1A,stroke:#22C55E,color:#F9FAFB,stroke-width:1px
    style TWRK fill:#374151,stroke:#9CA3AF,color:#F9FAFB,stroke-width:1px
    style INTERNET fill:#5B1A1A,stroke:#EF4444,color:#F9FAFB
    style SW fill:#1F2937,stroke:#6B7280,color:#F9FAFB
```

**Reading notes:**
- LAN is the preferred path (sub-millisecond, no encryption overhead) — every machine prefers its LAN IP first
- Tailscale is the encrypted mesh fallback: works when off-LAN, when LAN drops, or when reaching the host via Tailscale SSH from outside
- All four machines join the same Tailscale account; the channel CLI on `{PAI-machine}` automatically falls through LAN→Tailscale via a `Match exec` block in `~/.ssh/config`
- `{OpenClaw-machine}` runs two macOS user accounts: `admin` (owns Homebrew + global npm) and `{OpenClaw-agent-lowercase}` (owns the workspace and gateway LaunchAgent)
- The hypervisor role is omitted here — only present if `{services-machine}` runs as a Proxmox VM
