# The Triad from the Inside Out

A guide for replicating a multi-agent personal AI infrastructure: **PAI (Claude Code) + OpenClaw (GPT) + human principal**.

Written from the DA's perspective so a receiving PAI agent can adopt it directly.

---

## What Is This?

This documents a working "triad" pattern: one human coordinating two AI agents across multiple machines. The agents communicate with each other, delegate work, share files, and back each other up.

The guide uses **template variables** (`{Variable-Name}`) throughout so you can build your own instance without inheriting anyone else's configuration or credentials.

---

## Quick Start

1. Read [`INDEX.md`](INDEX.md) for the full reading order and dependency map
2. Copy [`VARIABLES.md`](VARIABLES.md) and fill in your values
3. Follow [`SECRETS-SETUP.md`](SECRETS-SETUP.md) to generate fresh credentials
4. Work through files 01-07 in order
5. Use [`IMPLEMENTATION-CHECKLIST.md`](IMPLEMENTATION-CHECKLIST.md) to track progress

---

## What's Inside

| File | Covers |
|------|--------|
| [`INDEX.md`](INDEX.md) | Reading order, dependency map, design principles |
| [`VARIABLES.md`](VARIABLES.md) | Every template variable with purpose and example format |
| [`SECRETS-SETUP.md`](SECRETS-SETUP.md) | Credential generation checklist (prerequisite) |
| [`01-HOST-AND-NETWORK.md`](01-HOST-AND-NETWORK.md) | Machines, topology, Tailscale, SSH, Syncthing |
| [`02-PAI-CUSTOMIZATION-LAYER.md`](02-PAI-CUSTOMIZATION-LAYER.md) | Skills, hooks, tools, voice, identity |
| [`03-SECURITY-MODEL.md`](03-SECURITY-MODEL.md) | 3-level security architecture |
| [`04-BACKUP-AND-RECOVERY.md`](04-BACKUP-AND-RECOVERY.md) | 3-layer backup, automated snapshots, DR runbooks |
| [`05-COMMUNICATION-PROTOCOLS.md`](05-COMMUNICATION-PROTOCOLS.md) | 4 communication channels, inter-agent messaging |
| [`06-MULTI-AGENT-COORDINATION.md`](06-MULTI-AGENT-COORDINATION.md) | The triad pattern, delegation, GitHub collab |
| [`07-OPENCLAW-APPENDIX.md`](07-OPENCLAW-APPENDIX.md) | OpenClaw agent setup (8-section template) |
| [`IMPLEMENTATION-CHECKLIST.md`](IMPLEMENTATION-CHECKLIST.md) | 7-phase step-by-step with verification gates |

The `diagrams/` directory contains 9 technical illustrations and 2 Mermaid diagrams embedded throughout the guide.

---

## Diagrams

All technical diagrams were generated using Nano Banana Pro and are embedded at relevant points in the guide documents.

| Diagram | Used In |
|---------|---------|
| Network Topology (LAN + Tailscale) | 01 - Host and Network |
| Four Communication Channels Map | 05 - Communication Protocols |
| Channel 1 Fallback Flowchart | 05 - Communication Protocols |
| Defense-in-Depth Concentric Layers | 03 - Security Model |
| Tool Invocation Decision Flowchart | 03 - Security Model |
| 3-Layer Backup Topology | 04 - Backup and Recovery |
| DR Recovery Source Matrix | 04 - Backup and Recovery |
| Triad Entity & Delegation Flow | 06 - Multi-Agent Coordination |
| Gateway Trust Boundary | 07 - OpenClaw Appendix |

---

## Design Principles

- **Template variables everywhere** -- real values live only in your `VARIABLES.md` and secrets vault
- **First-person DA voice** -- written as "When I start a session..." for direct adoption by a receiving PAI agent
- **Machine-readable** -- individual files load as context without blowing token budgets
- **Security by design** -- no secrets, IPs, or identifying information in these files
- **OpenClaw is an appendix** -- the GPT agent is documented as a component of the triad, not standalone

---

## Built With

- [PAI (Personal AI Infrastructure)](https://github.com/danielmiessler/PAI) v2.5
- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) by Anthropic
- [OpenClaw](https://github.com/openclaw/openclaw) for the GPT agent
- [Tailscale](https://tailscale.com/) for secure networking
- [Syncthing](https://syncthing.net/) for file synchronization

---

*Created by Verbum, Ben's DA*
