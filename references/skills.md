# OpenClaw Skills System

## Overview

Skills are markdown-based instruction files that teach the agent how to perform specific tasks. They are loaded into the agent's context when relevant and provide step-by-step guidance for using tools, APIs, and workflows.

## Skills vs Tools

OpenClaw is a **tool-calling agent**. The LLM generates structured `tool_call` outputs and the runtime executes them. Tools are the execution primitives:

| Tool | Purpose | Example |
|------|---------|---------|
| `exec` | Run a shell command or script | `exec python3 /path/to/weather.py current` |
| `read` | Read a file from disk | `read /sandbox/.openclaw-data/workspace/SOUL.md` |
| `write` | Write/create a file | `write /sandbox/.openclaw-data/workspace/notes.md` |
| `browser` | Fetch/render a URL | `browser https://api.example.com/data` |

**Skills do not replace tools.** Skills are instruction files that teach the agent *when* to use which tool and with *what* arguments. The skill says "to get weather, run `exec python3 /path/to/weather.py`" — the agent then generates a `tool_call` for the `exec` tool. Both tools and skills are **OpenClaw base features** — they work the same in plain OpenClaw and in NemoClaw.

The difference in NemoClaw: every tool execution passes through **OpenShell's** policy engine (network, filesystem, binary scoping). See `references/tool-calling.md` for the full tool calling flow and `references/access-control.md` for how OpenShell policy controls tool execution.

## Skill Locations

There are two skill sources, with different access levels in NemoClaw:

| Source | Path | Access | Managed By |
|--------|------|--------|-----------|
| Bundled | `/usr/local/lib/node_modules/openclaw/skills/<name>/SKILL.md` | Read-only (Landlock) | OpenClaw package |
| Custom | `/sandbox/.openclaw-data/skills/<name>/SKILL.md` | Read-write | User |

### Discovery order

OpenClaw discovers skills from both paths (this is base OpenClaw behavior, not NemoClaw-specific). **Custom skills with the same name as a bundled skill override the bundled one** — the custom SKILL.md is loaded instead. In NemoClaw, bundled skill paths are additionally Landlock read-only. This override is useful when a bundled skill has limitations in the sandbox (e.g., the bundled `weather` skill uses HTTP which is blocked by OpenShell's proxy — a custom `weather` skill using HTTPS and better APIs replaces it).

### Configuring skill loading

In `openclaw.json` (read-only in NemoClaw — changes require rebuild):

```json5
{
  skills: {
    // Whitelist which bundled skills are available
    allowBundled: ["healthcheck", "skill-creator"],  // weather removed — native skill replaces it

    // Load custom skills from additional directories
    load: {
      extraDirs: ["/sandbox/.openclaw-data/skills"]
    },

    // Per-skill configuration (API keys, env vars)
    entries: {
      "google-search": {
        enabled: true,
        env: {
          GOOGLE_API_KEY: "key",
          GOOGLE_CX: "cx-id"
        }
      }
    }
  }
}
```

## Skill File Format

A skill is a directory containing at minimum `SKILL.md`:

```
skills/
  my-skill/
    SKILL.md          # Required: skill instructions
    references/       # Optional: supporting docs
    scripts/          # Optional: helper scripts
```

### SKILL.md structure

```markdown
---
name: my-skill
description: "One-line description for skill discovery"
homepage: https://example.com
metadata: { "openclaw": { "emoji": "icon", "requires": { "bins": ["curl"] } } }
---

# Skill Name

## When to Use
Describe trigger conditions.

## When NOT to Use
Describe exclusions.

## Commands
Step-by-step instructions with code blocks.

## Notes
Additional guidance.
```

The frontmatter is used for skill discovery and status checks. The `requires.bins` field lists binaries that must be present for the skill to show as "ready".

## CLI Commands

```bash
# List all skills with status
openclaw skills list

# Detailed info about a specific skill
openclaw skills info <name>

# Check which skills are ready vs missing requirements
openclaw skills check
```

### Skill statuses

| Status | Meaning |
|--------|---------|
| ready | All requirements met, skill available |
| blocked | Required binary or dependency missing |
| missing | Skill referenced but file not found |

## Skill Loading into Agent Context

Skills are NOT always loaded into the agent's context. The loading process:

1. At session start, OpenClaw creates a **skills snapshot** listing available skills (name + description + path)
2. This snapshot is included in the system prompt as `<available_skills>`
3. The agent reads a skill's SKILL.md file (via the `read` tool) only when the task matches the description
4. The snapshot is **cached per session** — new or modified skills require a session reset

### Important: Session cache means changes aren't live

After adding or modifying a skill, existing sessions will not see the change. Reset the session to force a fresh skills snapshot. See `references/sessions.md` for how to reset sessions.

## Creating Custom Skills

```bash
# Inside the sandbox:
mkdir -p /sandbox/.openclaw-data/skills/my-skill

# Create the SKILL.md (via SSH from host):
ssh sandbox "cat > /sandbox/.openclaw-data/skills/my-skill/SKILL.md" << 'EOF'
# My Custom Skill

## When to use
Describe when this skill should be triggered.

## Instructions
Step-by-step guidance for the agent.
EOF
```

## Network Considerations

- All external URLs **must use HTTPS** (HTTP gets 403 from the OpenShell proxy)
- Every endpoint must have an explicit network policy entry — no blanket proxy rules
- Helper scripts should use `curl` (not Python urllib) because policy is enforced per binary
- Internal services use non-443 ports with `access: full` (bypasses TLS relay)
- External services use port 443 with `tls: terminate`

## Native Skill Design Principles

Custom skills in NemoClaw should follow these rules learned from production operation:

### 1. One helper script, one exec call

Wrap all logic in a helper script (Python or bash). The agent calls it via `exec` — one tool call, one response. Never rely on the agent to chain multiple curl calls (it's 4x slower and error-prone).

```
skills/
  my-api-skill/
    SKILL.md          # Instructions — MANDATORY description with exec command
    helper.py         # Helper script — does everything in one call
  my-device-skill/
    SKILL.md
    control.sh        # Single call: action + entity
```

### 2. SOUL.md overrides SKILL.md

The agent follows SOUL.md (identity file) before any skill instructions. If SOUL.md says "use curl to API X", the agent ignores the skill that says otherwise. All skill references in SOUL.md must point to the correct helper scripts.

### 3. Description must say MANDATORY (critical — see SKILL.md rule #14)

Smaller models (Qwen3.5-35B) skip reading SKILL.md if they think they already know how. The agent decides whether to read a skill **from the description alone**. Put the exact exec command in the skill description:

```yaml
# WORKS — model reads the skill and uses exec:
description: "MANDATORY for ALL weather queries. Run: exec python3 /skills/weather/weather.py current LOCATION"

# FAILS — model thinks it already knows how, skips the skill, fabricates data:
description: "Get weather data for a location"
```

The word `MANDATORY` and the exact command are both required. Without them, the model improvises — and for data-fetching skills, improvisation means fabricated results.

### 4. Session snapshot caching

The gateway creates a skill snapshot on pod startup. If custom skills are restored later (by watchdog), the snapshot is stale. Sessions must be cleared AFTER skill restore.

### 5. Avoid name collisions with bundled skills

If a custom skill has the same name as a bundled one, remove the bundled from `allowBundled` in the Dockerfile. Otherwise the agent may load the wrong one depending on snapshot timing.

### 6. No duplicate responses

The agent may output raw tool results AND then rephrase them. Add to SOUL.md: "NEVER send two messages for one question."

### 7. TOOLS.md shapes agent behavior (SKILL.md rule #14.6)

The agent copies examples from TOOLS.md verbatim. If TOOLS.md shows entity IDs, the agent uses entity IDs in its commands. If it shows friendly names, the agent uses friendly names. Always format TOOLS.md the way you want the agent to produce output.

### 8. Never put API tokens in workspace .md files (SKILL.md rule #14.7)

Helper scripts handle authentication internally. Bare tokens in TOOLS.md or SKILL.md get copied into curl commands by the agent, causing auth failures when it copies them incorrectly or truncates them.

### 9. All workspace files must be consistent (SKILL.md rule #14.8)

SOUL.md, TOOLS.md, and SKILL.md must all point to the same scripts and use the same terminology. If SOUL.md says "use ha-control.sh with friendly names" but TOOLS.md shows entity IDs, the agent picks whichever seems familiar and fails.

## Common Custom Skills

| Skill | Pattern | What it does |
|-------|---------|-------------|
| api-skill | Python helper | Multi-source data aggregation, one exec call |
| device-control | Bash helper | Device API wrapper: action + entity in ~0.2s |
| llm-skill | SKILL.md + direct API | Read sensors + call specialized LLM |
| email | SKILL.md + curl | Email API (e.g., SendGrid), rate-limited |
| search | SKILL.md + curl | Search API (e.g., Google CSE) + fallback |

### Helper script pattern

```python
#!/usr/bin/env python3
# helper.py — called via: exec python3 helper.py <action> [args]
# Uses subprocess + curl for HTTP (respects network policy binary rules)
# Formats output for voice: spelled-out units, no symbols
```

Key: the script uses `curl` via `subprocess` (not `urllib`) because the OpenShell proxy enforces network policy per binary. Scripts inherit the policy of the calling binary (`/usr/bin/curl`).

## Bundled Skills (OpenClaw 2026.3.24)

Common bundled skills:

| Skill | Description | Requirements |
|-------|-------------|-------------|
| weather | Weather data via public APIs | curl |
| healthcheck | Security audit and hardening | — |
| skill-creator | Create/edit/audit skills | — |
| github | GitHub operations via gh CLI | gh |
| coding-agent | Delegate to Codex/Claude Code | codex or claude |
| summarize | URL/podcast summarization | — |

Most bundled skills show as "blocked" in NemoClaw because their required binaries (gh, Telegram CLI, macOS tools, etc.) are not in the sandbox image — this is by design for security (reduced attack surface). Only skills whose requirements are met show as "ready".

## What to Back Up

Custom skills at `/sandbox/.openclaw-data/skills/` should be backed up. Bundled skills are part of the OpenClaw package and don't need backup.

```bash
# Backup custom skills via SSH
ssh sandbox "tar cf - -C /sandbox/.openclaw-data/skills ." | tar xf - -C backup/skills/

# Restore
ssh sandbox "tar xf -" < backup/skills.tar
```
