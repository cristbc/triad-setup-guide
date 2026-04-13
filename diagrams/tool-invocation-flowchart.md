# Tool Invocation — Decision Flowchart

Embed in `03-SECURITY-MODEL.md` near the SecurityValidator section.

```mermaid
flowchart TD
    REQ["PAI agent attempts<br/>Bash / Edit / Write / Read"] --> HOOK

    HOOK["PreToolUse hook fires<br/>(SecurityValidator)"]
    HOOK --> MATCH

    MATCH{"Match against<br/>security patterns"}

    MATCH -->|"matches DENY pattern"| BLOCK
    MATCH -->|"matches ASK pattern"| CONFIRM
    MATCH -->|"matches ALLOW pattern"| ALLOW
    MATCH -->|"matches ALERT pattern"| ALERT
    MATCH -->|"no pattern match"| DEFAULT

    BLOCK["BLOCK<br/>tool call rejected<br/>logged to MEMORY/SECURITY<br/>principal not prompted"]
    CONFIRM["CONFIRM<br/>principal prompted<br/>(AskUserQuestion with consequences)"]
    ALLOW["ALLOW<br/>tool call proceeds silently"]
    ALERT["ALERT<br/>tool call proceeds<br/>but logged + announced"]
    DEFAULT["DEFAULT<br/>(usually ALLOW for safe ops,<br/>CONFIRM for sensitive paths)"]

    CONFIRM -->|"approved"| EXEC
    CONFIRM -->|"denied"| ABORT["ABORT<br/>principal declined"]
    ALLOW --> EXEC
    ALERT --> EXEC
    DEFAULT --> EXEC

    EXEC["Tool executes"]
    EXEC --> POST

    POST["PostToolUse hook fires<br/>(logging, state updates)"]
    POST --> DONE["Done"]

    BLOCK --> DENIED["Denied<br/>agent must find another path"]
    ABORT --> DENIED

    style REQ fill:#1E40AF,stroke:#3B82F6,color:#F9FAFB
    style HOOK fill:#374151,stroke:#9CA3AF,color:#F9FAFB
    style MATCH fill:#1F2937,stroke:#6B7280,color:#F9FAFB
    style BLOCK fill:#5B1A1A,stroke:#EF4444,color:#F9FAFB
    style CONFIRM fill:#5B4A1A,stroke:#F59E0B,color:#F9FAFB
    style ALLOW fill:#1A3D1A,stroke:#22C55E,color:#F9FAFB
    style ALERT fill:#3D1A5B,stroke:#A855F7,color:#F9FAFB
    style DEFAULT fill:#374151,stroke:#9CA3AF,color:#F9FAFB
    style ABORT fill:#5B1A1A,stroke:#EF4444,color:#F9FAFB
    style EXEC fill:#1A3D1A,stroke:#22C55E,color:#F9FAFB
    style POST fill:#374151,stroke:#9CA3AF,color:#F9FAFB
    style DONE fill:#1A3D1A,stroke:#22C55E,color:#F9FAFB
    style DENIED fill:#5B1A1A,stroke:#EF4444,color:#F9FAFB
```

**Reading notes:**
- **BLOCK** is non-negotiable. The tool call never executes. The principal is *not* prompted because we don't want to give the agent any lever to argue past it.
- **CONFIRM** is the explicit consent path: the principal sees the action plus its consequences (force push will rewrite remote history, credential read exposes the file contents, etc.) and approves or denies.
- **ALERT** allows the action but creates an audit trail and a notification — useful for "I want to know this happened but it's not worth blocking."
- The default is ALLOW for everyday operations (file reads in the project, normal `git status`, etc.) and CONFIRM for sensitive paths (anything in `~/.ssh/`, `~/.openclaw/`, `settings.json`).
- Patterns live in `{PAI-Dir}/skills/PAI/USER/PAISECURITYSYSTEM/` so they survive PAI upgrades. (Template variable braces are intentionally NOT used inside the diagram nodes themselves because Mermaid parses `{...}` as a rhombus shape delimiter — the reading notes and body text are where variables go.)
