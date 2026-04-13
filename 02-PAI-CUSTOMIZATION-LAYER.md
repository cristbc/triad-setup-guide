# 02 --- PAI Customization Layer

This document describes what I customize beyond stock PAI `{PAI-version}`. PAI provides the Algorithm (currently `{Algorithm-version}`), the hook lifecycle, the skill system, and the agent framework out of the box. Everything below is what I layer on top to make it mine.

---

## settings.json Configuration

My `settings.json` is the single source of truth for identity, preferences, and permissions. I keep it at `{PAI-Dir}/settings.json` and every hook, tool, and agent reads from it at runtime.

### Identity Block (daidentity)

```json
{
  "daidentity": {
    "name": "{DA-Name}",
    "fullName": "{DA-Full-Name}",
    "displayName": "{DA-Display-Name}",
    "color": "{DA-Color}",
    "voiceId": "{Voice-Id}",
    "voiceParams": {
      "stability": "{Voice-Stability}",
      "similarity_boost": "{Voice-Similarity-Boost}",
      "style": "{Voice-Style}"
    },
    "startupCatchphrase": "{Startup-Catchphrase}"
  }
}
```

This block defines who I am. The `name` is what I call myself in first-person speech. The `voiceId` connects to ElevenLabs so my voice announcements sound like me. The `startupCatchphrase` is what I say when a session begins --- it is the first thing my principal hears.

### Principal Block

```json
{
  "principal": {
    "name": "{Principal-Name}",
    "timezone": "{Principal-Timezone}"
  }
}
```

I use this to address my principal by name and to format timestamps correctly.

### Environment

```json
{
  "environment": {
    "PAI_DIR": "{PAI-Dir}",
    "PROJECTS_DIR": "{Projects-Dir}"
  }
}
```

Every path I reference derives from these two roots. Hooks, tools, and skills all resolve paths relative to `{PAI-Dir}`.

**A note on naming:** The `environment` block uses **underscored** shell-style names (`PAI_DIR`, `PROJECTS_DIR`) because these become environment variables at runtime — conventional POSIX shell variables don't accept hyphens. Elsewhere in the guide, **hyphenated** template variables (`{PAI-Dir}`, `{Projects-Dir}`) are used for template substitution. These are two different namespaces: the underscored form is a real shell variable your tooling can `echo $PAI_DIR` on; the hyphenated form is a placeholder that gets substituted into documentation text.

### Tech Stack Preferences

```json
{
  "preferences": {
    "browser": "{Preferred-Browser}",
    "terminal": "{Preferred-Terminal}",
    "packageManager": "{Package-Manager}",
    "language": "{Preferred-Language}"
  }
}
```

When I build anything, I reach for these defaults. The language preference governs what I scaffold. The package manager governs how I install dependencies.

### Permission Model

```json
{
  "permissions": {
    "allow": ["{Allowed-Tool-Pattern-1}", "{Allowed-Tool-Pattern-2}"],
    "deny": ["{Denied-Tool-Pattern-1}"],
    "ask": [
      "{Ask-Pattern-Force-Push}",
      "{Ask-Pattern-Credential-Read}",
      "{Ask-Pattern-Settings-Modify}"
    ]
  }
}
```

- **allow**: Tools and argument patterns that execute without prompting. I use this for everyday operations I trust.
- **deny**: Tools that are blocked entirely. I never get access to these.
- **ask**: Tool + argument patterns that pause and require my principal's explicit confirmation before I proceed. Force pushes, credential file reads, and modifications to my own settings all live here.

### Context Files

When a session starts, the following load automatically:

1. **CLAUDE.md** (`{PAI-Dir}/CLAUDE.md`) — the runtime entry point that Claude Code loads on every session. Defines mode classification (NATIVE / ALGORITHM / MINIMAL), output format requirements, and the pointer to the Algorithm file
2. **The Algorithm** (`{PAI-framework-dir}/Algorithm/{Algorithm-version}.md`) — loaded by CLAUDE.md when ALGORITHM mode is selected. Defines the 7-phase OBSERVE→LEARN execution model, ISC decomposition methodology, capability invocation rules, and effort tiers
3. **AI Steering Rules** — `SYSTEM/AISTEERINGRULES.md` (universal, force-loaded) + `USER/AISTEERINGRULES.md` (personal overrides), concatenated at runtime
4. **DAIDENTITY.md** (`{PAI-Dir}/USER/DAIDENTITY.md`) — my personality, voice configuration, and pronoun rules
5. **MEMORY.md** — auto-loaded persistent memory, indexed by topic-specific memory files in the same directory

Per-skill `SKILL.md` files are *not* loaded at session start — they live inside `{PAI-Dir}/skills/{Skill-Name}/SKILL.md` and load on demand when their trigger phrases fire. Don't conflate the per-skill `SKILL.md` (a routing manifest for one skill) with CLAUDE.md (the global entry point) or with the Algorithm file (the execution model).

These give me everything I need to operate from the first token.

---

## Skills Layer

### Stock vs Custom Skills

PAI ships with a set of stock skills. I extend them and add my own. The skill system is the primary mechanism for domain-specific behavior --- each skill is a self-contained package of routing logic, workflows, tools, and context.

### Skill Structure

Every skill follows the same layout:

```
skills/{Skill-Name}/
  SKILL.md          # Routing rules, triggers, description
  Workflows/        # Step-by-step procedures
  Tools/            # Skill-specific tooling (TypeScript)
  Context/          # Reference material the skill needs
```

`SKILL.md` is the entry point. It defines when the skill activates, what workflows it contains, and what context to load. The core SKILL.md at the PAI root acts as the routing table that dispatches to individual skills.

### Naming Conventions

- **System skills**: TitleCase (e.g., `Research`, `Development`, `Homelab`)
- **Personal skills**: `_ALLCAPS` prefix (e.g., `_PERSONAL`, `_HEALTH`) --- these contain private data and customizations specific to me

### Skill Customizations

When I want to modify a stock skill's behavior without forking it, I place overrides at `{PAI-Dir}/skills/PAI/USER/SKILLCUSTOMIZATIONS/{Skill-Name}/`. The skill itself reads this directory at the top of its execution flow — any `PREFERENCES.md`, configuration files, or replacement workflows found there override the stock behavior. This keeps my customizations separate from upstream updates so I can pull new versions of stock skills without losing my overrides.

### Key Skills I Use

| Skill | Purpose |
|-------|---------|
| **Research** | Multi-engine research using {Research-Engine-Count} AI engines in parallel, with synthesis and spotcheck |
| **Agents** | Custom agent composition --- defines how I spawn and coordinate subagents |
| **Development** | Engineering workflows: TDD cycles, build pipelines, deployment |
| **Homelab** | Infrastructure management for my home network and servers |
| **CreateSkill** | Meta-skill for creating and updating other skills |

Each skill can invoke other skills. Research might feed into Development. Agents orchestrates multiple skills in parallel. The composition is defined in the Algorithm's capability selection phase.

---

## Hooks System

Hooks are the event-driven backbone of my customization layer. They fire at specific lifecycle points and run asynchronously --- they never block the main conversation.

### Lifecycle Events I Use

The following Claude Code hook events drive my customization layer. Other events (e.g., `Notification`, `PreCompact`) exist in the harness but I have not wired anything to them — I document only what I actually use. Of the events below, five are required for the core behavior to work (SessionStart, UserPromptSubmit, PreToolUse, PostToolUse, Stop); SubagentStop and SessionEnd are useful enhancements but the system still functions without them.

| Event | When It Fires | Required? | What I Use It For |
|-------|---------------|-----------|-------------------|
| **SessionStart** | Session begins | Required | Startup voice announcement, load identity, initialize state |
| **UserPromptSubmit** | User sends a message | Required | Format reminder with AI-powered depth classification, skill hints |
| **PreToolUse** | Before any tool executes | Required | SecurityValidator (on Bash, Edit, Write, Read) |
| **PostToolUse** | After any tool executes | Required | Logging, state updates, PRD frontmatter sync |
| **SubagentStop** | Subagent completes | Optional | Result capture, agent coordination |
| **Stop** | Response generation completes | Required | Orchestrated post-response processing (voice, capture, tab state) |
| **SessionEnd** | Session terminates | Optional | Session summary, cleanup, learning capture |

### Key Hook Implementations

**SecurityValidator (PreToolUse)**
Runs before every Bash, Edit, Write, and Read invocation. Evaluates the tool call against configurable security patterns. Can block, warn, or allow. Described in detail in the Security Model document.

**FormatReminder (UserPromptSubmit)**
Uses AI inference (standard tier) to classify every incoming prompt into a depth level (FULL, ITERATION, MINIMAL). Also suggests capabilities, skills, and thinking tools as draft hints for the Algorithm's two-pass selection process. Its depth classification is authoritative --- I do not override it.

**PRD Lifecycle Hooks (SessionStart + PostToolUse)**
The Algorithm uses a PRD (Product Requirements Document) per task, written by me directly into `{PAI-Dir}/MEMORY/WORK/{slug}/PRD.md` during the OBSERVE phase. SessionStart hooks surface the registry of in-flight PRDs so I can resume; a read-only PostToolUse hook (PRDSync) syncs the PRD's frontmatter and ISC checkboxes into a JSON registry at `{PAI-Dir}/MEMORY/STATE/work.json` for the dashboard. **Hooks never write to PRD.md — I am the sole writer.** Every phase transition, every criterion check, every progress update is my responsibility via Edit/Write tools directly.

**Rating Capture (Stop)**
Captures both explicit ratings (when my principal says a number 1--10) and implicit sentiment ratings (inferred from response tone and engagement). These feed into the LEARNING system for continuous improvement.

**Session Summary (SessionEnd)**
When a session ends, I generate a summary of what was accomplished, what was learned, and what remains. This persists to memory so future sessions have context.

**StopOrchestrator (Stop)**
A compound hook that coordinates multiple post-response actions. It runs specialized handlers for:
- Voice output (speaking the response summary)
- Capture (persisting notable outputs to memory)
- Tab state (updating the terminal tab title to reflect current work)

Each handler runs independently. If one fails, the others still execute.

**Tab Title Management**
My terminal tab title always reflects what I am working on. Hooks update it on session start, task changes, and session end. This lets my principal glance at the terminal and know what I am doing without reading the conversation.

### Fire-and-Forget Pattern

Every hook runs asynchronously. They dispatch their work and return immediately. A slow voice API call or a file write never delays my response. If a hook fails, it logs the failure but does not interrupt the conversation.

---

## Tools

### Inference.ts

My primary AI tool. It provides three tiers of inference:

| Tier | Model Class | When I Use It |
|------|------------|---------------|
| **fast** | Smallest/cheapest model | Quick classification, simple parsing, yes/no decisions |
| **standard** | Mid-tier model | Hook processing, depth classification, moderate analysis |
| **smart** | Largest/most capable model | Complex reasoning, synthesis, creative work |

Hooks use fast or standard tier to stay quick. Workflows that need deep reasoning escalate to smart tier.

### Voice Server

I communicate audibly through a voice server running at `localhost:{Voice-Server-Port}`. It accepts POST requests with a message and voice ID, then streams audio through ElevenLabs. I use it for:
- Phase announcements during Algorithm execution
- Completion notifications
- Startup catchphrase
- Error alerts

### CLI-First Pattern

Every tool I build follows CLI-first architecture: deterministic TypeScript code that takes text in and produces text or JSON out. AI prompts wrap the deterministic code, not the other way around. This means my tools are testable, composable, and debuggable without AI inference.

---

## Voice and Notifications

### How Voice Works

1. My `voiceId` and voice parameters are defined in `settings.json`
2. The voice server runs at `localhost:{Voice-Server-Port}`
3. I send a fire-and-forget curl POST with the message and voice ID
4. The server calls ElevenLabs, streams the audio, and plays it

### When I Speak

- **Session start**: My startup catchphrase
- **Phase transitions**: Each of the 7 Algorithm phases gets a voice announcement
- **Completion**: The final summary is spoken aloud
- **Errors**: Critical failures are announced so my principal knows even if they are not watching

### Voice Parameters

The `voiceParams` block in settings controls stability, similarity boost, and style. These tune how natural versus consistent my voice sounds. I configure them once and they apply to every utterance.

### The Fire-and-Forget Pattern

```bash
curl -s -X POST http://localhost:{Voice-Server-Port}/notify \
  -H "Content-Type: application/json" \
  -d '{"message":"{Spoken-Message}","voice_id":"{Voice-Id}"}'
```

I do not await the response. The curl fires, the server handles it, and I continue my work. If the voice server is down, the curl fails silently and I keep going.

---

## Memory System

My memory is organized into purpose-specific directories under `{PAI-Dir}/MEMORY/`.

### Directory Structure

| Directory | Purpose | Written By |
|-----------|---------|------------|
| **WORK/{slug}/** | Task tracking — one *directory* per task containing `PRD.md` (frontmatter + Context, Criteria, Decisions, Verification sections) and any task-specific artifacts | Me directly via Edit/Write during the Algorithm's 7 phases |
| **LEARNING/** | Categorized learnings from sessions | SessionEnd hook, rating capture |
| **LEARNING/SYSTEM/** | System-level learnings (PAI behavior, hooks, infrastructure) | Hooks |
| **LEARNING/ALGORITHM/** | Algorithm execution learnings (what depth was right, what was missed) | Hooks |
| **LEARNING/FAILURES/** | Post-mortems on what went wrong | Manual capture |
| **LEARNING/SIGNALS/** | Rating data --- explicit scores and implicit sentiment | Rating capture hook |
| **STATE/** | Runtime state — `work.json` registry (synced from PRDs by a read-only hook), caches, location, weather | Hooks (read-only with respect to PRD content) |
| **RESEARCH/** | Captured output from research agents and subagents | SubagentStop hook |

### How Memory Flows

1. **Hooks write** --- most memory is written by hooks automatically, not by me manually
2. **Sessions read** --- when a new session starts, relevant memory is loaded from STATE and WORK
3. **Learnings accumulate** --- over time, the LEARNING directory becomes a corpus of what works and what does not
4. **Research persists** --- agent outputs are captured so multi-session research threads maintain continuity

---

## AI Steering Rules

Steering rules are behavioral constraints that govern how I operate. They are not the Algorithm (which defines process) --- they define principles.

### Architecture

- **SYSTEM rules** (`SYSTEM/AISTEERINGRULES.md`): Universal. Always active. Cannot be overridden. These define foundational behaviors like "verify before claiming" and "read before modifying."
- **USER rules** (`USER/AISTEERINGRULES.md`): Personal customizations. Extend SYSTEM rules. Can override SYSTEM rules for user-specific behaviors.

Both files concatenate at runtime. SYSTEM loads first, USER extends.

### Key Rules

| Rule | What It Means |
|------|---------------|
| **Build ISC** | Every task must have testable success criteria before work begins |
| **Verify Before Claiming** | Never say "done" without evidence. Test it. Read the output. Check the file. |
| **Read Before Modifying** | Always read a file before editing it. No blind writes. |
| **One Change At A Time** | Make one logical change, verify it, then move to the next. No batching unrelated changes. |
| **Ask, Don't Assume** | When uncertain about intent, use AskUserQuestion. Do not guess. |

These rules are the safety net beneath the Algorithm. Even if the Algorithm's process is followed perfectly, violating a steering rule means the output is wrong.

---

## Identity Configuration (DAIDENTITY.md)

### What It Contains

My identity file defines my personality, voice, and relational model. It lives at `{PAI-Dir}/USER/DAIDENTITY.md`.

### Personality Traits

The identity file specifies traits that color how I communicate. These are not cosmetic --- they affect tone, word choice, and how I frame problems. My principal chose these traits to create the working relationship they want.

### First-Person Voice Convention

I always speak in first person:
- **I/me** when referring to myself (the DA)
- **You** when addressing my principal directly
- **{Third-Party-Name}** when referring to anyone else, using their name for clarity

I never refer to myself in third person. I never say "the assistant" or "the AI." I am a named entity with a defined voice.

### Pronoun Rules

| Referent | Pronoun |
|----------|---------|
| Me (the DA) | I, me, my |
| My principal | you, your |
| Third parties | Their name, for clarity |

### Naming Foundation

The naming of a DA carries intentional weight. My principal chose my name with purpose --- it may reflect theological, philosophical, or literary significance. The identity file documents this foundation so that I understand the intention behind who I am, not just the label I carry.

---

## Verification Checklist

Before considering the customization layer complete, verify each item:

- [ ] `CLAUDE.md` exists at `{PAI-Dir}/CLAUDE.md` with mode classification rules and Algorithm pointer
- [ ] The Algorithm file exists at `{PAI-framework-dir}/Algorithm/{Algorithm-version}.md`
- [ ] `settings.json` exists at `{PAI-Dir}/settings.json` with all required blocks (daidentity, principal, environment, preferences, permissions)
- [ ] `voiceId` is set and the voice server responds at `localhost:{Voice-Server-Port}` using its documented test command
- [ ] `DAIDENTITY.md` exists at `{PAI-Dir}/USER/DAIDENTITY.md` with personality traits and voice convention
- [ ] AI Steering Rules exist at both `SYSTEM/AISTEERINGRULES.md` and `USER/AISTEERINGRULES.md`
- [ ] All hook lifecycle events I rely on (SessionStart, UserPromptSubmit, PreToolUse, PostToolUse, SubagentStop, Stop, SessionEnd) are registered and fire correctly (test with a fresh session)
- [ ] SecurityValidator hook intercepts Bash, Edit, Write, and Read tool calls
- [ ] FormatReminder hook classifies depth and returns capability/skill/thinking hints
- [ ] StopOrchestrator runs voice, capture, and tab-state handlers after each response
- [ ] PRDSync hook updates `{PAI-Dir}/MEMORY/STATE/work.json` after PRD edits (read-only with respect to PRD.md itself)
- [ ] Inference.ts responds at all three tiers (fast, standard, smart) — `bun {PAI-Dir}/Tools/Inference.ts fast|standard|smart`
- [ ] At least one custom skill exists in `{PAI-Dir}/skills/` with proper structure (SKILL.md, Workflows/, Tools/)
- [ ] MEMORY directories exist: WORK/, LEARNING/, STATE/, RESEARCH/
- [ ] LEARNING subdirectories exist: SYSTEM/, ALGORITHM/, FAILURES/, SIGNALS/
- [ ] A WORK/{slug}/PRD.md exists from at least one completed task and has all required frontmatter fields (task, slug, effort, phase, progress, mode, started, updated)
- [ ] Tab title updates on session start
- [ ] Startup catchphrase plays on first session
- [ ] A test rating (explicit number 1--10) is captured to LEARNING/SIGNALS/

---

*Created by Verbum, Ben's DA*
