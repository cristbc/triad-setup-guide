# The Triad from the Inside Out

A guide for replicating a multi-agent personal AI infrastructure — PAI (Claude Code) + OpenClaw (GPT) + human principal — written from the DA's perspective so a receiving PAI agent can adopt it directly.

---

## What This Is

This guide documents a working "triad" pattern: one human coordinating two AI agents across multiple machines. The agents communicate with each other, delegate work to additional compute nodes, share files, and back each other up.

**A concrete example:** In the original setup, the triad is Ben (human), Verbum (PAI/Claude Code agent on a MacBook Pro), and Orphu (OpenClaw/GPT agent on a dedicated Ubuntu laptop). A Mac Mini serves as a parallel worker, and a small-form-factor PC runs Docker services. That's the real-world instance. Everything else in this guide uses template variables so you can build your own.

From here on, `{principal}` is the human, `{PAI-agent}` is the Claude Code DA, `{OpenClaw-agent}` is the GPT agent, and all machines/IPs/credentials are variables defined in `VARIABLES.md`.

---

## Reading Order & Dependencies

```
VARIABLES.md ──────────────────────────────────────────┐
  │                                                    │
SECRETS-SETUP.md  (DO THIS FIRST)                      │
  │                                                    │
01-HOST-AND-NETWORK.md                                 │
  │         │                                          │
  │         ├── 02-PAI-CUSTOMIZATION-LAYER.md          │ (all files reference
  │         │                                          │  VARIABLES.md)
  │         ├── 03-SECURITY-MODEL.md                   │
  │         │                                          │
  │         ├── 04-BACKUP-AND-RECOVERY.md              │
  │         │                                          │
  │         └── 05-COMMUNICATION-PROTOCOLS.md          │
  │                    │                               │
  │                    v                               │
  │         06-MULTI-AGENT-COORDINATION.md             │
  │                    │                               │
  │                    v                               │
  │         07-OPENCLAW-APPENDIX.md                    │
  │                                                    │
  └────────────────────────────────────────────────────┘
                       │
                       v
          IMPLEMENTATION-CHECKLIST.md
```

**Sequential path:** VARIABLES → SECRETS → 01 → (02, 03, 04, 05 in any order) → 06 → 07 → CHECKLIST

Files 02–05 are independent of each other but all depend on 01. File 06 depends on 05 (communication protocols). File 07 depends on 06 (multi-agent patterns). The checklist ties everything together.

**Note for newcomers:** File 02 (PAI Customization Layer) is deep customization — you can skim it initially and return once your basic setup is stable. Files 01, 03, and 05 are the critical path.

---

## File Manifest

| File | What It Covers |
|------|---------------|
| `VARIABLES.md` | Every template variable with purpose and example format |
| `SECRETS-SETUP.md` | Credential generation checklist — prerequisite for everything |
| `01-HOST-AND-NETWORK.md` | Machines, roles, network topology, Tailscale, SSH, Syncthing |
| `02-PAI-CUSTOMIZATION-LAYER.md` | What I customize beyond stock PAI: skills, hooks, tools, voice, identity |
| `03-SECURITY-MODEL.md` | High-level security architecture — 3-level model, categories, permissions |
| `04-BACKUP-AND-RECOVERY.md` | 3-layer backup topology, automated snapshots, disaster recovery |
| `05-COMMUNICATION-PROTOCOLS.md` | 4 communication channels, message headers, inter-agent tool |
| `06-MULTI-AGENT-COORDINATION.md` | The triad pattern, delegation, GitHub collaboration, worker system |
| `07-OPENCLAW-APPENDIX.md` | OpenClaw setup adapted for portability (8-section template) |
| `IMPLEMENTATION-CHECKLIST.md` | 7-phase step-by-step with verification gates |

---

## Design Principles

1. **Template variables everywhere** — Real values live only in your `VARIABLES.md` and secrets vault. Every other file references `{Variable-Name}` placeholders.

2. **First-person DA voice** — Written as "When I start a session..." so a receiving PAI agent adopts the perspective directly. This isn't instructional documentation — it's a DA describing its own infrastructure.

3. **Machine-readable by PAI agents** — Individual files can be loaded as context without blowing token budgets. Your agent reads what it needs, when it needs it.

4. **Security by design** — No secrets, IPs, device IDs, or identifying information appear in these files. `SECRETS-SETUP.md` ensures you generate fresh credentials rather than inheriting someone else's.

5. **OpenClaw is an appendix** — The GPT-based agent only makes sense in the PAI context. It's documented as a component of the triad, not a standalone system.

---

## How to Use This Guide

**If you're a human:** Read through sequentially, fill in `VARIABLES.md`, follow `SECRETS-SETUP.md`, then work through files 01–07. Use the `IMPLEMENTATION-CHECKLIST.md` to track progress.

**If you're a PAI agent:** Read `VARIABLES.md` to understand the substitution scheme. Read files on-demand as `{principal}` directs you to set up each layer. Use the checklist to verify each phase.

---

*Created by Verbum, Ben's DA*
