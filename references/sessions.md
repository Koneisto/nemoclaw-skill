# OpenClaw Session Management

## Session Storage

Sessions are stored inside the sandbox at:

```
/sandbox/.openclaw-data/agents/<agent-id>/sessions/
  sessions.json          # Index: maps session keys to session IDs
  <uuid>.jsonl           # Per-session log (JSON Lines: messages, tool calls, model changes)
```

The default agent is `main`, so the typical path is:
```
/sandbox/.openclaw-data/agents/main/sessions/
```

## Session Keys

Sessions are identified by keys in `sessions.json`. The key format depends on the channel:

| Channel | Key Format | Example |
|---------|-----------|---------|
| Dashboard (web UI) | `agent:<agent>:<provider>:<device-uuid>` | `agent:main:openai:7ce37...` |
| HA Voice pipeline | `ha-voice` | `ha-voice` |
| Signal bridge | `sig-<phone-digits>` | `sig-358401234567` |
| Telegram bridge | `tg-<chat-id>` | `tg-123456789` |
| CLI (`--session-id`) | Exact value passed | `test`, `debug` |

Each key maps to a `sessionId` (UUID) and metadata:

```json
{
  "ha-voice": {
    "sessionId": "0793ae45-6cea-4a57-ac9a-bf581c612f7c",
    "updatedAt": 1775079788658,
    "skillsSnapshot": { ... }
  }
}
```

## Session Lifecycle

- **Created**: On first message to a new session key
- **Persistent**: Sessions accumulate messages until explicitly reset
- **Context**: Workspace files (SOUL.md, TOOLS.md, etc.) are loaded at session start
- **Skills snapshot**: Available skills are cached in the session at creation time — adding a new skill requires session reset to take effect

### Important: Workspace changes require session reset

When you modify SOUL.md, TOOLS.md, or add/change skills, existing sessions will NOT see the changes. This is because OpenClaw loads workspace files **once per session** and sets `systemSent: true` in `sessions.json` to mark that the system prompt is already delivered. After that flag is set, the session's message history contains the old SOUL.md/TOOLS.md text — restoring new files to disk has no effect until the session is deleted and a new one is created.

The session must be reset (deleted) so the next message creates a fresh session that loads the updated files.

## Resetting a Session

All SSH examples below use `ssh sandbox` shorthand — see SKILL.md "SSH Access to Sandbox" section for alias setup.

### Method 1: Inside sandbox (slash command)
```
/reset
/new
```
These are OpenClaw slash commands that clear the current session. Note: `/reset` may also trigger workspace operations depending on agent configuration.

### Method 2: Via SSH (recommended for automation)

Delete a specific session file and clean the index.

**Important:** Change `SESSION_KEY` below to match your target session key (e.g., `sig-405066104` for Signal, `tg-123456` for Telegram, `ha-voice` for HA voice). Check the session keys in `sessions.json` first if unsure.

```bash
ssh sandbox 'python3 -c "
import json, os

SESSION_KEY = \"ha-voice\"  # <-- CHANGE THIS to your target session key

with open(\"/sandbox/.openclaw-data/agents/main/sessions/sessions.json\", \"r\") as f:
    data = json.load(f)

if SESSION_KEY in data:
    sid = data[SESSION_KEY][\"sessionId\"]
    path = f\"/sandbox/.openclaw-data/agents/main/sessions/{sid}.jsonl\"
    if os.path.exists(path):
        os.remove(path)
        print(f\"Removed {sid}.jsonl\")
    del data[SESSION_KEY]
    with open(\"/sandbox/.openclaw-data/agents/main/sessions/sessions.json\", \"w\") as f:
        json.dump(data, f, indent=2)
    print(\"Session index cleaned\")
else:
    print(f\"No session found for key: {SESSION_KEY}\")
"'
```

### Method 3: Reset all sessions (after workspace restore)

After restoring workspace files (SOUL.md, skills, etc.), wipe all sessions so the agent re-reads updated files. This is the canonical procedure — see `references/workspace-backup.md` Step 13 for the full restore context.

```bash
ssh sandbox "rm /sandbox/.openclaw-data/agents/main/sessions/sessions.json /sandbox/.openclaw-data/agents/main/sessions/*.jsonl 2>/dev/null"
```

**When to use which method:**
- **Method 1** (slash command): Quick reset during interactive use
- **Method 2** (Python script): Surgical — reset one specific session by key
- **Method 3** (rm all): After workspace restore or skill deployment — forces ALL sessions to re-read workspace files

## Inspecting Sessions

### List active sessions
```bash
ssh sandbox 'cat /sandbox/.openclaw-data/agents/main/sessions/sessions.json' | python3 -m json.tool
```

### Read session messages
```bash
ssh sandbox 'cat /sandbox/.openclaw-data/agents/main/sessions/<uuid>.jsonl'
```

Each line is a JSON object with `type` field:
- `session` — session metadata (version, timestamp, cwd)
- `model_change` — model switch event
- `message` — user or assistant message (role, content, usage)
- `tool_use` / `tool_result` — tool call and result
- `thinking_level_change` — reasoning level change

### Search for specific content in sessions
```bash
ssh sandbox 'grep -l "search term" /sandbox/.openclaw-data/agents/main/sessions/*.jsonl'
```

## Session Configuration

Session behavior is configured in `openclaw.json` (read-only in NemoClaw — changes require rebuild):

```json5
{
  session: {
    scope: "per-sender",       // "per-sender" | "global"
    reset: {
      mode: "idle",            // "daily" | "idle"
      idleMinutes: 60          // auto-reset after idle period
    },
    resetTriggers: ["/new", "/reset"]
  }
}
```

## What to Back Up

Session files are generally NOT backed up — they're ephemeral conversation state. Back up workspace files (SOUL.md, TOOLS.md, etc.) and skills instead. Those define the agent's behavior; sessions are just conversation history.
