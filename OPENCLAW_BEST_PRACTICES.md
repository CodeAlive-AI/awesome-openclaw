# OpenClaw Best Practices

Compiled from official docs, community guides, and hands-on audit (Feb 2026, OpenClaw 2026.2.15).

---

## 1. Context Budget Management (CRITICAL)

The single most impactful optimization. Every token in bootstrap files loads into every conversation turn.

### Bootstrap File Size Budget

- **Default `bootstrapMaxChars`**: 20,000 characters
- **Recommended**: 10,000–15,000 characters
- **If total exceeds limit**: OpenClaw truncates files using 70/20/10 split (70% head, 20% tail, 10% truncation marker) — instructions silently get lost
- Set explicitly: `openclaw config set agents.defaults.bootstrapMaxChars 15000`

### File Size Targets

| File | Target | Purpose |
|------|--------|---------|
| AGENTS.md | < 2KB | Operational instructions only |
| SOUL.md | < 3KB | Persona, role, behavior rules |
| USER.md | < 1.5KB | User profile |
| MEMORY.md | < 5KB | Curated long-term facts |
| IDENTITY.md | < 1KB | Name, emoji, vibe |
| HEARTBEAT.md | < 4KB | Periodic task checklist |
| TOOLS.md | < 1KB | Environment-specific notes |
| **Total** | **< 15KB** | **Must fit within bootstrapMaxChars** |

### Key Rules

- **Measure total size regularly**: `wc -c ~/.openclaw/workspace/*.md`
- **No duplication across files** — SOUL.md defines behavior, MEMORY.md stores facts. Don't repeat product details in both.
- **Every line must earn its place** — if removing a line doesn't change bot behavior, remove it.
- **SKILL.md is NOT part of bootstrap** — loaded only when skill is invoked. Put detailed instructions there, not in SOUL.md.

---

## 2. AGENTS.md — Keep It Under 2KB

AGENTS.md is injected into every single conversation. It's the most expensive file in terms of tokens.

### What to include

- Session startup checklist (which files to read)
- Memory write rules (daily logs, long-term memory)
- Safety rules (don't send external without confirmation)
- Platform formatting rules (no tables in Telegram)
- Heartbeat reference (point to HEARTBEAT.md)

### What to remove

- Group chat behavior (unless you use groups)
- Camera/TTS/SSH/device notes (move to TOOLS.md)
- Email/calendar/weather checks (unless needed)
- Emoji reaction guidelines
- Generic examples and long explanations

### Template

```markdown
# AGENTS.md

## Every Session
1. Read SOUL.md, USER.md, MEMORY.md
2. Read memory/YYYY-MM-DD.md (today + yesterday)

## Memory
- Daily: memory/YYYY-MM-DD.md
- Long-term: MEMORY.md
- Write to files — mental notes don't survive restarts

## Safety
- Don't send external without confirmation
- trash > rm

## [Your specific operational rules here]
```

---

## 3. Memory System

### Architecture

| Layer | File | Purpose |
|-------|------|---------|
| Daily logs | `memory/YYYY-MM-DD.md` | Append-only, raw context |
| Long-term | `MEMORY.md` | Curated facts, decisions, preferences |
| State tracking | `memory/*.json` | Structured state (heartbeat counters, etc.) |

### Memory Search (Embeddings)

Memory search enables semantic queries over memory files. Without it, the bot can only do exact file reads.

**Setup requirements** (pick one):
- `GEMINI_API_KEY` in environment (free tier sufficient)
- `OPENAI_API_KEY` in environment
- Local embeddings via `node-llama-cpp` (~600MB model, needs RAM)

**For systemd deployments**: Add the API key to the `EnvironmentFile` referenced in your service unit (e.g., `gateway.env`), not just shell profile. The CLI and the gateway process have separate environments.

**Verify**: `openclaw memory status --deep` (pass the API key in env if running CLI manually)

### Auto-flush Before Compaction

- `compaction.mode: "safeguard"` triggers a silent agent turn before context pruning
- This reminds the model to write durable memory before old context is lost
- **Always use safeguard mode** — default in recent versions

---

## 4. Session Management

### Daily Reset

Configure automatic session reset to prevent unbounded context growth:

```bash
openclaw config set session.reset.mode daily
openclaw config set session.reset.atHour 4  # 4 AM gateway local time
```

### Session Reset After Config Changes

Bootstrap files are read only at session start. After changing SOUL.md, MEMORY.md, AGENTS.md, etc.:
- Send `/new` in chat, OR
- Restart the gateway (`systemctl restart openclaw.service`)
- CLI `openclaw message send` sends FROM the bot — it doesn't trigger `/new` processing

### Context Commands (in chat)

- `/new` — start fresh session (reloads all bootstrap files)
- `/reset` — clear current session
- `/context list` — show what's loaded in system prompt
- `/compact` — summarize older context to free tokens

---

## 5. Heartbeat Configuration

### Best Practices

- **Use HEARTBEAT.md as a checklist** — the bot reads it on each heartbeat
- **If HEARTBEAT.md is effectively empty** (only blank lines + headers), OpenClaw skips the heartbeat to save API calls
- **Track state in a JSON file** (`memory/heartbeat-state.json`) for message counting, rate limiting, topic rotation
- **Set `activeHours`** to avoid night-time messages and wasted API calls

### Cost Optimization

- Consider using a cheaper/faster model for heartbeat: `heartbeat.model` in config
- Note: session `modelOverride` can interfere with `heartbeat.model`
- Heartbeat runs in the main session — it shares context with DM conversations

### Config Example

```json
"heartbeat": {
  "every": "30m",
  "activeHours": {
    "start": "10:00",
    "end": "22:00",
    "timezone": "Asia/Almaty"
  },
  "target": "telegram"
}
```

---

## 6. File Ownership & Permissions

### The rsync Problem

When syncing files from macOS to a Linux server via rsync, files get the local Mac UID (e.g., `501:staff`) instead of the server user. The bot process (running as `openclaw`) may fail to write to these files silently.

**Fix after every rsync**:
```bash
chown -R openclaw:openclaw /home/openclaw/.openclaw/workspace/
```

Or use rsync with `--chown`:
```bash
rsync -av --chown=openclaw:openclaw ./adviser/ root@openclaw:/home/openclaw/.openclaw/workspace/
```

### Recommended Permissions

- Directories: `755` or `700`
- Config files: `600` (especially openclaw.json with tokens)
- Workspace files: `644`
- Credentials directory: `700`

---

## 7. Security Checklist

| Check | Expected | How to verify |
|-------|----------|---------------|
| Gateway bind | `loopback` (127.0.0.1) | `openclaw config get gateway.bind` |
| Public IP | None (use Tailscale) | `curl ifconfig.me` should fail |
| DM policy | `pairing` | `openclaw config get channels.telegram.dmPolicy` |
| Group policy | `allowlist` | `openclaw config get channels.telegram.groupPolicy` |
| mDNS | `off` | `openclaw config get discovery.mdns.mode` |
| Security audit | 0 critical | `openclaw security audit` |
| Node.js | 22.12.0+ | `node --version` |
| Third-party skills | Verified only | Never install unverified ClawHub skills |

### Periodic Audits

```bash
openclaw security audit --deep
openclaw doctor
```

---

## 8. Deployment (systemd)

### Service File Template

```ini
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

### Environment File (`gateway.env`)

Keep API keys here, not in openclaw.json or shell profile:

```bash
GEMINI_API_KEY=AIza...
PATH=/home/openclaw/.npm-global/bin:/usr/local/bin:/usr/bin:/bin
HOME=/home/openclaw
```

### Operations

```bash
systemctl daemon-reload              # after editing service file
systemctl restart openclaw.service   # apply config changes
systemctl status openclaw.service    # check health
journalctl -u openclaw -f            # live logs (alternative to openclaw logs)
```

---

## 9. Updating OpenClaw

```bash
# Check current version
openclaw --version

# Check latest available
npm info openclaw version

# Update
npm install -g openclaw@latest

# Restart
systemctl restart openclaw.service

# Verify
openclaw --version
openclaw doctor
```

After updating, always:
1. Run `openclaw doctor --fix` to handle schema migrations
2. Run `openclaw security audit` to check for new findings
3. Restart the gateway

---

## 10. Skills Best Practices

### Structure

```
skills/
  my-skill/
    SKILL.md          # YAML frontmatter + instructions
    (optional files)
```

### SKILL.md Size

SKILL.md is loaded only when the skill is invoked, not on every message. It's safe to put detailed instructions here (up to ~8KB is fine). Keep the detailed "how-to" in SKILL.md and only the behavioral core in SOUL.md.

### Loading Priority

1. `<workspace>/skills/` — highest (per-agent)
2. `~/.openclaw/skills/` — managed (all agents)
3. Bundled — lowest (defaults)

### Security

- **Never install unverified third-party skills** — 341 malicious ClawHub skills found in Feb 2026 (credential stealers)
- Review source code before installing any skill
- Prefer skills from known publishers

---

## 11. Monitoring & Debugging

### Key Commands

| Command | Purpose |
|---------|---------|
| `openclaw status` | Full overview |
| `openclaw status --deep` | Deep channel probes |
| `openclaw doctor` | Config validation |
| `openclaw logs --follow` | Live gateway logs |
| `openclaw sessions` | Active sessions |
| `openclaw memory search "query"` | Test memory search |
| `openclaw cron list` | Scheduled jobs |

### Health Checks

- Gateway responds: `curl -s http://127.0.0.1:18789/health`
- Telegram connected: `openclaw channels status`
- Memory search works: `openclaw memory status --deep`

---

## 12. Common Pitfalls

| Pitfall | Symptom | Fix |
|---------|---------|-----|
| Bootstrap files too large | Bot ignores some instructions | Trim files, check total < bootstrapMaxChars |
| Missing file ownership | Bot can't update memory/state | `chown -R openclaw:openclaw workspace/` after rsync |
| No embedding provider | Memory search unavailable | Set GEMINI_API_KEY in gateway.env |
| Changed bootstrap, no /new | Bot uses old persona | Send `/new` or restart gateway |
| Env vars in shell only | Gateway doesn't see API keys | Put in EnvironmentFile, not .bashrc |
| SOUL.md duplicates MEMORY.md | Wasted context tokens | Keep facts in MEMORY.md only |
| Generic AGENTS.md template | Irrelevant instructions waste tokens | Trim to <2KB, remove unused sections |

---

## Sources

- [OpenClaw Official Docs](https://docs.openclaw.ai/)
- [Agent Workspace](https://docs.openclaw.ai/concepts/agent-workspace)
- [Memory](https://docs.openclaw.ai/concepts/memory)
- [Sessions](https://docs.openclaw.ai/concepts/session)
- [Context](https://docs.openclaw.ai/concepts/context)
- [Heartbeat](https://docs.openclaw.ai/gateway/heartbeat)
- [Configuration](https://docs.openclaw.ai/gateway/configuration)
- [Security](https://docs.openclaw.ai/gateway/security)
- [AGENTS.md Template](https://docs.openclaw.ai/reference/templates/AGENTS)
- [Skills](https://docs.openclaw.ai/tools/skills)
