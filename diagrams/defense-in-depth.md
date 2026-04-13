# Defense in Depth — Concentric Security Layers

Embed in `03-SECURITY-MODEL.md` near the top, after the introduction.

```mermaid
graph TB
    subgraph L0["Outermost: Network Layer"]
        L0_DESC["LAN-only services (no public ports)<br/>Tailscale encrypted mesh for off-LAN access<br/>SSH key-based auth, restricted to LAN-subnet"]
    end

    subgraph L1["Host Layer"]
        L1_DESC["FileVault disk encryption<br/>macOS user account isolation<br/>(admin owns Homebrew, agent user owns workspace)<br/>Per-user authorized_keys"]
    end

    subgraph L2["Application Layer"]
        L2_DESC["OpenClaw gateway: loopback only (127.0.0.1)<br/>Gateway token authentication<br/>Telegram dmPolicy=pairing + groupPolicy=allowlist<br/>Channel auth-profiles.json chmod 600"]
    end

    subgraph L3["Agent Behavior Layer"]
        L3_DESC["AI Steering Rules (SYSTEM + USER)<br/>SecurityValidator hook (PreToolUse)<br/>BLOCK / CONFIRM / ALERT pattern matching<br/>Human-gated relay for inbound from Telegram"]
    end

    subgraph L4["Innermost: Principal Authority"]
        CORE["{principal}<br/>(decision authority,<br/>last line of defense)"]
    end

    L0 --> L1
    L1 --> L2
    L2 --> L3
    L3 --> L4

    style L0 fill:#1F2937,stroke:#6B7280,color:#F9FAFB
    style L1 fill:#1E3A5F,stroke:#3B82F6,color:#F9FAFB
    style L2 fill:#3D1A5B,stroke:#A855F7,color:#F9FAFB
    style L3 fill:#5B4A1A,stroke:#F59E0B,color:#F9FAFB
    style L4 fill:#5B1A1A,stroke:#EF4444,color:#F9FAFB
    style CORE fill:#374151,stroke:#9CA3AF,color:#F9FAFB,stroke-width:3px
```

**Reading notes:**
- Each layer is independently capable of stopping a class of attack. Defense-in-depth means an attacker has to defeat *every* layer to reach the principal.
- The innermost layer is `{principal}` — no matter how good the automated layers are, the human is the final arbiter on irreversible decisions (force pushes, credential reads, settings changes). The ASK pattern matching surfaces these to the principal explicitly.
- Layer 2 (Application) is where the OpenClaw-specific hardening lives: loopback gateway, `dmPolicy=pairing`, the bot allowlist. These are the doors a network attacker would try first.
- Layer 3 (Agent Behavior) is where prompt injection gets caught: human-gated relay means an agent can't be talked into doing something just by receiving a message.
