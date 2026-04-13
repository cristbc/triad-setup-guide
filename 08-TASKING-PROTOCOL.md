# 08 — Tasking Protocol

This is the operational doctrine that ties together the channels (`05`), the multi-agent coordination patterns (`06`), and the OpenClaw runtime (`07`). Without this doctrine you can have everything technically wired up and still hit multi-channel confusion, lost briefs, resume loops, and 3-hour silent stalls. Every rule in this file exists because the absence of it caused a real incident.

Read this last, but read this. The minimum viable triad runs on these principles:

---

## Design Principles (Non-Negotiable)

Every decision below derives from these. If you find yourself wanting to violate one, you're probably about to recreate an incident I've already lived through.

1. **File first, gateway second.** The task brief is a durable file. The gateway message is the wake-up signal pointing to that file. Chat memory is ephemeral — it survives session compaction, gateway restarts, and heartbeat cycles. A brief sent only as a gateway message gets lost. A brief on disk does not.

2. **One channel per intent.** Each communication need maps to exactly one channel. No "sometimes gateway, sometimes Telegram, sometimes file drop." Ambiguity caused a 3-hour resume loop on the reference triad. The Channel Selection Matrix below is definitive — there is no "or."

3. **Heartbeat is a health cycle, not a task dispatcher.** The heartbeat runs a periodic health check and scans the inbox so briefs that arrived during idle or degraded-mode windows don't sit indefinitely — but that intake scan is a *pickup* mechanism, not a dispatcher. `{PAI-agent}`'s wake signal is the primary dispatch path. The heartbeat is also not a progress driver and not a recovery mechanism for stale work. If your tasking latency depends on the heartbeat firing under normal conditions, the tasking is broken.

4. **CURRENT-TASK.md stays small.** Five fields plus a short append-only history. Not a narrative. Not a diagnosis. Not a conversation. As `{OpenClaw-agent}` himself put it: *"As a long-running narrative, it becomes the thing that breaks the thing."*

5. **Gateway commands route to the main session with continuity.** The gateway delivers messages to `{OpenClaw-agent}`'s persistent main session, which has conversation history. That means gateway messages have context of prior work — use this for coordination, not for delivering large briefs.

---

## Channel Selection Matrix (Definitive)

This table is the entire tasking doctrine in one view. When in doubt, return here.

| Intent | Channel | Command | Why this one |
|--------|---------|---------|-------------|
| **Deliver a new task brief** | File drop to `{Agent-comms-inbox}/` | `ssh {OpenClaw-agent-lowercase} "cat > {Agent-comms-inbox}/{PAI-AGENT-NAME}-TO-{OPENCLAW-AGENT-NAME}-<slug>.md << 'BRIEF' ... BRIEF"` | Durable, re-readable, survives compaction |
| **Wake `{OpenClaw-agent}` to process a brief** | Gateway CLI | `{InterAgent-Tool} send "New task brief at comms/inbox/{PAI-AGENT-NAME}-TO-{OPENCLAW-AGENT-NAME}-<slug>.md. Read it and begin."` | Immediate, routes to main session with context |
| **Quick coordination (no brief needed)** | Gateway CLI | `{InterAgent-Tool} send "<message>"` | Synchronous, returns response, main session context |
| **Check if `{OpenClaw-agent}` is alive** | Gateway CLI | `{InterAgent-Tool} status` or `{InterAgent-Tool} wake` | Health check, no side effects |
| **`{OpenClaw-agent}` reports task completion to `{principal}`** | Telegram Relay topic | `{OpenClaw-agent}` posts to `{Telegram-Topic-Relay}` | `{principal}` sees it on phone, sacred channel |
| **`{OpenClaw-agent}` reports routine status** | Telegram Ops topic | `{OpenClaw-agent}` posts to `{Telegram-Topic-Ops}` | Operational visibility, not urgent |
| **`{OpenClaw-agent}` reports a failure or alert** | Telegram Alerts topic | `{OpenClaw-agent}` posts to `{Telegram-Topic-Alerts}` | Failures need their own channel |
| **`{principal}` tasks `{OpenClaw-agent}` directly (mobile)** | Telegram DM with `{telegram-bot}` | Direct message | `{principal}` is principal; direct access always valid |
| **`{principal}` tasks `{OpenClaw-agent}` during a `{PAI-agent}` session** | `{PAI-agent}` writes a brief + sends a wake signal | See Scenario B below | Keeps provenance clean |
| **`{OpenClaw-agent}` sends a message to `{PAI-agent}`** | Human-gated relay via `{principal}` | `{OpenClaw-agent}` posts `[{OPENCLAW-AGENT-NAME}->{PAI-AGENT-NAME}]` to Telegram, `{principal}` relays | Security boundary; no automated AI-to-AI |
| **Gateway is down** | File drop only (no wake) | SSH write to `{Agent-comms-inbox}/`; heartbeat picks up on next cycle | Degraded mode; latency = next heartbeat cycle |

### Channels NOT to Use for Tasking

| Anti-pattern | Why it fails |
|--------------|-------------|
| Overwriting `CURRENT-TASK.md` directly to redirect `{OpenClaw-agent}` | Races with `{OpenClaw-agent}`'s own edits; caused a checkpoint collision incident |
| Sending the full task brief as a gateway message | Chat memory compacts; the brief gets lost after a few turns |
| Posting task instructions to the Telegram Relay topic | Relay is for `{OpenClaw-agent}` → `{principal}` escalations, not `{PAI-agent}` → `{OpenClaw-agent}` tasking |
| Writing to `{Agent-comms-inbox}/` without a gateway wake | `{OpenClaw-agent}` won't see it until the next heartbeat (up to one full interval delay) |
| Using the Ops topic for tasking | Ops is for status, not instructions |

---

## Scenario Protocols

### Scenario A: Tandem Collaboration

`{principal}` and `{PAI-agent}` are in a Claude Code session and need `{OpenClaw-agent}` to work on a related piece simultaneously.

**Protocol:**

1. **`{PAI-agent}` writes the brief:**

   ```bash
   ssh {OpenClaw-agent-lowercase} "cat > {Agent-comms-inbox}/{PAI-AGENT-NAME}-TO-{OPENCLAW-AGENT-NAME}-<slug>.md << 'BRIEF'
   # Task: <title>

   ## Context
   <What {PAI-agent} is working on, why this piece matters>

   ## Deliverables
   <Exact files/artifacts to produce, in {Agent-wip-dir}/<slug>/>

   ## Acceptance Criteria
   <Bullet list, testable>

   ## Coordination
   - {PAI-agent} is actively working on: <related piece>
   - Report completion to: Relay topic
   - If blocked, message {PAI-agent} via: [{OPENCLAW-AGENT-NAME}->{PAI-AGENT-NAME}] on Telegram

   ## Priority
   NORMAL | HIGH | URGENT
   BRIEF"
   ```

2. **`{PAI-agent}` wakes `{OpenClaw-agent}`:**

   ```bash
   {InterAgent-Tool} send "New tandem task brief at comms/inbox/{PAI-AGENT-NAME}-TO-{OPENCLAW-AGENT-NAME}-<slug>.md. \
     Read it and begin. I'm working on <related piece> in parallel. \
     Report progress to Ops, completion to Relay."
   ```

3. **`{OpenClaw-agent}` acknowledges** via the gateway response (`{PAI-agent}` sees it immediately).

4. **`{PAI-agent}` checks progress** if needed:

   ```bash
   {InterAgent-Tool} send "Status on <slug>?"
   ```

5. **`{OpenClaw-agent}` reports completion** to the Relay topic. `{principal}` relays to `{PAI-agent}` if the response needs further action.

6. **`{PAI-agent}` retrieves artifacts:**

   ```bash
   ssh {OpenClaw-agent-lowercase} "ls -la {Agent-wip-dir}/<slug>/"
   ssh {OpenClaw-agent-lowercase} "cat {Agent-wip-dir}/<slug>/<artifact>.md"
   ```

   Or via Syncthing at `{Sync-OpenClaw-dir}/wip/<slug>/` on `{PAI-machine}`.

---

### Scenario B: Independent Tasking

During a `{PAI-agent}` session, `{principal}` wants `{OpenClaw-agent}` to do something unrelated to `{PAI-agent}`'s current work.

**Two options depending on complexity:**

**Option 1 — Simple task (fits in a message):**

`{principal}` tells `{PAI-agent}`: "Ask `{OpenClaw-agent}` to <simple thing>."

`{PAI-agent}` sends via gateway:

```bash
{InterAgent-Tool} send "From {principal}: <task description>. This is independent of my current work."
```

**Option 2 — Complex task (needs a brief):**

`{PAI-agent}` writes a brief and wakes:

```bash
ssh {OpenClaw-agent-lowercase} "cat > {Agent-comms-inbox}/{PAI-AGENT-NAME}-TO-{OPENCLAW-AGENT-NAME}-<slug>.md << 'BRIEF'
# Task: <title>
## Origin: {principal} request during {PAI-agent} session (independent)
## Context
<details>
## Deliverables
<what to produce>
## Acceptance Criteria
<testable bullets>
## Priority
NORMAL
BRIEF"

{InterAgent-Tool} send "New independent task from {principal} at comms/inbox/{PAI-AGENT-NAME}-TO-{OPENCLAW-AGENT-NAME}-<slug>.md. Read and begin."
```

**What happens to `{OpenClaw-agent}`'s current task:**

- If `{OpenClaw-agent}` is `IDLE`: picks up immediately
- If `{OpenClaw-agent}` is `IN_PROGRESS`: reads the brief's priority field
  - **NORMAL:** brief stays in inbox, agent picks it up after the current task completes
  - **HIGH:** agent pauses the current task (sets a `## Paused Task` block in CURRENT-TASK.md), starts the new one
  - **URGENT:** same as HIGH, plus the agent posts to Relay acknowledging the interrupt

**Should `{principal}` just DM `{telegram-bot}` directly?** Yes, for quick things. `{principal}` is T0 and can always DM directly. The protocol above is for when `{principal}` wants `{PAI-agent}` to compose the task properly — with context, acceptance criteria, and provenance.

---

### Scenario C: Heartbeat Fires While `{OpenClaw-agent}` is Idle

The heartbeat cron fires on its `{Heartbeat-Interval}` cadence and `{OpenClaw-agent}` has no active task.

**What the heartbeat does (in order):**

1. **Read HEARTBEAT.md** — follow its checklist strictly
2. **Check CURRENT-TASK.md status field:**
   - `IDLE` → proceed to intake
   - `IN_PROGRESS` → see resume logic in Scenario E below
   - `COMPLETE` / `REVIEW` → do not pick up new work; skip to health checks
   - `BLOCKED` → apply backoff escalation (1h, 4h, 24h)
3. **Intake scan:** Check `{Agent-comms-inbox}/` for new `{PAI-AGENT-NAME}-TO-{OPENCLAW-AGENT-NAME}-*.md` files
   - If found: read the highest-priority brief, set CURRENT-TASK.md to `IN_PROGRESS`, begin work
   - Multiple briefs: process by priority (URGENT > HIGH > NORMAL), then by file timestamp (oldest first within a priority)
   - After processing: move the brief from `{Agent-comms-inbox}/` to `{Agent-comms-archive}/` with a date prefix
4. **Health checks:** Verify Syncthing is healthy, output directories exist, disk space is adequate
5. **Post heartbeat ack to Ops topic** — only if something noteworthy happened. **Silent = healthy.**

**What the heartbeat does NOT do:**

- Drive task progress. If `{OpenClaw-agent}` is `IN_PROGRESS`, the heartbeat checks for stalls but does not try to advance the work.
- Pick up work when status is `REVIEW` or `COMPLETE`
- Post to Relay (the sacred topic). The heartbeat only escalates to Relay when a `BLOCKED` task crosses the 1-hour threshold.

---

### Scenario D: Priority Interrupt

`{OpenClaw-agent}` is working on Task A and a higher-priority Task B arrives.

**Who can interrupt:**

- **`{principal}` (T0):** Can interrupt anything, any time, via any channel
- **`{PAI-agent}` (T1):** Can interrupt via gateway + brief with HIGH or URGENT priority
- **The heartbeat:** Cannot interrupt. It only detects stalls.

**Protocol:**

1. **`{PAI-agent}` writes the interrupt brief:**

   ```bash
   ssh {OpenClaw-agent-lowercase} "cat > {Agent-comms-inbox}/{PAI-AGENT-NAME}-TO-{OPENCLAW-AGENT-NAME}-<slug>.md << 'BRIEF'
   # PRIORITY INTERRUPT: <title>
   ## Priority: HIGH | URGENT
   ## Interrupts: <current task name>
   ## Context
   <why this can't wait>
   ## Deliverables
   <what to produce>
   ## Acceptance Criteria
   <testable bullets>
   ## After completion
   Resume interrupted task: <task name>
   BRIEF"
   ```

2. **`{PAI-agent}` sends the interrupt signal:**

   ```bash
   {InterAgent-Tool} send "PRIORITY INTERRUPT. New brief at comms/inbox/{PAI-AGENT-NAME}-TO-{OPENCLAW-AGENT-NAME}-<slug>.md. \
     Pause current work, read the brief, and begin immediately."
   ```

3. **`{OpenClaw-agent}`'s response:**

   - Copies the current task's fields into a `## Paused Task` block in CURRENT-TASK.md, recording what was interrupted and where it left off
   - Reads the interrupt brief
   - Overwrites the `## Active Task` block with the interrupt task's fields (status: `IN_PROGRESS`, new brief, fresh checkpoint)
   - Works the interrupt to completion
   - On completion: posts to Relay, then checks for a `## Paused Task` block
   - If a `## Paused Task` block exists: moves its fields back into `## Active Task` (status: `IN_PROGRESS`), clears the `## Paused Task` section, resumes

4. **What happens to the interrupted task:**

   - State preserved in CURRENT-TASK.md under a `## Paused Task` section (not a status change — the active-task `status` field always describes the currently running task, which during the interrupt is the interrupt itself)
   - Work artifacts remain in `{Agent-wip-dir}/<original-slug>/`
   - `{OpenClaw-agent}` resumes automatically after the interrupt completes
   - If `{principal}` cancels the original task during the interrupt, `{PAI-agent}` sends: "Cancel paused task `<name>`. Do not resume."

---

### Scenario E: Resume Loop Detection

`{OpenClaw-agent}` is `IN_PROGRESS` on a task and the heartbeat fires.

The heartbeat compares `last_checkpoint_utc` in CURRENT-TASK.md against the previous heartbeat's value:

| Observation | Action |
|-------------|--------|
| **Checkpoint changed** | Agent is making progress. Brief ack to Ops. Done. |
| **Unchanged for 1 heartbeat (60 min)** | Warning state. Post to Ops: "Progress stalled on `<task>`, investigating." |
| **Unchanged for 2+ heartbeats (120+ min)** | **Resume loop confirmed.** Heartbeat MUST either: (a) produce at least one new artifact in `{Agent-wip-dir}/` before exiting, OR (b) escalate to Relay: "Resume loop detected on `<task>`. Need intervention." |

**The heartbeat MUST NOT post "Recovered from restart" or "Resuming `<task>`" without producing output.** That message *is* the resume loop. This rule exists because the reference triad spent 3 hours posting "Resuming LLM Knowledge Base foundation" every 60 minutes while producing zero artifacts. The fix is hard-coded in HEARTBEAT.md as a gate.

---

## File Ownership Table

### CURRENT-TASK.md

| Aspect | Rule |
|--------|------|
| Location | `{Agent-workspace-dir}/CURRENT-TASK.md` |
| Primary writer | `{OpenClaw-agent}` |
| Who else can write | `{PAI-agent}` — ONLY to break a confirmed resume loop (last resort, not routine) |
| When `{PAI-agent}` writes | Only after confirming via gateway that `{OpenClaw-agent}` is looping and cannot self-correct |
| Readers | `{OpenClaw-agent}` (every heartbeat + every work session), `{PAI-agent}` (via SSH for status checks) |
| Max size | ~50 lines. If it exceeds this, `{OpenClaw-agent}` trims `batch_progress` to last 3 entries |
| Required fields | `status`, `owner`, `assigned_by`, `brief` (1 line), `last_checkpoint_utc`, `next_action`, `artifacts_path` |
| Optional fields | `batch_progress` (3 entries max), `paused_task` section |
| State history | Append-only, but limited to last 10 entries. Older entries archived to `memory/task-history.md` |

**Field definitions:**

```markdown
## Active Task
**status:** IDLE | IN_PROGRESS | BLOCKED | REVIEW | COMPLETE | CANCELLED
**owner:** {OpenClaw-agent-lowercase}
**assigned_by:** {principal} | {PAI-agent} | self
**brief:** <one-line task description>
**last_checkpoint_utc:** <ISO 8601>
**next_action:** <what to do next, one line>
**artifacts_path:** {Agent-wip-dir}/<slug>/

## Paused Task (only present during a priority interrupt)
**brief:** <one-line description of interrupted task>
**resume_after:** <interrupt task slug>
**wip_path:** {Agent-wip-dir}/<original-slug>/
```

**Note on interrupt state:** During a priority interrupt, the `status` field always describes the *currently active* task (the interrupt itself, which is `IN_PROGRESS`). The interrupted task is captured in a separate `## Paused Task` block so its state survives. The status enum deliberately does not include a "PAUSED" value — there is always exactly one active task, and status reflects that one.

### HEARTBEAT.md

| Aspect | Rule |
|--------|------|
| Location | `{Agent-workspace-dir}/HEARTBEAT.md` |
| Primary writer | `{PAI-agent}` (protocol design) and `{OpenClaw-agent}` (operational updates) |
| Readers | `{OpenClaw-agent}` (every heartbeat cycle) |
| Purpose | The checklist the heartbeat cron reads and follows |
| Max size | ~60 lines. This is a checklist, not a knowledge base. |
| What it contains | Task state check rules, intake check steps, health check steps, posting guidelines |
| What it does NOT contain | Task details, briefs, progress narratives, diagnoses |

### `{Agent-comms-inbox}/`

| Aspect | Rule |
|--------|------|
| Location | `{Agent-comms-inbox}/` on `{OpenClaw-machine}` (Syncthing-synced to `{Sync-OpenClaw-dir}/comms/inbox/` on `{PAI-machine}`) |
| Writers | `{PAI-agent}` (via SSH or Syncthing), `{principal}` (via Telegram instructions to `{OpenClaw-agent}`) |
| Reader | `{OpenClaw-agent}` (during heartbeat intake + on gateway wake command) |
| Naming convention | `{PAI-AGENT-NAME}-TO-{OPENCLAW-AGENT-NAME}-<kebab-slug>.md` (adapt the prefix to your `{PAI-agent}` and `{OpenClaw-agent}` names) |
| Lifecycle | Write → `{OpenClaw-agent}` reads → `{OpenClaw-agent}` moves to `{Agent-comms-archive}/<YYYYMMDD>-<slug>.md` |
| Cleanup | `{OpenClaw-agent}` archives after processing. Files older than 7 days in archive can be deleted. |
| Queue behavior | Multiple briefs can coexist. Priority field in each brief determines processing order. |
| Anti-pattern | Do NOT leave processed briefs in inbox. They confuse the next heartbeat intake. |

---

## Anti-Patterns and Their Fixes

### Anti-Pattern 1: The Resume Loop

**What happened:** `{OpenClaw-agent}`'s heartbeat fired every 60 minutes, saw `IN_PROGRESS` in CURRENT-TASK.md, posted "Recovered from restart. Resuming `<task>`" — and produced zero new output. For 3+ hours.

**Root causes:**

1. CURRENT-TASK.md was acting as both task state AND heartbeat recovery instructions
2. `{OpenClaw-agent}` tried to edit CURRENT-TASK.md with a stale `old_string` from a prior heartbeat's checkpoint timestamp; the edit failed silently, so the checkpoint never advanced
3. The heartbeat had no hard gate against posting recovery messages without producing artifacts

**Fix (built into this protocol):**

- Heartbeat detects resume loops by comparing `last_checkpoint_utc` across heartbeats
- After 2 unchanged heartbeats: must produce output OR escalate. No third "resuming" message allowed.
- CURRENT-TASK.md stays small enough that edit collisions are unlikely (fewer fields = fewer race targets)
- `{PAI-agent}` overwrites CURRENT-TASK.md only as a last resort, not as routine tasking

### Anti-Pattern 2: Chat-Only Task Briefs

**What happened:** `{PAI-agent}` sent a detailed task as a gateway message. After several turns of `{OpenClaw-agent}` working, session compaction dropped the original brief. `{OpenClaw-agent}` lost context of what was requested.

**Fix:** All substantive tasks get a file brief in `{Agent-comms-inbox}/`. The gateway message just points to the file. `{OpenClaw-agent}` can re-read the file at any time.

### Anti-Pattern 3: Multi-Channel Confusion

**What happened:** `{PAI-agent}` sometimes wrote to `{Agent-comms-inbox}/` and hoped the heartbeat would pick it up. Sometimes sent via gateway. Sometimes overwrote CURRENT-TASK.md. `{OpenClaw-agent}` couldn't predict where instructions would arrive.

**Fix:** This protocol defines exactly one channel per intent (see Channel Selection Matrix above). No ambiguity.

### Anti-Pattern 4: Stale Inbox Accumulation

**What happened:** 21 files sitting in `{Agent-comms-inbox}/` over weeks, some weeks old. Many were processed but never archived. Some arrived while `{OpenClaw-agent}` was busy and were forgotten.

**Fix:**

- `{OpenClaw-agent}` archives briefs after processing (move to `{Agent-comms-archive}/`)
- Heartbeat intake scans inbox every cycle when `IDLE`
- `{PAI-agent}` does NOT drop briefs silently — always pairs a file drop with a gateway wake

### Anti-Pattern 5: CURRENT-TASK.md as a Novel

**What happened:** CURRENT-TASK.md grew to 100+ lines with full diagnosis, evidence, state history going back weeks, and multiple nested sections. The heartbeat had to parse all of it every cycle, and edits to specific fields became fragile.

**Fix:**

- Max 50 lines. Hard limit.
- State history: last 10 entries only. Older entries go to `memory/task-history.md`.
- `batch_progress`: last 3 entries only.
- No diagnosis or evidence sections. Those belong in `{Agent-wip-dir}/` artifacts.

---

## Quick Reference Card

### `{PAI-agent}` Tasking `{OpenClaw-agent}` (Standard Flow)

```bash
# 1. Write brief
ssh {OpenClaw-agent-lowercase} "cat > {Agent-comms-inbox}/{PAI-AGENT-NAME}-TO-{OPENCLAW-AGENT-NAME}-<slug>.md << 'BRIEF'
<brief content>
BRIEF"

# 2. Wake
{InterAgent-Tool} send \
  "New task brief at comms/inbox/{PAI-AGENT-NAME}-TO-{OPENCLAW-AGENT-NAME}-<slug>.md. Read and begin."

# 3. Check progress (optional)
{InterAgent-Tool} send "Status on <slug>?"

# 4. Retrieve artifacts
ssh {OpenClaw-agent-lowercase} "cat {Agent-wip-dir}/<slug>/<artifact>"
# Or via Syncthing: {Sync-OpenClaw-dir}/wip/<slug>/
```

### `{PAI-agent}` Quick Command (No Brief Needed)

```bash
{InterAgent-Tool} send "<simple request>"
```

### Priority Interrupt

```bash
# 1. Write interrupt brief (with "PRIORITY INTERRUPT" header)
ssh {OpenClaw-agent-lowercase} "cat > {Agent-comms-inbox}/{PAI-AGENT-NAME}-TO-{OPENCLAW-AGENT-NAME}-<slug>.md << 'BRIEF'
# PRIORITY INTERRUPT: <title>
...
BRIEF"

# 2. Send interrupt signal
{InterAgent-Tool} send \
  "PRIORITY INTERRUPT. Brief at comms/inbox/{PAI-AGENT-NAME}-TO-{OPENCLAW-AGENT-NAME}-<slug>.md. Pause current work and begin."
```

### Check `{OpenClaw-agent}` Health

```bash
{InterAgent-Tool} status
{InterAgent-Tool} wake
```

---

## Verification

After standing up the tasking protocol on a fresh triad:

- [ ] `{Agent-comms-inbox}/`, `{Agent-comms-archive}/`, `{Agent-wip-dir}/`, `drafts/`, `notes/`, `memory/` exist under `{Agent-workspace-dir}/`
- [ ] `CURRENT-TASK.md` exists with required fields and status `IDLE`
- [ ] `HEARTBEAT.md` exists, is ~60 lines or less, and reads as a checklist
- [ ] Heartbeat trigger configured in `~/.openclaw/openclaw.json` under `agents.defaults.heartbeat`
- [ ] One heartbeat cycle completes without errors
- [ ] A NORMAL brief flows end-to-end: write → wake → read → work → archive
- [ ] A HIGH-priority interrupt pauses an in-progress task and resumes correctly
- [ ] A simulated stall is caught by resume-loop detection within 2 cycles
- [ ] The paste-ready relay format triggers automatically when `{principal}` pastes a `[{OPENCLAW-AGENT-NAME}->{PAI-AGENT-NAME}]` message
- [ ] An external dead-man's-switch alerts on missing heartbeat acks (silent failure mitigation)

---

*Created by Verbum, Ben's DA. Doctrine adapted from real incidents. Every rule earned the hard way.*
