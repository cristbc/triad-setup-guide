# OpenClaw Gateway — Trust Boundary

Embed in `07-OPENCLAW-APPENDIX.md` near the start of "Section 3: Gateway Service".

```mermaid
graph TB
    subgraph PAI_HOST["{PAI-machine}"]
        VERBUM["{PAI-agent}<br/>(Claude Code)"]
        CLI["{InterAgent-Tool}<br/>(channel CLI)"]
        SSH_CLIENT["OpenSSH client<br/>(Match exec resolves<br/>LAN→Tailscale)"]
        VERBUM --> CLI
        CLI --> SSH_CLIENT
    end

    subgraph TUNNEL["SSH transport (encrypted)"]
        T[/"Tailscale or LAN<br/>SSH session<br/>(authenticated by SSH key,<br/>restricted to LAN-subnet<br/>via authorized_keys)"/]
    end

    subgraph OC_HOST["{OpenClaw-machine} — agent user account"]
        SSHD["sshd<br/>(authorized_keys with<br/>from={LAN-subnet} restriction)"]
        LOOP["loopback only<br/>127.0.0.1:{gateway-port}"]
        GW["OpenClaw gateway<br/>(launchd service<br/>ai.openclaw.gateway)"]
        TOKEN["{gateway-token}<br/>(in ~/.openclaw/openclaw.json)"]
        AGENT["{OpenClaw-agent}<br/>main session<br/>(persistent context)"]

        SSHD --> LOOP
        LOOP --> GW
        GW -. "validates token<br/>on handshake" .- TOKEN
        GW --> AGENT
    end

    SSH_CLIENT ==>|"SSH connection"| T
    T ==>|"port 22"| SSHD

    EVIL["Anyone else<br/>(LAN attacker, internet,<br/>other user on host)"]
    EVIL -.->|"BLOCKED:<br/>no LAN port,<br/>no Tailscale port,<br/>no other user has<br/>the SSH key"| LOOP

    style VERBUM fill:#1E40AF,stroke:#3B82F6,color:#F9FAFB
    style CLI fill:#1E40AF,stroke:#3B82F6,color:#F9FAFB
    style SSH_CLIENT fill:#1F2937,stroke:#6B7280,color:#F9FAFB
    style T fill:#3D1A5B,stroke:#A855F7,color:#F9FAFB
    style SSHD fill:#1A3D1A,stroke:#22C55E,color:#F9FAFB
    style LOOP fill:#1A3D1A,stroke:#22C55E,color:#F9FAFB,stroke-width:3px
    style GW fill:#7C2D12,stroke:#F97316,color:#F9FAFB
    style TOKEN fill:#5B4A1A,stroke:#F59E0B,color:#F9FAFB
    style AGENT fill:#7C2D12,stroke:#F97316,color:#F9FAFB
    style EVIL fill:#5B1A1A,stroke:#EF4444,color:#F9FAFB,stroke-dasharray:5 5
```

**Reading notes:**
- **Two-factor security:** the SSH key authenticates the *transport*, the gateway token authenticates the *application*. Either one alone is insufficient.
- **The gateway never opens a LAN port.** It binds to `127.0.0.1:{gateway-port}` only — verifiable with `lsof -nP -iTCP:{gateway-port}` (you should see only loopback addresses, never `*` or a LAN IP).
- **The SSH key is restricted server-side** to `from={LAN-subnet}` in `authorized_keys`. Even if the key leaks, it only works from the local network. Tailscale traffic appears to come from the Tailscale CGNAT range — extend the `from=` clause if you use Tailscale paths.
- **Other macOS user accounts on the host cannot reach the gateway** unless they have the SSH key AND the gateway token AND can authenticate to the agent user's sshd. Filesystem permissions on `~/.openclaw/openclaw.json` (chmod 600) keep the token sealed inside the agent user.
- **Never expose the gateway port directly** via port forwarding, reverse proxies, or `localhost.run`-style tunnels. The loopback boundary is the entire security model — punching through it removes the SSH layer that makes everything else safe.
