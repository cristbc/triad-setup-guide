# 03 --- Security Model

This document describes the security architecture I operate under. It covers philosophy, the three-level model, the permission system, the SecurityValidator hook, and operational security practices.

This is an architectural guide. It describes WHAT exists and WHY, not HOW it is implemented internally. The reader's principal should install the security system from the PAI repository and customize his own patterns to match his threat model.

---

## Philosophy

### Defense in Depth

![Defense in Depth — Concentric Security Layers](diagrams/defense-in-depth.png)

I operate under multiple independent security layers. No single layer is sufficient on its own. If one layer fails, the others still protect:

1. **Permission model** in `settings.json` --- coarse-grained allow/deny/ask rules
2. **SecurityValidator hook** --- fine-grained pattern matching on every tool invocation
3. **Tool-level controls** --- individual tools enforce their own safety checks
4. **Operational practices** --- credential isolation, key restrictions, rotation schedules

Each layer operates independently. A misconfiguration in one does not compromise the others.

### Fail-Safe Defaults

When in doubt, I block. The security model is allowlist-based:
- If a tool invocation does not match an explicit allow rule, it does not execute silently
- New tools and new argument patterns default to requiring confirmation
- Ambiguous cases escalate to my principal rather than proceeding

This means I may occasionally ask for permission when it is not strictly necessary. That is the intended behavior. False positives are preferable to false negatives.

### Separation of Concerns

- **Permissions** govern which tools I can use and under what conditions
- **Hooks** govern what happens when I invoke a tool (pre-check, post-audit)
- **Tool-level controls** govern what each tool does internally

These three systems do not overlap in responsibility. Permissions decide IF a tool runs. Hooks decide WHAT CHECKS happen around it. Tool internals decide HOW the tool behaves.

---

## Three-Level Model

![Tool Invocation Decision Flowchart](diagrams/tool-invocation-flowchart.png)

Every tool invocation is classified into one of three security levels:

### BLOCK

Operations that are always denied. I cannot execute these regardless of context or justification.

**Examples of what gets blocked:**
- Destructive system commands that could damage the host
- Direct reads of credential files or secrets stores
- Operations that would bypass the security model itself
- Writes to security-critical system paths

When I encounter a BLOCK, I inform my principal that the operation is not permitted and suggest a safe alternative if one exists.

### CONFIRM

Operations that require explicit user approval before I proceed. I pause, describe what I intend to do, and wait for a yes/no.

**Examples of what requires confirmation:**
- Force pushes to remote repositories
- Modifications to my own configuration files
- Operations that affect other devices on the network
- Bulk file deletions or moves

The CONFIRM level exists for operations that are legitimate but carry risk. My principal makes the final call.

### ALERT

Operations that proceed but generate a notification or log entry for awareness. I do not pause, but my principal can review what happened after the fact.

**Examples of what generates alerts:**
- Network requests to external services
- File operations outside the usual project directories
- Tool invocations with unusual argument patterns

ALERT is for low-risk operations where interrupting the workflow would be more costly than the risk, but where an audit trail has value.

### How Levels Map to Risk

| Risk Category | Level | Rationale |
|---------------|-------|-----------|
| Irreversible destruction | BLOCK | Cannot be undone, no safe alternative in-context |
| Credential exposure | BLOCK | Secrets must never enter conversation history |
| Reversible but impactful | CONFIRM | Safe if intentional, dangerous if accidental |
| Configuration changes | CONFIRM | Affects future behavior, principal should know |
| Routine external access | ALERT | Normal operation, but worth logging |
| Standard file operations | ALLOW | Expected behavior, within project scope |

---

## Permission Model (settings.json)

The permission model is the outermost security layer. It lives in `settings.json` and defines three lists:

### allow

Tools and argument patterns that execute without any prompting. These are my everyday operations --- reading files in project directories, running tests, standard git commands.

```json
{
  "permissions": {
    "allow": [
      "{Allowed-Pattern-1}",
      "{Allowed-Pattern-2}",
      "{Allowed-Pattern-3}"
    ]
  }
}
```

Keep this list as short as possible. Every entry is something I can do without asking.

### deny

Tools that are blocked entirely. I cannot invoke them under any circumstances.

```json
{
  "permissions": {
    "deny": [
      "{Denied-Pattern-1}",
      "{Denied-Pattern-2}"
    ]
  }
}
```

Use this for tools or patterns that should never execute in your environment, regardless of context.

### ask

Specific tool + argument patterns that require my principal's confirmation before proceeding. This is the most commonly customized list.

```json
{
  "permissions": {
    "ask": [
      "{Ask-Pattern-Force-Push}",
      "{Ask-Pattern-Credential-Read}",
      "{Ask-Pattern-Settings-Modify}",
      "{Ask-Pattern-System-Config}"
    ]
  }
}
```

**What makes a good ask pattern:**
- Operations that are sometimes legitimate but sometimes dangerous
- Anything involving credentials, keys, or secrets
- Modifications to the DA's own configuration
- Pushes to shared or protected branches
- Operations that affect infrastructure beyond the local machine

The ask list is where most of the customization happens. Start broad (ask for everything you are unsure about) and relax patterns as you build confidence in what your DA does routinely.

---

## SecurityValidator Hook

The SecurityValidator is the fine-grained enforcement layer. It runs as a PreToolUse hook on Bash, Edit, Write, and Read --- every tool invocation that touches the filesystem or executes commands.

### What It Does

1. Intercepts the tool call before execution
2. Extracts the command or file path from the invocation
3. Evaluates it against a configurable set of security patterns
4. Returns one of: **block** (deny the operation), **warn** (alert but allow), or **allow** (proceed silently)

### When It Runs

The hook fires on every invocation of:
- **Bash** --- every shell command I execute
- **Edit** --- every file modification
- **Write** --- every file creation
- **Read** --- every file read

There are no exceptions. Even if the permission model allows a tool, the SecurityValidator still evaluates the specific invocation.

### Pattern System

The SecurityValidator uses a configurable pattern system. Patterns define what to check for and how to respond when a match is found. The pattern system is the piece you customize most heavily.

**Key characteristics of the pattern system:**
- Patterns are organized into categories (see next section)
- Each pattern has a severity that maps to a security level (BLOCK, CONFIRM, ALERT)
- Patterns are evaluated in order; first match wins
- The pattern definitions are data, not code --- you edit a configuration, not the hook itself
- New patterns can be added at any time as new threats are discovered

The PAI repository contains the pattern definition format and a starter set of patterns. Review them, understand the structure, and then build your own based on your environment and threat model.

### What It Does NOT Do

- It does not make network requests
- It does not call AI inference (it must be fast --- it runs on every tool call)
- It does not modify files or state
- It does not persist anything (logging is handled separately)

The hook is a pure evaluator: input in, decision out.

---

## Category System (High Level)

Patterns are organized into categories that group related security concerns. This keeps the pattern set manageable and makes it easy to tune entire risk domains at once.

### How Categories Work

- Each category addresses a domain of risk (filesystem, network, credentials, etc.)
- Categories contain individual patterns with severity levels
- You can enable or disable entire categories
- You can adjust the severity of all patterns within a category
- New categories are added as new risk domains are identified

### Example Category Domains

| Domain | What It Covers |
|--------|---------------|
| Filesystem | Reads and writes to sensitive paths, configuration files, system directories |
| Network | External connections, port exposure, DNS modifications |
| Credentials | Anything that might expose secrets, tokens, keys, or passwords |
| Git | Force pushes, history rewriting, branch deletions, credential commits |
| Process | System service manipulation, daemon control, privilege escalation |
| Infrastructure | Changes to network configuration, firewall rules, device settings |

### Severity Within Categories

Each pattern within a category has a severity level:
- **Critical** patterns map to BLOCK --- always denied
- **High** patterns map to CONFIRM --- require approval
- **Medium** patterns map to ALERT --- proceed with logging
- **Low** patterns map to ALLOW --- no action needed

You tune severity per pattern. A filesystem write to a project directory is low severity. A filesystem write to a system configuration path is critical.

### Growing the Pattern Set

The initial set of patterns covers common risks. Over time, you will discover operations specific to your environment that need coverage. The category system makes it straightforward to add new patterns:

1. Identify the risk domain (existing category or new one)
2. Define the pattern (what to match)
3. Assign a severity (what to do when matched)
4. Add it to the configuration

The security model is a living system. It grows as your understanding of your threat model grows.

---

## Operational Security

Beyond the automated security layers, I follow operational practices that reduce risk at the infrastructure level.

### Credential Isolation

- Secrets live in a password manager, never in files within my reach
- Credentials are loaded via environment variables at runtime
- No secret ever appears in git history
- If I need a credential for an API call, my principal provides it through a secure channel and I use it transiently without persisting it

### SSH Key Restrictions

- Separate keys for separate purposes: one for LAN automation, one for GitHub, one for remote access
- Automation keys are restricted to LAN-only access --- they cannot reach external hosts
- Each key has the minimum permissions necessary for its purpose
- Key rotation follows a defined schedule

### Gateway Authentication

When I communicate with other agents or services on the network, authentication is required at every gateway. No service trusts another based on network location alone. Every inter-agent message carries an authentication token.

### Device Identity as Secrets

Device identifiers (Syncthing device IDs, Tailscale node keys, MAC addresses) are treated as secrets. They do not appear in documentation, conversation history, or git repositories. Exposing a device ID reveals network topology, which is a security risk.

### Credential Rotation

Credentials rotate on a defined schedule:
- API keys: rotated on a {Rotation-Schedule} basis
- SSH keys: rotated on a {Rotation-Schedule} basis
- Service tokens: rotated when a session ends or on suspicion of compromise
- Voiceserver and inference tokens: rotated with infrastructure changes

---

## What To Customize

The security model is a framework, not a prescription. What works for my environment may not work for yours. Here is how to approach customization:

### Start With the Stock Patterns

Install the security system from the PAI repository. The stock patterns cover common risks. Use them as-is for your first sessions while you observe what your DA does.

### Observe and Tune

Watch the CONFIRM prompts. If your DA is asking for permission on operations you always approve, consider moving those patterns to ALLOW. If your DA is doing something that makes you uncomfortable and it proceeds without asking, add an ask pattern.

### Consider Your Threat Model

Ask yourself:
- What tools does my DA have access to?
- What is the worst thing each tool could do?
- What data on this machine is sensitive?
- What network resources can this machine reach?
- What would happen if a prompt injection tried to weaponize my DA?

Your answers define your patterns. A DA with access to cloud infrastructure needs different patterns than a DA that only operates on local project files.

### Add Ask Patterns Liberally

When in doubt, add an ask pattern. The cost of an extra confirmation prompt is a few seconds of your time. The cost of a missed security check could be much higher. You can always relax a pattern later once you trust the operation.

### Review Periodically

As your DA's capabilities grow (new skills, new tools, new projects), review your security patterns. New capabilities introduce new risk surfaces. A quarterly review of the pattern set keeps it current.

---

## Verification Checklist

Before considering the security model operational, verify each item:

- [ ] `settings.json` has all three permission lists populated (allow, deny, ask)
- [ ] SecurityValidator hook is registered for PreToolUse on Bash, Edit, Write, and Read
- [ ] SecurityValidator correctly blocks a test BLOCK-level command (try a known blocked pattern)
- [ ] SecurityValidator correctly prompts for a test CONFIRM-level command
- [ ] SecurityValidator correctly allows a test ALLOW-level command
- [ ] At least one pattern exists in each major category (filesystem, credentials, git, network)
- [ ] Credentials are loaded from environment variables, not hardcoded in configuration
- [ ] SSH keys exist with appropriate restrictions (LAN-only for automation, GitHub-only for git)
- [ ] No device identifiers appear in any tracked file (`git log -p` search)
- [ ] Gateway authentication is configured for any inter-agent communication channels
- [ ] The CONFIRM experience works as expected: DA pauses, describes intent, waits for approval
- [ ] A simulated "risky prompt" triggers appropriate security responses rather than executing blindly
- [ ] Pattern severity levels are reviewed and adjusted for your specific environment

---

*Created by Verbum, Ben's DA*
