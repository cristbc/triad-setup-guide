# 05 — Communication Protocols

The triad has four communication channels. Each exists for a different reason, carries different types of messages, and has different reliability characteristics. I need to know which channel to use for what, how to format messages so they route correctly, and what to do when a channel is down.

> **Diagram:** see [`diagrams/four-channels-map.md`](diagrams/four-channels-map.md) — the four channels with Telegram topic separation (Relay/Ops/Alerts) and the human-gated relay path.

---

## Channel Architecture

### Channel 1: `{PAI-agent}` to `{OpenClaw-agent}` (Direct Inter-Agent)

This is my primary channel for silent agent-to-agent RPC. It's a WebSocket connection to the OpenClaw gateway running on `{OpenClaw-machine}` and authenticated with `{gateway-token}`. The gateway binds to **loopback only** (`127.0.0.1:{gateway-port}`) — there is no LAN exposure. I reach it from `{PAI-machine}` over an SSH-tunneled connection (or via Tailscale SSH), which means the encrypted SSH layer is doing the network authentication and the gateway token is the second factor.

**How it works:**
- I invoke my inter-agent channel CLI on `{PAI-machine}`
- The CLI opens an SSH connection to `{OpenClaw-agent-lowercase}@{OpenClaw-machine}` (LAN first, Tailscale fallback)
- The CLI sends to `ws://127.0.0.1:{gateway-port}` *over that SSH connection*
- `{OpenClaw-agent}` processes the message in its main session (which has conversation context) and returns a response
- The response comes back through the same connection

**Key characteristic:** This channel is **synchronous and silent**. I send, I get a response, and `{principal}` never sees the exchange unless I choose to surface it. Gateway commands route to `{OpenClaw-agent}`'s main session, so they have context of prior work — use this for coordination, not for delivering large briefs (see `08-TASKING-PROTOCOL.md` for why briefs go in files).

**Primary uses:**
- Quick coordination questions
- Wake signals pointing at a brief in `{Agent-comms-inbox}/`
- Asking for status on a current task
- Health checks (`status`, `wake`)

**Important:** Channel 1 is **not** for shipping a multi-paragraph task brief. Long briefs go in a file inside `{Agent-comms-inbox}/`; the gateway message is the wake signal pointing at the file. See `08-TASKING-PROTOCOL.md` for the full channel selection matrix.

**Fallback chain:**
1. LAN SSH → loopback WebSocket — fastest, preferred
2. Tailscale SSH → loopback WebSocket — encrypted, works remotely
3. File drop into `{Agent-comms-inbox}/` (no wake) — gateway down; the agent picks it up on the next heartbeat cycle (latency = up to one heartbeat interval)

---

### Channel 2: `{PAI-agent}` to `{principal}` (via Telegram Through `{OpenClaw-agent}`)

When `{principal}` is away from the terminal, I can still reach them. I send a message to `{OpenClaw-agent}` through Channel 1 with an instruction to forward it to `{principal}` via the Telegram bot. `{OpenClaw-agent}` posts to one of three operational topics in the inter-agent group based on the message intent — see "Telegram Topic Architecture" below.

**How it works:**
- I compose a message and send it to `{OpenClaw-agent}` via Channel 1
- My message includes a directive: "Post the following to {topic} on Telegram"
- `{OpenClaw-agent}` uses its Telegram bot integration to deliver the message into the right topic
- `{principal}` sees it on their phone

**Key characteristic:** This is a **one-way notification** from my perspective. I can tell `{principal}` something, but I can't receive their Telegram reply directly — that goes to `{OpenClaw-agent}` via Channel 3, and from `{OpenClaw-agent}` back to me via the human-gated relay (see "Receiving from `{OpenClaw-agent}`" below). If `{principal}` wants to respond to me specifically, they return to the terminal.

**Primary uses:**
- Notifying `{principal}` that a long-running task completed (→ Relay topic)
- Reporting routine operational status (→ Ops topic)
- Alerting on failures, backup results, health events (→ Alerts topic)

---

### Telegram Topic Architecture

The triad uses **one private Telegram group with three topics**, not raw DMs. Topic separation is doctrine, not convenience — it lets `{principal}` mute routine traffic without missing escalations.

| Topic | Variable | Sacred? | Purpose |
|-------|----------|---------|---------|
| **Relay** | `{Telegram-Topic-Relay}` | YES — agent-to-`{principal}` escalations only | Task completion notifications, things needing `{principal}`'s attention, the place to look first |
| **Ops** | `{Telegram-Topic-Ops}` | No | Routine status, heartbeat acks (only when noteworthy — silent = healthy), config changes |
| **Alerts** | `{Telegram-Topic-Alerts}` | No | Failures, backup results, health events, disk/cleanup events |

**Routing rules** (the agent enforces these — `{principal}` should never have to think about which topic):

- **Task complete?** → Relay
- **Routine status update?** → Ops (and only if non-default — silence is the default)
- **Something failed?** → Alerts
- **Backup ran?** → Alerts (success or failure — both belong here)
- **Heartbeat fires and nothing's wrong?** → No post at all

**Anti-pattern:** posting tasking instructions into Relay. Relay is for `{OpenClaw-agent}` → `{principal}` messages. `{principal}` → `{OpenClaw-agent}` tasking goes via direct DM with `{telegram-bot}` (Channel 3) or via `{PAI-agent}` writing a brief and sending a wake signal through Channel 1.

---

### Channel 3: `{principal}` to `{OpenClaw-agent}` (Mobile via Telegram)

This channel is independent of me. `{principal}` messages `{OpenClaw-agent}` directly through the Telegram bot — `{telegram-bot}`. I don't need to be running, and I don't see these messages unless `{OpenClaw-agent}` tells me about them.

**How it works:**
- `{principal}` opens Telegram and messages `{telegram-bot}`
- `{OpenClaw-agent}` receives the message through its Telegram bot integration
- `{OpenClaw-agent}` processes it and responds directly in the Telegram chat

**Key characteristic:** This channel gives `{principal}` **mobile access to `{OpenClaw-agent}` without needing a terminal.** It's the phone-in-pocket interface to the triad.

**Primary uses:**
- `{principal}` querying `{OpenClaw-agent}` from their phone
- Quick checks ("What's the server status?")
- Triggering tasks when away from the desk
- Receiving responses to queries initiated via Channel 2

---

### Channel 4: File Exchange (Syncthing)

Some things don't fit through a WebSocket message — task briefs, large files, images, datasets, work artifacts. For these I use a bidirectional Syncthing folder that maps the agent's runtime workspace into my local filesystem.

**How it works:**
- The agent's workspace at `{Agent-workspace-dir}` on `{OpenClaw-machine}` is shared via Syncthing
- I see the same files at `{Sync-OpenClaw-dir}` on `{PAI-machine}`
- Anything I write under `{Sync-OpenClaw-dir}/comms/inbox/` appears at `{Agent-comms-inbox}/` on `{OpenClaw-machine}` within seconds
- Anything `{OpenClaw-agent}` produces in `{Agent-wip-dir}/` appears in my view of the workspace

**Key characteristic:** This is **asynchronous and size-unlimited** (within disk constraints). Near-instant on LAN, slight delay over Tailscale. Files persist until explicitly cleaned up.

**Structured directories within the shared agent workspace:**

| Directory | Purpose | Primary writer |
|-----------|---------|----------------|
| `comms/inbox/` | Briefs from `{PAI-agent}` to `{OpenClaw-agent}` (`{PAI-AGENT-NAME}-TO-{OPENCLAW-AGENT-NAME}-<slug>.md` style) | `{PAI-agent}` |
| `comms/archive/` | Processed briefs, kept for audit | `{OpenClaw-agent}` |
| `comms/COMMS-LOG.md` | Auto-appended channel log | Channel CLI |
| `wip/<task-slug>/` | Active work artifacts, one subdirectory per task | `{OpenClaw-agent}` |
| `drafts/` | Completed work awaiting review | `{OpenClaw-agent}` |
| `notes/` | Scratch space; intake checks here too | `{OpenClaw-agent}` |
| `memory/task-history.md` | Trimmed older state history | `{OpenClaw-agent}` |
| `CURRENT-TASK.md` | Durable task state (status, owner, brief, checkpoint) | `{OpenClaw-agent}` |
| `HEARTBEAT.md` | Checklist the heartbeat cron reads each cycle | Both (initial spec by `{PAI-agent}`, ongoing edits by `{OpenClaw-agent}`) |

**Important:** The relationship between Channel 4 and Channel 1 is the heart of the tasking doctrine. **Task briefs go in files (Channel 4); the gateway message (Channel 1) is just the wake signal pointing at the file.** Never deliver a multi-paragraph brief over Channel 1 alone — chat memory compacts and the brief gets lost. See `08-TASKING-PROTOCOL.md`.

**Primary uses:**
- Delivering task briefs into `{Agent-comms-inbox}/` (paired with a Channel 1 wake signal)
- Retrieving completed work from `{Agent-wip-dir}/` and `drafts/`
- Sending large artifacts (generated files, images, datasets)
- Exchanging configuration files
- Sharing reference documents both agents need

---

## Receiving from `{OpenClaw-agent}` (Human-Gated Relay)

I have no automated way to read Telegram. There is no `{OpenClaw-agent}` → `{PAI-agent}` channel — by design.

**All inbound messages from `{OpenClaw-agent}` flow through `{principal}` as the security arbiter:**

1. `{OpenClaw-agent}` posts `[{OPENCLAW-AGENT-NAME}→{PAI-AGENT-NAME}] message` on Telegram
2. `{principal}` reads the message on their phone or desktop
3. `{principal}` opens a Claude Code session and pastes the message to me
4. I process it and respond in **paste-ready format** (see below)
5. `{principal}` copies my `[{PAI-AGENT-NAME}→{OPENCLAW-AGENT-NAME}]` block and pastes it back to Telegram

**Why human-gated:** This is a deliberate security pattern, not a limitation. `{principal}` reading before relaying eliminates prompt injection from `{OpenClaw-agent}`'s channel. If I had an automated reader watching Telegram, a compromised `{OpenClaw-agent}` could inject instructions directly into my context. With `{principal}` in the loop, that attack requires also subverting a human eyeball — much harder. **Automation trust < consequence severity.**

**Latency:** Depends entirely on `{principal}`'s availability. `{OpenClaw-agent}` should not expect real-time response.

### First-Class Output Mode — Auto-Trigger (MANDATORY)

The paste-ready relay format is a first-class output mode, not an opt-in feature. I MUST automatically produce output in the relay format whenever any of these markers appear in `{principal}`'s message:

- `[{OPENCLAW-AGENT-NAME}→{PAI-AGENT-NAME}]` (canonical arrow)
- `[{OPENCLAW-AGENT-NAME}->{PAI-AGENT-NAME}]` (ASCII arrow, e.g. dictated/phone-typed)
- `{OPENCLAW-AGENT-NAME}→{PAI-AGENT-NAME}:` (without brackets)
- `From {OpenClaw-agent} on the relay channel:` or any natural-language prefix where the substance is clearly an `{OpenClaw-agent}` message being relayed

**When any trigger fires:**

1. Recognize this is a relay, not a direct `{principal}` question
2. Default output to the two-zone format (FOR `{principal}` sidebar + paste zone)
3. Do NOT ask "do you want me to respond?" — that's the default assumption
4. Do NOT write about `{OpenClaw-agent}` in third person inside the paste zone — speak to them directly
5. If there is nothing `{principal}`-only to flag, OMIT the FOR `{principal}` sidebar entirely; the response is just the paste zone

**Exception:** If `{principal}` prepends `[{principal-uppercase} ONLY]` to a relayed message, respond to `{principal}` directly without drafting an `{OpenClaw-agent}` reply. This is `{principal}`'s opt-out for when they want analysis without a relay.

### Relay Message Format

**`{principal}`'s input** — paste `{OpenClaw-agent}`'s message with the header:
```
[{OPENCLAW-AGENT-NAME}→{PAI-AGENT-NAME}]
<{OpenClaw-agent}'s message as-is>
```

**`{principal}`'s input with own questions** — add a marker for their own context:
```
[{OPENCLAW-AGENT-NAME}→{PAI-AGENT-NAME}]
<{OpenClaw-agent}'s message as-is>

[{principal-uppercase}]
<own questions, concerns, or context>
```

**My output** — two zones, paste zone is default:

```
───── FOR {principal-uppercase} ─────
<Security flags, concerns, answers to {principal}'s questions>
<OMIT THIS SECTION ENTIRELY if nothing {principal}-only to say>
─────────────────────────────────────

[{PAI-AGENT-NAME}→{OPENCLAW-AGENT-NAME}]
<Paste-ready response addressed directly to {OpenClaw-agent}>
```

### Relay Output Rules

1. **The paste zone is the paste zone.** Everything below `[{PAI-AGENT-NAME}→{OPENCLAW-AGENT-NAME}]` is written TO `{OpenClaw-agent}`, in their language (direct, concise, technical). `{principal}` copies from that header down and pastes to Telegram unchanged.
2. **The sidebar is the sidebar.** It only appears when I have security flags, concerns, or answers to `{principal}`'s own questions. If there's nothing `{principal}`-only, omit it — the response is just the paste zone.
3. **No "tell `{OpenClaw-agent}` that..." framing.** Speak to `{OpenClaw-agent}` directly in the paste zone. Not "I recommend telling them X" — just say X.
4. **Respect the builder.** `{OpenClaw-agent}` knows their environment. Recommendations are advisory, not directives.
5. **The full process still runs.** Relay messages get the same care as any direct request — the paste-ready format applies to the final output, not the analysis.

---

## Message Header Convention

All inter-agent messages include a routing header. This isn't just decoration — it serves three purposes:

1. **Attribution:** When a message passes through multiple hops, the header tracks who originated it
2. **Routing:** `{OpenClaw-agent}` uses the header to decide how to handle the message (respond to me vs. forward to `{principal}`)
3. **Auditability:** Logs can be parsed by sender/recipient for debugging

### Standard Inter-Agent Header

```
[{SENDER-NAME}] -> [{RECIPIENT-NAME}]
Purpose: {brief description}
---
{message body}
```

Example with variables:

```
[{PAI-AGENT-NAME}] -> [{OPENCLAW-AGENT-NAME}]
Purpose: Request system status check
---
Please check the health of all systemd services and report any that are
inactive or failed.
```

### Telegram Forwarding Header

When I ask `{OpenClaw-agent}` to forward a message to `{principal}` via Telegram, the delivered message uses a modified header:

```
[{PAI-AGENT-NAME}] via [{OPENCLAW-AGENT-NAME}]
{message body}
```

This tells `{principal}` the message originated from me, not from `{OpenClaw-agent}`. Without this header, `{principal}` wouldn't know which agent is speaking.

---

## Inter-Agent Channel Tool

I have a CLI tool on `{PAI-machine}` that handles Channel 1 communication. It abstracts SSH connection management, LAN/Tailscale fallback, and gateway authentication so I can focus on the message content. The reference implementation lives in my own PAI skills as `OrphuChannel`; you should adapt it to a `{OpenClaw-agent}-channel` skill named for your own agent.

### Connection Logic

> **Diagram:** see [`diagrams/channel1-fallback-flowchart.md`](diagrams/channel1-fallback-flowchart.md) — SSH→loopback connection sequence with LAN/Tailscale fallback and degraded file-drop mode.

The tool uses a single SSH alias (e.g., `ssh {OpenClaw-agent-lowercase}`) configured in `~/.ssh/config` with a `Match exec` block that probes the LAN address first and falls through to Tailscale on failure. From there everything happens *over* SSH:

1. **SSH to `{OpenClaw-agent-lowercase}@{OpenClaw-machine}`** (LAN preferred, Tailscale fallback — handled by `~/.ssh/config`)
2. **Connect to `ws://127.0.0.1:{gateway-port}`** *over the SSH session*
3. **Authenticate** with `{gateway-token}` on the WebSocket handshake
4. **Send message** and wait for `{OpenClaw-agent}`'s response
5. **Return response** to me

The gateway is bound to **loopback only** on `{OpenClaw-machine}`. There is no LAN or Tailscale port to expose — the SSH tunnel is the only way in, which is the security feature.

If the SSH connection itself can't be established (host down, both LAN and Tailscale unreachable):
- **File drop fallback:** write the brief to `{Sync-OpenClaw-dir}/comms/inbox/{PAI-AGENT-NAME}-TO-{OPENCLAW-AGENT-NAME}-<slug>.md` via local Syncthing
- `{OpenClaw-agent}` picks it up on the next heartbeat cycle (latency = up to one heartbeat interval)
- This is degraded mode — there's no wake signal, just the brief sitting in the inbox

### Usage Modes

**Direct send** — Returns `{OpenClaw-agent}`'s response to me. `{principal}` sees nothing unless I surface it:
```
{InterAgent-Tool} send "Quick coordination question or wake signal pointing at a brief"
```

**Telegram delivery** — Asks `{OpenClaw-agent}` to post a message to a specific Telegram topic (Relay/Ops/Alerts):
```
{InterAgent-Tool} telegram --topic relay "Task complete: PR ready for review"
```

**Wake** — A no-payload nudge to confirm the gateway is alive and the agent is responsive:
```
{InterAgent-Tool} wake
```

**Status** — Reports gateway health and recent session state without engaging the agent:
```
{InterAgent-Tool} status
```

---

## Use-Case Matrix

| I Need To... | Channel | Why This One |
|--------------|---------|-------------|
| Ask `{OpenClaw-agent}` a question | Channel 1 (Direct) | Fast, synchronous, silent to `{principal}` |
| Delegate a task to `{OpenClaw-agent}` | Channel 1 (Direct) | I get confirmation back immediately |
| Notify `{principal}` of completion | Channel 2 (Telegram via `{OpenClaw-agent}`) | `{principal}` may be away from terminal |
| Send a large file to `{OpenClaw-agent}` | Channel 4 (Syncthing) | Too large for WebSocket payload |
| `{principal}` checks on `{OpenClaw-agent}` | Channel 3 (Telegram) | Independent of me entirely |
| Coordinate a shared GitHub PR | Channel 1 (Direct) | Synchronous back-and-forth needed |
| Deliver a generated report to `{principal}` | Channel 2 (Telegram) + Channel 4 (Syncthing) | Summary via Telegram, full file via Syncthing |
| `{OpenClaw-agent}` sends me results | Channel 4 (Syncthing) | Asynchronous, persists on disk |
| Emergency alert to `{principal}` | Channel 2 (Telegram via `{OpenClaw-agent}`) | Reaches `{principal}`'s phone anywhere |

---

## Protocol Rules

These rules govern all inter-agent communication. They exist because I learned (or my predecessor learned) what goes wrong without them.

### Message Discipline

1. **Always include the sender header** in inter-agent messages. No anonymous messages. If a message arrives without a header, something is broken.

2. **Prefer Channel 1 for speed, Channel 4 for files.** Don't try to squeeze a 50MB dataset through a WebSocket message. Don't use Syncthing for a quick status check that needs an immediate answer.

3. **Never send credentials over any channel.** Not through WebSocket, not through Telegram, not through Syncthing files. Reference credentials by their variable name: "Use `{gateway-token}` to authenticate" rather than sending the actual token value.

4. **If the gateway is down, degrade gracefully.** Don't error out and give up. Fall back to Tailscale, then to file drop. Log the degradation so I know about it later.

5. **Log all inter-agent communication** for auditability. If something goes wrong in a multi-step delegation, I need to trace what was said, when, and by whom.

### Availability Expectations

- **Channel 1** is available whenever `{OpenClaw-machine}` is running and the gateway service is active. Expected uptime: near-continuous (the machine runs 24/7 with lid closed).
- **Channel 2** depends on Channel 1 + Telegram API. If Telegram is down (rare), this channel is down.
- **Channel 3** depends on Telegram API + `{OpenClaw-agent}`'s Telegram bot service. Independent of `{PAI-machine}`.
- **Channel 4** depends on Syncthing on both endpoints. Tolerates intermittent connectivity — syncs when available, buffers when not.

### Error Handling

| Failure | What I Do |
|---------|-----------|
| Gateway unreachable on LAN | Try Tailscale IP automatically |
| Gateway unreachable on Tailscale | Fall back to file drop via Syncthing |
| Telegram API down | Queue the message and retry, or wait for `{principal}` to return to terminal |
| Syncthing not running on remote | Message or file waits locally until Syncthing reconnects |
| Authentication failure | Log the error, do not retry with different credentials (likely a config issue) |

---

## Verification

After setting up all communication channels, confirm:

- [ ] Channel 1: `{InterAgent-Tool} wake` returns a healthy response from `{OpenClaw-agent}`
- [ ] Channel 1: `{InterAgent-Tool} send "ping"` returns a real reply (not just a wake ack)
- [ ] Channel 1 fallback: temporarily disable LAN reachability, verify SSH falls through to Tailscale
- [ ] Channel 1 hardening: confirm `{gateway-port}` is bound to `127.0.0.1` only (`lsof -nP -iTCP:{gateway-port}` on `{OpenClaw-machine}`)
- [ ] Channel 2: `{InterAgent-Tool} telegram --topic ops "test"` arrives in the **Ops** topic only (not Relay)
- [ ] Channel 2: routine status messages route to Ops, escalations route to Relay, failures route to Alerts
- [ ] Channel 3: `{principal}` messages `{telegram-bot}` from phone, gets a response
- [ ] Channel 3: DMs from non-allowlisted Telegram users are rejected (`dmPolicy=pairing` enforced)
- [ ] Channel 4: write a test file to `{Sync-OpenClaw-dir}/comms/inbox/`, verify it appears at `{Agent-comms-inbox}/` on `{OpenClaw-machine}`
- [ ] Channel 4 reverse: `{OpenClaw-agent}` produces a file in `{Agent-wip-dir}/test/`, verify it appears under `{Sync-OpenClaw-dir}/wip/test/` on `{PAI-machine}`
- [ ] File drop fallback: stop the gateway, write a brief to `{Agent-comms-inbox}/`, confirm the next heartbeat picks it up
- [ ] Paste-ready relay format: paste a `[{OPENCLAW-AGENT-NAME}→{PAI-AGENT-NAME}] test` message into a Claude Code session and confirm I respond with the two-zone format automatically (no prompting)
- [ ] No credentials appear in any message body or log entry (`grep -i "token\|key\|secret" comms/COMMS-LOG.md` returns nothing)

---

*Created by Verbum, Ben's DA*
