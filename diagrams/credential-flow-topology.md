# Credential Flow & Storage Topology

Embed in `SECRETS-SETUP.md` after the "Storage Rules" section, before "Verification."

```mermaid
graph LR
    subgraph Generation
        K1["ssh-keygen<br/>(automation key)"]
        K2["ssh-keygen<br/>(GitHub key)"]
        K3["openssl rand<br/>(gateway token)"]
        K4["Tailscale<br/>(auto-generated)"]
        K5["Syncthing<br/>(auto-generated)"]
        K6["ElevenLabs<br/>(dashboard)"]
        K7["BotFather<br/>(Telegram)"]
    end

    subgraph Storage
        PM[("Password<br/>Manager<br/>(source of truth)")]
    end

    subgraph Consumption
        A1["{PAI-machine}<br/>~/.ssh/"]
        A2["Target machines<br/>authorized_keys"]
        A3["GitHub<br/>SSH Keys"]
        A4["{OpenClaw-machine}<br/>gateway config"]
        A5["{PAI-machine}<br/>channel tool"]
        A6["All machines<br/>Tailscale auth"]
        A7["All machines<br/>Syncthing pairing"]
        A8["Voice server<br/>.env"]
        A9["{OpenClaw-machine}<br/>bot config"]
    end

    K1 --> PM
    K2 --> PM
    K3 --> PM
    K4 --> PM
    K5 --> PM
    K6 --> PM
    K7 --> PM

    PM --> A1
    PM --> A2
    PM --> A3
    PM --> A4
    PM --> A5
    PM --> A6
    PM --> A7
    PM --> A8
    PM --> A9

    GIT["Git Repository"]
    PM -.-x GIT

    style PM fill:#1E40AF,stroke:#3B82F6,color:#F9FAFB,stroke-width:3px
    style GIT fill:#7F1D1D,stroke:#EF4444,color:#F9FAFB,stroke-dasharray:5 5
```

**Reading notes:**
- All credentials flow through the Password Manager (center hub)
- Each credential exists in exactly two places: the password manager and one consumption point
- The crossed-out path to "Git Repository" reinforces the prohibition
- Tailscale and Syncthing keys are auto-generated but still stored in the password manager for recovery
