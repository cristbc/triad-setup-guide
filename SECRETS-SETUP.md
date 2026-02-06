# Secrets Setup — DO THIS FIRST

Before following any other guide, generate every credential your system needs. This ensures you have fresh, unique secrets — never inherited from someone else's installation.

Every credential below maps to a variable in `VARIABLES.md`. After generation, store all values in your password manager (1Password, Bitwarden, etc.). They should exist in exactly two places: your password manager and the specific config file that consumes them. Nowhere else.

---

## Credential Generation Checklist

### 1. SSH Keys

Generate two separate ed25519 keys — one for automation (passphrase-less, LAN-only) and one for GitHub.

```bash
# Automation key (no passphrase — used by scripts)
ssh-keygen -t ed25519 -f ~/.ssh/{SSH-automation-key} -N "" -C "homelab-automation"

# GitHub key (with passphrase)
ssh-keygen -t ed25519 -f ~/.ssh/{SSH-github-key} -C "github-{principal}"
```

**Post-generation:**
- Add `{SSH-automation-key}.pub` to `~/.ssh/authorized_keys` on every target machine
- Restrict automation key server-side: `from="{LAN-subnet}"` in `authorized_keys`
- Add `{SSH-github-key}.pub` to your GitHub account → Settings → SSH Keys
- For the GitHub key (which has a passphrase), use `ssh-add` with your OS keychain so you don't re-enter it constantly: `ssh-add --apple-use-keychain ~/.ssh/{SSH-github-key}` (macOS) or configure `ssh-agent` on Linux
- Store both private keys in your password manager

### 2. OpenClaw Gateway Token

```bash
openssl rand -hex 24
```

**Post-generation:**
- Store as `{gateway-token}` in your password manager
- Configure in OpenClaw gateway service config on `{OpenClaw-machine}`
- Configure in your inter-agent channel tool on `{PAI-machine}`

### 3. Tailscale

No manual key generation needed. Tailscale handles key exchange automatically.

**Setup:**
- Install Tailscale on every machine
- Authenticate each machine to the same Tailscale account
- Enable Tailscale SSH on machines that need remote access
- Record each machine's Tailscale IP as the corresponding `{*-TS-IP}` variable

### 4. Syncthing Device IDs

Auto-generated on first run. No manual generation.

**Setup:**
- Install Syncthing on each machine
- Start Syncthing once to generate device ID
- Record each machine's device ID as `{Syncthing-device-ID-*}`
- Pair machines according to the hub-and-spoke topology in `01-HOST-AND-NETWORK.md`

**Important:** Syncthing device IDs function as authentication tokens. Treat them as secrets — don't publish them.

### 5. ElevenLabs Voice

**Setup:**
- Create account at elevenlabs.io
- Clone or create a voice
- Copy the Voice ID from the dashboard
- Store as `{Voice-Id}`
- Configure voice parameters (`{Voice-Stability}`, `{Voice-Similarity-Boost}`, etc.) to taste

### 6. Telegram Bot (for mobile access to OpenClaw agent)

**Setup:**
- Message @BotFather on Telegram
- `/newbot` → choose display name → choose username
- Record bot username as `{telegram-bot}`
- Record API token as `{telegram-bot-token}`
- Store token in your password manager

### 7. API Keys (not templated — varies by installation)

You'll need API keys for services you choose to use. Common ones:

| Service | Where to Get | Used By |
|---------|-------------|---------|
| Anthropic API | console.anthropic.com | Claude Code (usually handled by `claude` CLI auth) |
| OpenAI API | platform.openai.com | OpenClaw agent |
| ElevenLabs API | elevenlabs.io/app | Voice server |
| Remove.bg API | remove.bg/dashboard | Background removal tool (optional) |

Store all API keys in your password manager. On machines that need them, load via environment variables (`.env` files or shell profile), never hardcoded in scripts.

---

## Storage Rules

1. **Password manager** is the source of truth for all credentials
2. **Config files** consume credentials (one file per credential, not scattered)
3. **Environment variables** for API keys (loaded from `.env`, never committed to git)
4. **No credentials in git** — ever. Use `.gitignore` patterns for `.env`, private keys, token files
5. **Rotate quarterly** — especially `{gateway-token}` and API keys

---

## Verification

After generating all credentials, confirm:

- [ ] `{SSH-automation-key}` connects to all target machines without password prompt
- [ ] `{SSH-automation-key}` is restricted to `{LAN-subnet}` in every `authorized_keys`
- [ ] `{SSH-github-key}` authenticates to GitHub: `ssh -T git@github.com`
- [ ] `{gateway-token}` authenticates to OpenClaw gateway
- [ ] All Syncthing devices paired and syncing
- [ ] Telegram bot responds to `/start`
- [ ] Voice server plays audio when called with `{Voice-Id}`
- [ ] All credentials stored in password manager
- [ ] No credentials committed to any git repository

---

*Created by Verbum, Ben's DA*
