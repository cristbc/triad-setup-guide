# Channel 1 — SSH→Loopback Fallback Sequence

Embed in `05-COMMUNICATION-PROTOCOLS.md` inside the "Inter-Agent Channel Tool" section.

```mermaid
flowchart TD
    START(["{InterAgent-Tool} send 'message'"]) --> SSH

    SSH{"SSH alias resolves<br/>via ~/.ssh/config<br/>Match exec block"}
    SSH -->|"LAN reachable<br/>(< 1s probe)"| LAN
    SSH -->|"LAN unreachable<br/>(timeout)"| TS

    LAN["SSH to<br/>{OpenClaw-agent-lowercase}@{OpenClaw-LAN-IP}"]
    TS["SSH to<br/>{OpenClaw-agent-lowercase}@{OpenClaw-TS-IP}<br/>(via Tailscale)"]

    LAN --> WS
    TS --> WS

    WS["Open WebSocket to<br/>ws://127.0.0.1:{gateway-port}<br/><i>over the SSH session</i>"]
    WS --> AUTH

    AUTH{"Authenticate<br/>with {gateway-token}"}
    AUTH -->|"OK"| SEND
    AUTH -->|"FAIL"| ERR1

    SEND["Send message to<br/>{OpenClaw-agent} main session"]
    SEND --> RESP

    RESP["{OpenClaw-agent} processes<br/>(has session context)"]
    RESP --> RETURN

    RETURN(["Response returned to<br/>{InterAgent-Tool}"]) --> DONE(["Done"])

    SSH -.->|"BOTH unreachable<br/>(host down)"| FILE
    FILE["Degraded mode:<br/>write brief to<br/>{Sync-OpenClaw-dir}/comms/inbox/<br/>via local Syncthing"]
    FILE --> WAIT
    WAIT["No wake signal possible.<br/>Heartbeat picks up the brief<br/>on next cycle (≤ {Heartbeat-Interval})"]
    WAIT --> DONE2(["Async done"])

    ERR1[["Auth failure<br/>(log + abort, do not retry)"]]

    style START fill:#1E40AF,stroke:#3B82F6,color:#F9FAFB
    style SSH fill:#374151,stroke:#9CA3AF,color:#F9FAFB
    style LAN fill:#1A3D1A,stroke:#22C55E,color:#F9FAFB
    style TS fill:#3D1A5B,stroke:#A855F7,color:#F9FAFB
    style WS fill:#1F2937,stroke:#6B7280,color:#F9FAFB
    style AUTH fill:#374151,stroke:#9CA3AF,color:#F9FAFB
    style SEND fill:#1E40AF,stroke:#3B82F6,color:#F9FAFB
    style RESP fill:#7C2D12,stroke:#F97316,color:#F9FAFB
    style RETURN fill:#1A3D1A,stroke:#22C55E,color:#F9FAFB
    style FILE fill:#5B4A1A,stroke:#F59E0B,color:#F9FAFB
    style WAIT fill:#5B4A1A,stroke:#F59E0B,color:#F9FAFB
    style ERR1 fill:#5B1A1A,stroke:#EF4444,color:#F9FAFB
```

**Reading notes:**
- The gateway is **loopback only** on `{OpenClaw-machine}` — there is no LAN or Tailscale port to expose. SSH is the entire transport.
- LAN is preferred (faster, no Tailscale relay overhead). The 1-second probe is a `Match exec` block in `~/.ssh/config` that connects via TCP to the LAN IP and falls through if it can't.
- Tailscale is the encrypted fallback for off-LAN scenarios (travel, ISP outage, LAN reconfig).
- If both transports fail, file-drop into `{Sync-OpenClaw-dir}/comms/inbox/` is the only path. Latency in this mode = up to one heartbeat cycle. There is no wake signal — the heartbeat picks up new briefs on its next intake scan.
- Auth failure (token mismatch) is **not retried**. It's almost always a config issue, not a transient problem; retrying with a wrong token wastes time.
