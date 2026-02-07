# OpenClaw on GCP: Budget Installation Guide

A tested, step-by-step guide for running [OpenClaw](https://openclaw.ai/) on a Google Cloud VM for under $20/month. Covers Telegram integration, security hardening, and real-world pitfalls discovered during deployment.

> **Tested with:** OpenClaw 2026.2.6 on GCP e2-small, Ubuntu 24.04 LTS, Node.js 22.22.0
>
> **Last updated:** February 2026

---

## What is OpenClaw?

[OpenClaw](https://github.com/openclaw/openclaw) is an open-source AI agent gateway. It connects language models (Claude, GPT, Gemini, etc.) to messaging platforms — Telegram, WhatsApp, Discord, iMessage, Slack, Signal, and more. You self-host it, it runs 24/7, and your AI agent can send/receive messages, execute tasks, run on schedules, and maintain memory across conversations.

**Key capabilities:**
- Multi-platform messaging with a single agent
- Group chats with mention-gating
- Cron jobs, heartbeat checks, webhooks
- Skills system (extensible behavior modules)
- Long-term memory (daily logs + curated facts via vector search)
- Media pipeline (images, audio, documents)
- WebChat, macOS/iOS/Android clients
- MIT license

**Official resources:**
- Docs: https://docs.openclaw.ai/
- GitHub: https://github.com/openclaw/openclaw
- Security: https://docs.openclaw.ai/gateway/security

---

## Security Warnings

Read this section before installation.

### Known Issues (February 2026)

1. **Malicious Skills on ClawHub** — 341 skills found stealing credentials via Atomic Stealer malware. **Do not** install third-party skills without reviewing source code.
2. **RCE + Command Injection CVEs** — Multiple vulnerabilities patched in recent releases. Always run the latest version.
3. **Node.js 22.12.0+** required — Earlier versions have unpatched vulnerabilities.

### Architectural Risks (by design)

These are trade-offs of running an AI agent with tool access:

- **Credentials stored as files** on the filesystem (config, OAuth tokens, channel tokens)
- **AI agent can execute shell commands** within its permission scope
- **Prompt injection is unsolved** — incoming messages can attempt to manipulate the model
- **Session transcripts** contain conversation history — readable by anyone with filesystem access

### Non-Negotiable Security Measures

1. **Gateway on loopback** — never expose port 18789 to the internet
2. **No public IP on the VM** — use Tailscale (or similar VPN) for remote access
3. **Pairing mode for DMs** — require explicit approval for new contacts
4. **Run `openclaw security audit --deep`** after every configuration change
5. **File permissions** — 700 for directories, 600 for config/credential files
6. **Modern models only** for tool-enabled agents — they resist prompt injection better
7. **Never install unverified skills** from any marketplace

---

## Prerequisites

- GCP account with active billing (free trial credits work)
- [Tailscale](https://tailscale.com/) account (free tier is sufficient)
- Telegram account (for creating a bot via @BotFather)
- `gcloud` CLI installed on your local machine
- AI provider API key (Anthropic, OpenAI, Google, etc.) or a subscription with OAuth support

### System Requirements

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| CPU | 1 vCPU | 2 vCPU |
| RAM | 2 GB (+ swap) | 4 GB |
| Disk | 20 GB SSD | 30 GB SSD |
| OS | Ubuntu 22.04 LTS | Ubuntu 24.04 LTS |
| Node.js | 22.12.0+ | Latest 22.x |

**Observed resource usage** (e2-small, idle gateway + Telegram channel):
- RAM: ~830 MB of 1.9 GB (+ 2 GB swap barely touched)
- Disk: ~6.7 GB of 30 GB
- CPU: <1% idle

---

## Table of Contents

1. [GCP Project Setup](#1-gcp-project-setup)
2. [Create the VM](#2-create-the-vm)
3. [Tailscale and Network](#3-tailscale-and-network)
4. [Install OpenClaw](#4-install-openclaw)
5. [Configuration](#5-configuration)
6. [Telegram Channel](#6-telegram-channel)
7. [Systemd Service](#7-systemd-service)
8. [Security Hardening](#8-security-hardening)
9. [Cron Jobs and Automation](#9-cron-jobs-and-automation)
10. [Management and Troubleshooting](#10-management-and-troubleshooting)

---

## 1. GCP Project Setup

```bash
# Create a project (or use an existing one)
gcloud projects create <PROJECT_ID> --name="<PROJECT_NAME>"

# Link billing
gcloud billing projects link <PROJECT_ID> --billing-account=<BILLING_ACCOUNT_ID>

# Enable Compute Engine API
gcloud services enable compute.googleapis.com --project=<PROJECT_ID>

# Set a budget alert (recommended — prevents surprise bills)
gcloud services enable billingbudgets.googleapis.com --project=<PROJECT_ID>
gcloud billing budgets create \
  --billing-account=<BILLING_ACCOUNT_ID> \
  --display-name="openclaw-budget" \
  --budget-amount=50USD \
  --filter-projects="projects/<PROJECT_ID>" \
  --threshold-rule=percent=50 \
  --threshold-rule=percent=90 \
  --threshold-rule=percent=100
```

---

## 2. Create the VM

### VM type comparison

| Type | vCPU | RAM | $/month | Verdict |
|------|------|-----|---------|---------|
| e2-micro | 2 (shared) | 1 GB | ~$7 | Too little RAM. Gateway OOMs. |
| **e2-small** | **2 (shared)** | **2 GB** | **~$17** | **Works with swap. Tested.** |
| e2-medium | 2 (shared) | 4 GB | ~$28 | Comfortable. No swap needed. |
| e2-standard-2 | 2 | 8 GB | ~$54 | Overkill for OpenClaw alone. |

**Prices include 30 GB SSD.** `us-central1` region. Prices may vary.

### Create the VM

```bash
gcloud compute instances create openclaw \
  --project=<PROJECT_ID> \
  --zone=us-central1-a \
  --machine-type=e2-small \
  --image-family=ubuntu-2404-lts-amd64 \
  --image-project=ubuntu-os-cloud \
  --boot-disk-size=30GB \
  --boot-disk-type=pd-ssd \
  --tags=allow-ssh \
  --metadata=enable-oslogin=TRUE
```

### Add swap (required for e2-small)

With 2 GB RAM the gateway uses ~830 MB at idle, but spikes during model calls. Swap prevents OOM kills.

```bash
# SSH in first
gcloud compute ssh openclaw --zone=us-central1-a --project=<PROJECT_ID>

# Create 2 GB swap
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

# Verify: should show 2 GB swap
free -h
```

### Firewall for initial SSH

```bash
gcloud compute firewall-rules create allow-ssh \
  --project=<PROJECT_ID> \
  --direction=INGRESS \
  --priority=1000 \
  --network=default \
  --action=ALLOW \
  --rules=tcp:22 \
  --source-ranges=0.0.0.0/0 \
  --target-tags=allow-ssh
```

### Estimated monthly cost

| Component | Cost |
|-----------|------|
| e2-small VM | ~$12 |
| 30 GB SSD | ~$5 |
| Cloud NAT | ~$1 |
| **Total** | **~$18/mo** |

---

## 3. Tailscale and Network

### Why Tailscale?

Tailscale creates an encrypted mesh VPN between your devices. Once configured, you can SSH to the VM by hostname (e.g., `ssh root@openclaw`) without a public IP. Free for personal use.

### Install Tailscale on the VM

```bash
# On the VM (via gcloud SSH while public IP still exists)
curl -fsSL https://tailscale.com/install.sh | sh

# Start Tailscale — this prints an auth URL
sudo tailscale login --ssh
# Open the URL in your browser to authorize the machine
```

> **Gotcha:** `tailscale up --ssh` blocks waiting for browser auth and can hang in non-interactive SSH sessions. Use `tailscale login --ssh` instead — it prints the URL and exits.

### Install Tailscale on your local machine

Download from https://tailscale.com/download. Log in with the same account.

### Cloud NAT (outbound internet without public IP)

After removing the public IP, the VM needs Cloud NAT for outgoing traffic (API calls, npm installs, Telegram polling).

```bash
# Create a Cloud Router
gcloud compute routers create nat-router \
  --project=<PROJECT_ID> \
  --network=default \
  --region=us-central1

# Create Cloud NAT
gcloud compute routers nats create nat-config \
  --project=<PROJECT_ID> \
  --router=nat-router \
  --region=us-central1 \
  --nat-all-subnet-ip-ranges \
  --auto-allocate-nat-external-ips
```

### Remove the public IP

```bash
gcloud compute instances delete-access-config openclaw \
  --zone=us-central1-a \
  --project=<PROJECT_ID> \
  --access-config-name="external-nat"
```

### Verify

```bash
# From your local machine
tailscale status                # Should list "openclaw"
ssh root@openclaw               # Should connect via Tailscale

# On the VM — verify outbound works via NAT
curl -sI https://google.com | head -3
```

> **Gotcha:** First Tailscale SSH connection may require browser re-auth. The VM will print an auth URL — open it to approve.

---

## 4. Install OpenClaw

### Create a dedicated user

Run as root on the VM:

```bash
useradd -m -s /bin/bash openclaw
```

### Install Node.js 22

```bash
curl -fsSL https://deb.nodesource.com/setup_22.x | bash -
apt-get install -y nodejs
node --version   # Must be v22.12.0 or later
```

### Install OpenClaw

Switch to the `openclaw` user:

```bash
su - openclaw
curl -fsSL https://openclaw.ai/install.sh | bash
```

The installer will print a PATH warning. Fix it:

```bash
# The installer puts the binary in ~/.npm-global/bin/, not ~/.openclaw/bin/
echo 'export PATH="$HOME/.npm-global/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
openclaw --version
```

> **Gotcha:** The installer's PATH suggestion in the output is wrong — it concatenates the existing `$PATH` into the export line. Use the simplified version above.

### Interactive onboarding (optional)

If you have a direct TTY (not piped SSH), run:

```bash
openclaw onboard --install-daemon
```

This wizard walks you through provider auth, gateway setup, and channel config.

> **Gotcha:** Onboarding requires an interactive terminal. It fails via `ssh host "command"` or in scripts. If you're setting up headless, skip onboarding and configure manually (next section).

---

## 5. Configuration

If onboarding ran successfully, skip this section. Otherwise, create the config manually.

### Create directory structure

```bash
su - openclaw
mkdir -p ~/.openclaw/agents/main/agent
mkdir -p ~/.openclaw/agents/main/sessions
mkdir -p ~/.openclaw/credentials
mkdir -p ~/.openclaw/workspace/memory
mkdir -p ~/.openclaw/workspace/skills
```

> **Gotcha:** The `agents/main/sessions` directory is not auto-created. Without it, `openclaw doctor` reports a critical error.

### Create config file

```bash
cat > ~/.openclaw/openclaw.json << 'EOF'
{
  "agents": {
    "defaults": {
      "model": {
        "primary": "anthropic/claude-opus-4-6"
      },
      "workspace": "/home/openclaw/.openclaw/workspace",
      "compaction": {
        "mode": "safeguard"
      }
    }
  },
  "channels": {
    "telegram": {
      "enabled": true,
      "botToken": "<YOUR_TELEGRAM_BOT_TOKEN>",
      "dmPolicy": "pairing",
      "groupPolicy": "allowlist",
      "streamMode": "partial",
      "groups": { "*": { "requireMention": true } }
    }
  },
  "gateway": {
    "port": 18789,
    "mode": "local",
    "bind": "loopback",
    "auth": {
      "mode": "token",
      "token": "<GENERATE_WITH_openssl_rand_-hex_24>"
    }
  },
  "discovery": {
    "mdns": { "mode": "off" }
  },
  "logging": {
    "redactSensitive": "tools"
  }
}
EOF
chmod 600 ~/.openclaw/openclaw.json
```

Replace:
- `<YOUR_TELEGRAM_BOT_TOKEN>` — from @BotFather
- `<GENERATE_WITH_openssl_rand_-hex_24>` — run `openssl rand -hex 24` to generate

**Model options:**
- Anthropic: `anthropic/claude-opus-4-6`, `anthropic/claude-sonnet-4`
- OpenAI: `openai/gpt-5.2`, `openai-codex/gpt-5.2` (subscription OAuth)
- Google: `google/gemini-2.5-pro`

For subscription-based OAuth (OpenAI Codex, Claude subscription), you need interactive onboarding or manual `auth-profiles.json` setup. API key auth is simpler for headless setups.

### API key auth (headless)

For API key providers, set keys via environment variables:

```bash
cat > ~/.openclaw/gateway.env << 'EOF'
ANTHROPIC_API_KEY=sk-ant-...
PATH=/home/openclaw/.npm-global/bin:/usr/local/bin:/usr/bin:/bin
HOME=/home/openclaw
EOF
chmod 600 ~/.openclaw/gateway.env
```

---

## 6. Telegram Channel

### Create a bot

1. Open [@BotFather](https://t.me/BotFather) in Telegram
2. Send `/newbot`, follow the prompts
3. Copy the bot token

Optional BotFather settings:
- `/setjoingroups` — control whether the bot can be added to groups
- `/setprivacy` — message visibility in groups (disable for `@mention` mode)

### First contact (pairing)

With `dmPolicy: "pairing"` (default):

1. Send any message to your bot in Telegram
2. The bot replies with a **pairing code** (valid 1 hour)
3. Approve on the server:

```bash
openclaw pairing approve telegram <CODE>
```

After approval, the bot responds to your messages.

**Alternative — allowlist** (skip pairing):

```bash
openclaw config set channels.telegram.dmPolicy allowlist
openclaw config set channels.telegram.allowFrom '["YOUR_TELEGRAM_USER_ID"]'
```

Find your user ID: send a message to the bot and check `openclaw logs --follow` — it logs the sender ID.

---

## 7. Systemd Service

If `openclaw onboard --install-daemon` didn't run (headless setup), create the service manually:

```ini
# /etc/systemd/system/openclaw.service
[Unit]
Description=OpenClaw Gateway
After=network.target

[Service]
Type=simple
User=openclaw
Group=openclaw
WorkingDirectory=/home/openclaw
EnvironmentFile=/home/openclaw/.openclaw/gateway.env
ExecStart=/home/openclaw/.npm-global/bin/openclaw gateway run
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

> **Gotcha:** The binary path is `~/.npm-global/bin/openclaw`, not `~/.openclaw/bin/openclaw`. The installer uses npm's global prefix, not the `~/.openclaw` directory.

```bash
sudo systemctl daemon-reload
sudo systemctl enable openclaw
sudo systemctl start openclaw
sudo systemctl status openclaw
```

Verify the gateway is listening:

```bash
ss -tlnp | grep 18789
# Should show: 127.0.0.1:18789 LISTEN
```

---

## 8. Security Hardening

### Run security audit

```bash
openclaw security audit --deep
openclaw security audit --fix    # Auto-fixes common issues
```

> **Gotcha:** The installer creates `~/.openclaw/credentials/` with 775 permissions. The audit flags this as CRITICAL. Run `--fix` or manually `chmod 700` all directories.

### File permissions checklist

```bash
chmod 700 ~/.openclaw
chmod 700 ~/.openclaw/credentials
chmod 700 ~/.openclaw/agents
chmod 700 ~/.openclaw/agents/main
chmod 700 ~/.openclaw/agents/main/agent
chmod 600 ~/.openclaw/openclaw.json
chmod 600 ~/.openclaw/gateway.env
chmod 600 ~/.openclaw/credentials/*
chmod 600 ~/.openclaw/agents/*/agent/auth-profiles.json
```

### Recommended hardening config

```json5
{
  gateway: {
    mode: "local",
    bind: "loopback",          // Never change to 0.0.0.0
    auth: { mode: "token" }    // Always require auth
  },
  channels: {
    telegram: {
      dmPolicy: "pairing",     // Or "allowlist"
      groupPolicy: "allowlist",
      groups: { "*": { requireMention: true } }
    }
  },
  discovery: {
    mdns: { mode: "off" }      // Prevents info disclosure on LAN
  },
  logging: {
    redactSensitive: "tools"   // Redact tool args in logs
  }
}
```

### Sandbox mode (optional, requires Docker)

Isolate tool execution in containers:

```bash
sudo apt-get install -y docker.io
sudo usermod -aG docker openclaw
# Log out and back in for group to take effect
```

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "all",
        scope: "agent",
        workspaceAccess: "rw"   // "ro" or "none" for stricter isolation
      }
    }
  }
}
```

### Tool access control

Restrict dangerous tools for non-owner agents:

```json5
{
  agents: {
    defaults: {
      tools: {
        deny: ["browser", "web_fetch", "exec"]
      }
    }
  }
}
```

### Skills safety

- **Never install skills from untrusted sources** (Feb 2026: 341 malicious ClawHub skills found)
- Review `SKILL.md` contents before installing any skill
- Skills can read environment variables and execute code — treat as trusted code
- Per-agent skills: `~/.openclaw/workspace/skills/`
- Shared skills: `~/.openclaw/skills/`

### Credential management

- Store API keys in `gateway.env` (permissions 600), not inline in config
- Use `EnvironmentFile=` in systemd, not `Environment=`
- Rotate all tokens after any suspected compromise
- Never commit credentials to git

### Incident response

If you suspect compromise:

1. **Stop:** `systemctl stop openclaw`
2. **Isolate:** Verify `gateway.bind` is `loopback`
3. **Rotate:** Gateway token, model API keys, channel bot tokens
4. **Review:** Logs at `/tmp/openclaw/openclaw-YYYY-MM-DD.log` and session transcripts at `~/.openclaw/agents/<agentId>/sessions/*.jsonl`
5. **Audit:** `openclaw security audit --deep`
6. **Restart** only after audit is clean

---

## 9. Cron Jobs and Automation

### Create scheduled jobs

```bash
# Daily morning briefing at 10:00 (delivered to Telegram)
openclaw cron add \
  --name "Morning Briefing" \
  --cron "0 10 * * *" \
  --tz "America/New_York" \
  --session isolated \
  --message "Good morning. Check MEMORY.md and summarize today's priorities." \
  --deliver \
  --channel telegram \
  --to "<YOUR_TELEGRAM_USER_ID>"

# Weekly review every Monday at 11:00
openclaw cron add \
  --name "Weekly Review" \
  --cron "0 11 * * 1" \
  --tz "America/New_York" \
  --session isolated \
  --message "Weekly review. Read memory logs from the past 7 days and summarize progress." \
  --deliver \
  --channel telegram \
  --to "<YOUR_TELEGRAM_USER_ID>"
```

### Manage cron jobs

```bash
openclaw cron list              # List all jobs with next run time
openclaw cron run <JOB_ID>      # Trigger manually
openclaw cron remove <JOB_ID>   # Delete a job
```

### Heartbeat

OpenClaw runs a heartbeat check every 30 minutes by default. Configure checks in `~/.openclaw/workspace/HEARTBEAT.md`. If nothing requires attention, the agent responds with `HEARTBEAT_OK` (no notification sent).

---

## 10. Management and Troubleshooting

### Essential commands

```bash
# Status and health
openclaw status               # Overview of gateway, channels, sessions
openclaw status --deep        # With live probes
openclaw health               # System health check
openclaw doctor               # Validate config, detect issues, migrate legacy
openclaw doctor --fix         # Auto-fix detected issues

# Gateway control
openclaw gateway status
openclaw gateway restart
openclaw gateway stop
openclaw gateway --force      # Kill and restart

# Channels
openclaw channels status      # Connection status for all channels
openclaw logs --follow        # Live gateway logs

# Security
openclaw security audit
openclaw security audit --deep
openclaw security audit --fix

# Memory
openclaw memory search "query"

# Update
curl -fsSL https://openclaw.ai/install.sh | bash   # Re-run installer
openclaw doctor && openclaw gateway restart          # Post-update
```

### Dashboard access

The Control UI runs at `http://127.0.0.1:18789/`. Access it via:

**SSH tunnel:**
```bash
ssh -N -L 18789:127.0.0.1:18789 user@gateway-host
# Open http://localhost:18789/ in your browser
```

**Tailscale Serve:**
```json5
{
  gateway: {
    tailscale: { mode: "serve" }
  }
}
```
Access at `https://<hostname>.<tailnet>.ts.net/`

### Troubleshooting

**Gateway won't start:**
```bash
journalctl -u openclaw --no-pager -n 50     # Check systemd logs
openclaw gateway --force --verbose           # Force restart with debug output
```

**Telegram not responding:**
```bash
openclaw channels status       # Check channel state
openclaw logs --follow         # Watch for errors
```

Check that the bot token is correct and that no other instance is polling the same token (Telegram allows only one poller per bot token).

**Tailscale SSH needs re-auth:**
```bash
sudo tailscale login --ssh     # Prints a new auth URL
```

**No internet after removing public IP:**

Verify Cloud NAT is configured:
```bash
curl -sI https://google.com | head -1    # Should return HTTP/2 301
```

If it fails, check that the Cloud Router and NAT exist in the same region as the VM.

**Bootstrap files not taking effect:**

Send `/new` in the chat to start a fresh session. Bootstrap files (SOUL.md, USER.md, MEMORY.md, etc.) are only read at session start, not mid-conversation.

**Legacy migration (from Moltbot/Clawdbot):**

```bash
openclaw doctor --fix    # Auto-migrates legacy paths and config keys
```

---

## Key Paths

| Path | Purpose | Permissions |
|------|---------|-------------|
| `~/.openclaw/` | Config root | 700 |
| `~/.openclaw/openclaw.json` | Main config | 600 |
| `~/.openclaw/gateway.env` | Environment variables | 600 |
| `~/.openclaw/credentials/` | Channel tokens, OAuth | 700 |
| `~/.openclaw/agents/<id>/` | Agent data | 700 |
| `~/.openclaw/agents/<id>/sessions/` | Conversation transcripts | 700 |
| `~/.openclaw/workspace/` | Agent workspace (bootstrap files, skills, memory) | 700 |
| `~/.npm-global/bin/openclaw` | CLI binary (npm global install) | — |
| `/tmp/openclaw/` | Logs | — |

## Bootstrap Files

Place these in `~/.openclaw/workspace/` to customize the agent:

| File | Purpose |
|------|---------|
| `SOUL.md` | Persona, tone, behavioral boundaries |
| `USER.md` | Information about you (the operator) |
| `IDENTITY.md` | Agent name, emoji, vibe |
| `MEMORY.md` | Long-term curated memory |
| `AGENTS.md` | Operational instructions |
| `TOOLS.md` | Tool usage guidelines |
| `HEARTBEAT.md` | Periodic check tasks |

Empty files are skipped. Send `/new` in chat after editing to reload.

---

## Gotchas Summary

Issues discovered during real deployment:

| Issue | Cause | Fix |
|-------|-------|-----|
| `openclaw: command not found` | Installer uses `~/.npm-global/bin/`, not `~/.openclaw/bin/` | Add `~/.npm-global/bin` to PATH |
| Onboarding fails via SSH | Requires interactive TTY (`/dev/tty`) | Configure manually or use a direct terminal |
| OAuth auth fails headless | OAuth flow requires browser redirect | Use API key auth for headless, or copy `auth-profiles.json` from another machine |
| `credentials dir writable by others` | Installer creates dirs with 775 | `chmod 700 ~/.openclaw/credentials` |
| `Session store dir missing` | Not auto-created on fresh install | `mkdir -p ~/.openclaw/agents/main/sessions` |
| Two bots fight over token | Telegram allows one poller per token | Stop old instance before starting new one |
| VM OOM on e2-small | 2 GB RAM, gateway spikes during inference | Add 2 GB swap file |

---

## Further Reading

- [OpenClaw Documentation](https://docs.openclaw.ai/)
- [Security Guide](https://docs.openclaw.ai/gateway/security)
- [Getting Started](https://docs.openclaw.ai/start/getting-started)
- [Updating](https://docs.openclaw.ai/install/updating)
- [Cron Jobs](https://docs.openclaw.ai/automation/cron-jobs)
- [Skills](https://docs.openclaw.ai/tools/skills)
- [Telegram Channel](https://docs.openclaw.ai/channels/telegram)
- [Tailscale Setup](https://docs.openclaw.ai/gateway/tailscale)

---

## License

This guide is provided as-is under the [MIT License](https://opensource.org/licenses/MIT). Not affiliated with OpenClaw.
