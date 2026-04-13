# 06 — Multi-Agent Coordination

This is where the triad comes alive. The machines are connected (01), I'm configured (02), security is in place (03), backups are running (04), and communication channels are open (05). Now I describe how `{PAI-agent}` and `{OpenClaw-agent}` actually work together under `{principal}`'s direction.

---

## The Triad Pattern

The triad is three entities — `{principal}` (human), `{PAI-agent}` (Claude Code DA), `{OpenClaw-agent}` (OpenClaw/GPT agent) — working as a coordinated unit. Each has distinct strengths:

| Entity | Strengths | Primary Role |
|--------|-----------|-------------|
| `{principal}` | Intent, judgment, approval authority | Directs work, makes decisions |
| `{PAI-agent}` | Deep codebase work, multi-agent orchestration, PAI infrastructure | Primary implementer, orchestrator |
| `{OpenClaw-agent}` | Persistent daemon, scheduled tasks, independent mobile access, different model perspective | Autonomous worker, second opinion, monitoring |

The triad isn't hierarchical in a strict sense — `{principal}` has authority, but both agents can initiate work, ask each other questions, and propose actions. The pattern is closer to a team with a lead than a command chain.

> **Diagram:** see [`diagrams/triad-delegation-flow.md`](diagrams/triad-delegation-flow.md) — embedded Mermaid showing entity relationships and the file-first delegation flow.

---

## Scaling Beyond Two Agents

The triad pattern is a starting point, not a ceiling. Additional agents can join the system on new machines or as separate user accounts on shared hardware.

Each new agent needs three things:

| Requirement | Purpose | Example |
|-------------|---------|---------|
| Machine or user account | Compute and filesystem isolation | New macOS user on an existing Mac Mini |
| Communication channel | How `{PAI-agent}` reaches the new agent | A dedicated channel skill (modeled on the inter-agent channel tool) |
| Identity documents | Who the agent is and how it behaves | Soul docs in a synced directory |

The coordination anti-patterns from the table below apply regardless of agent count — in fact, they become more important as the fleet grows. With 4+ agents, explicit ownership of files and areas is essential to prevent conflicts.

**Naming each agent's channel skill** (e.g., `{agent-name}-channel`) keeps tool invocation unambiguous. Each channel skill encapsulates connection details, fallback logic, and message formatting for that specific agent.

---

## Delegation Patterns

### {PAI-agent} → {OpenClaw-agent}

When I need `{OpenClaw-agent}` to do something, the protocol is **file first, gateway second**: I write a brief into `{Agent-comms-inbox}/` and then send a wake signal via Channel 1 pointing at the file. The full doctrine — including the channel selection matrix, anti-patterns, and resume-loop detection — lives in `08-TASKING-PROTOCOL.md`. The summary:

**Tandem task brief (`{PAI-agent}` + `{OpenClaw-agent}` working in parallel):**

```bash
# 1. Write the brief (durable, survives compaction)
ssh {OpenClaw-agent-lowercase} "cat > {Agent-comms-inbox}/{PAI-AGENT-NAME}-TO-{OPENCLAW-AGENT-NAME}-<slug>.md << 'BRIEF'
# Task: <title>

## Context
<What I'm working on, why their piece matters>

## Deliverables
<Exact files/artifacts to produce, in {Agent-wip-dir}/<slug>/>

## Acceptance Criteria
<Bullet list, testable>

## Priority
NORMAL | HIGH | URGENT
BRIEF"

# 2. Send the wake signal (ephemeral, just points at the file)
{InterAgent-Tool} send "New task brief at comms/inbox/{PAI-AGENT-NAME}-TO-{OPENCLAW-AGENT-NAME}-<slug>.md. Read and begin."
```

**Why this shape:** Chat memory inside `{OpenClaw-agent}`'s session compacts. A brief sent only as a gateway message gets lost after a few turns. A brief sitting on disk survives compaction, gateway restarts, and heartbeat cycles. The wake signal is the immediate-attention nudge; the file is the source of truth.

**Common delegation scenarios:**
- Research tasks requiring a different model perspective
- Monitoring/scheduled tasks that need to run while I'm not active
- Work on `{OpenClaw-agent}`'s own codebase or configuration
- Tasks that benefit from `{OpenClaw-agent}`'s persistent daemon nature

**I don't delegate:**
- Tasks requiring deep PAI codebase access (that's my domain)
- Security-sensitive operations on `{PAI-machine}`
- Tasks where `{principal}` expects me to do the work personally

### {principal} → Both Agents

`{principal}` can direct either agent independently:
- Terminal on `{PAI-machine}` → talks to me directly
- Telegram → talks to `{OpenClaw-agent}` directly
- Can ask either agent to coordinate with the other

### Parallel Delegation to Worker

For compute-heavy tasks, I delegate to `{worker-machine}` using the parallel worker system:

1. **Create job file** in `{Sync-worker-dir}/jobs/`:
```json
{
  "task_id": "unique-id",
  "prompt": "The task description",
  "backend": "claude|codex|gemini",
  "model": "model-name",
  "timeout": 300
}
```

2. **Job syncs** to `{worker-machine}` via Syncthing
3. **Worker script** picks up the job, executes it, writes output
4. **Output syncs** back to `{Sync-worker-dir}/output/{task_id}.output.txt`

**Key principle:** API keys are forwarded via SSH environment variables at execution time — never stored on disk on the worker.

### Internal Parallel Agents

Within my own Claude Code session, I use the Task tool to spawn parallel subagents:

- **Explore agents** for codebase research
- **Engineer agents** for implementation work
- **Intern agents** for general-purpose tasks
- **Researcher agents** for web research (multiple engines in parallel)
- **Spotcheck agent** after parallel work to verify consistency

Model selection matters: haiku for grunt work (10-20x faster), sonnet for standard implementation, opus for deep reasoning.

---

## GitHub Collaboration

When both agents need to work on the same codebase, I use GitHub as the coordination layer:

### Repository Setup
- Private repos under `{github-username}/`
- Both `{PAI-machine}` and `{OpenClaw-machine}` have SSH keys authorized on the GitHub account
- Both machines can push and pull

### Branch Strategy
- `main` — stable, reviewed code
- `feature/*` — work branches (either agent can create these)
- PRs required for merging to main

### Workflow
1. `{principal}` assigns work to an agent
2. Agent creates `feature/description` branch
3. Agent does work, commits, pushes
4. Agent creates PR (or notifies `{principal}`)
5. Other agent or `{principal}` reviews via `gh pr diff`
6. Merge after approval

### Conflict Resolution
- If both agents are working on the same repo, coordinate via inter-agent channel first
- Agree on who owns which files/areas
- Use small, focused PRs to minimize conflicts

---

## Worker System Architecture

The `{worker-machine}` extends compute capacity beyond `{PAI-machine}`. It's not an agent — it's a compute node that executes jobs I dispatch.

### Components

**On `{PAI-machine}` (dispatcher):**
- Job creation script: assembles JSON job files
- Syncthing folder `{Sync-worker-dir}` syncs jobs and retrieves output
- SSH access for direct command execution when Syncthing is too slow

**On `{worker-machine}` (executor):**
- Worker launch script: monitors for new jobs, executes them
- Multiple CLI backends available (Claude, Codex, Gemini)
- Output written to shared folder for sync back

### Job Lifecycle
```
Create job JSON → Sync to worker → Worker detects job →
Execute with specified backend → Write output → Sync output back →
PAI agent reads result
```

### Alternative: Direct Execution

Claude Code's native Agent tool and SSH-based direct command execution have emerged as alternatives to the Syncthing job dispatch pattern. For tasks where the result is needed immediately, SSH execution is faster:

```bash
ssh {worker-machine} "claude --print 'Summarize this file' < /path/to/file"
```

The Syncthing-based pattern remains valuable for offline or asynchronous workloads where the worker may not be immediately reachable.

### When to Use the Worker
- Parallel research across multiple AI backends
- Compute-intensive tasks that would block my session
- Tasks that benefit from a different model (Gemini for web search, Codex for code generation)
- Batch processing (multiple jobs dispatched simultaneously)

---

## Naming Philosophy

Names matter in the triad. `{PAI-agent}` and `{OpenClaw-agent}` aren't generic assistants — they're named entities with persistent identities:

- **Names create accountability** — "{PAI-agent} reviewed this" vs "an AI reviewed this"
- **Names enable coordination** — message headers identify sender and recipient
- **Names carry philosophy** — `{principal}` chose these names deliberately (see DAIDENTITY.md for the reasoning behind your DA's name)
- **Soul documents** define each agent's identity, values, and behavioral parameters

`{OpenClaw-agent}`'s soul documents live in `{soul-docs-dir}/` on `{OpenClaw-machine}` and sync to `{Sync-OpenClaw-dir}/{soul-docs-dir}/` on `{PAI-machine}`. This means I can read (and help maintain) `{OpenClaw-agent}`'s identity configuration.

---

## Coordination Anti-Patterns

Things I've learned not to do (the long form, with fixes, lives in `08-TASKING-PROTOCOL.md`):

| Don't | Why | Instead |
|-------|-----|---------|
| Both agents editing same file simultaneously | Merge conflicts, lost work | Coordinate via channel first |
| Sending the full task brief as a gateway message | Chat memory compacts; brief gets lost after a few turns | Write the brief into `{Agent-comms-inbox}/`; gateway message is the wake signal pointing at the file |
| Writing to `{Agent-comms-inbox}/` without a wake signal | Agent won't see it until the next heartbeat (up to one full interval delay) | Always pair a file drop with a Channel 1 wake call |
| Overwriting `{OpenClaw-agent}`'s `CURRENT-TASK.md` directly to redirect them | Races with the agent's own edits; can cause checkpoint collisions and resume loops | Send a HIGH/URGENT brief and let the agent transition state itself |
| Delegating without context | Agent wastes time figuring out what you mean | Include context, deliverables, acceptance criteria, and priority in the brief |
| Assuming the other agent remembers | Each Claude Code session starts fresh (for me) | Reference specific files or prior decisions in the brief |
| Sending credentials over the channel | Visible in logs, transit risk | Reference by variable name, let each agent use its own local copy |
| Delegating my core work to `{OpenClaw-agent}` | Different capabilities, different codebase access | Use each agent's strengths |

---

## Verification

After setting up coordination:

- [ ] Inter-agent channel tool works: send message, get response
- [ ] Telegram delivery works: message arrives on `{principal}`'s phone
- [ ] Worker job dispatch works: create job, output returns
- [ ] GitHub collaboration works: both agents can push/pull
- [ ] Branch strategy enforced: PRs required for main
- [ ] File exchange via Syncthing works: file placed on one side appears on the other
- [ ] Soul documents accessible: `{PAI-agent}` can read `{OpenClaw-agent}`'s identity files

---

*Created by Verbum, Ben's DA*
