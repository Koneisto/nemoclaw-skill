# OpenClaw Configuration Reference (openclaw.json)

## Overview

`openclaw.json` is the agent configuration file inside the NemoClaw sandbox. It controls model settings, gateway behavior, workspace paths, channel integrations, and agent identity.

**Critical**: In NemoClaw, `openclaw.json` is **read-only** (Landlock protected). Changes require a full sandbox rebuild via `nemoclaw onboard`. Plan configuration carefully before onboarding.

**Format**: JSON5 (supports comments and trailing commas). All fields optional with safe defaults.

**Location**: `/sandbox/.openclaw/openclaw.json` (or `/sandbox/.openclaw-data/openclaw.json`)

## Model Configuration

The most commonly needed settings for NemoClaw:

```json5
{
  agents: {
    defaults: {
      // Primary model and fallbacks
      model: {
        primary: "provider/model",
        fallbacks: ["provider/model"]
      },

      // Context and token limits
      contextTokens: 200000,      // max context window
      timeoutSeconds: 600,        // per-request timeout

      // Model parameters
      params: {
        cacheRetention: "long"    // "long" | "short" | "none"
      },

      // Thinking/reasoning defaults
      thinkingDefault: "adaptive", // "off"|"minimal"|"low"|"medium"|"high"|"xhigh"|"adaptive"

      // Model aliases (built-in examples)
      models: {
        "provider/model": {
          alias: "shortname",
          params: {}
        }
      }
    }
  }
}
```

### Built-in Model Aliases

| Alias | Resolves To |
|-------|-------------|
| `opus` | `anthropic/claude-opus-4-6` |
| `sonnet` | `anthropic/claude-sonnet-4-6` |
| `gpt` | `openai/gpt-5.4` |
| `gpt-mini` | `openai/gpt-5-mini` |
| `gemini` | `google/gemini-3.1-pro-preview` |
| `gemini-flash` | `google/gemini-3-flash-preview` |

### NemoClaw Inference Provider (example config)

In NemoClaw, the model provider uses `inference.local` (routed by OpenShell), not the host vLLM URL directly:

```json5
{
  agents: {
    defaults: {
      model: { primary: "inference/model" },
      userTimezone: "America/New_York",
      timeFormat: "24"
    }
  },
  models: {
    mode: "merge",
    providers: {
      "inference": {
        baseUrl: "https://inference.local/v1",  // OpenShell routes this to vLLM
        apiKey: "unused",                        // credential injection handles this
        api: "openai-completions",
        models: [{
          id: "model",
          name: "inference/model",
          reasoning: false,
          input: ["text"],
          cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
          contextWindow: 65536,
          maxTokens: 8192
        }]
      }
    }
  }
}
```

Key points:
- Provider name is `inference` (not `vllm-local`) inside the config
- BaseUrl is `https://inference.local/v1` — OpenShell gateway routes this to the actual vLLM host
- The sandbox never knows the real vLLM endpoint
- `apiKey: "unused"` because OpenShell handles credential injection

## Gateway Configuration

Example gateway config:

```json5
{
  gateway: {
    mode: "local",
    auth: {
      token: "<GATEWAY_TOKEN>"           // changes on rebuild!
    },
    controlUi: {
      allowInsecureAuth: true,          // for LAN HTTP access
      dangerouslyDisableDeviceAuth: true, // skip device pairing
      allowedOrigins: [
        "http://127.0.0.1:18789",
        "http://<HOST_LAN_IP>:18789",     // LAN access
        "http://<HOSTNAME>.local:18789",
        "http://<HOSTNAME>:18789"
      ]
    },
    http: {
      endpoints: {
        chatCompletions: { enabled: true } // for HA Bridge
      }
    },
    trustedProxies: ["127.0.0.1", "::1"]
  }
}
```

Key: `allowedOrigins` must include every host/IP that accesses the dashboard over LAN. Set via `CHAT_UI_EXTRA_ORIGINS` env var in Dockerfile patches.

## Agent Configuration

```json5
{
  agents: {
    defaults: {
      workspace: "~/.openclaw/workspace",
      userTimezone: "America/New_York",
      timeFormat: "24",
      imageMaxDimensionPx: 1200
    },
    list: [{
      id: "main",
      default: true,
      name: "Main Agent",
      identity: {
        name: "MyAssistant",
        theme: "helpful AI assistant",
        emoji: "🤖"
      }
    }]
  }
}
```

### Per-Agent Model Overrides

```json5
{
  agents: {
    list: [{
      id: "main",
      model: "vllm-local/model",
      thinkingDefault: "high",
      params: { cacheRetention: "long" }
    }]
  }
}
```

## Channel Configurations

### Telegram Bridge

```json5
{
  channels: {
    telegram: {
      enabled: true,
      botToken: "token",              // or env: TELEGRAM_BOT_TOKEN
      dmPolicy: "allowlist",          // "pairing"|"allowlist"|"open"|"disabled"
      allowFrom: ["tg:userId"],
      historyLimit: 50,
      streaming: "off",              // "off"|"partial"|"block"|"progress"
      actions: {
        reactions: true,
        sendMessage: true
      }
    }
  }
}
```

### Signal Bridge

```json5
{
  channels: {
    signal: {
      enabled: true,
      account: "+1XXXXXXXXXX",
      dmPolicy: "allowlist",
      allowFrom: ["+1XXXXXXXXXX"],
      historyLimit: 50,
      reactionNotifications: "off"
    }
  }
}
```

### Discord Bridge

```json5
{
  channels: {
    discord: {
      enabled: true,
      token: "token",                // or env: DISCORD_BOT_TOKEN
      dmPolicy: "allowlist",
      allowFrom: ["userId"],
      historyLimit: 20,
      textChunkLimit: 2000,
      streaming: "off"
    }
  }
}
```

### DM Policies

| Policy | Behavior |
|--------|----------|
| `pairing` | Unknown senders get one-time approval code (expires 1 hour) |
| `allowlist` | Only approved senders via `allowFrom` |
| `open` | All inbound DMs permitted |
| `disabled` | Ignore all DMs |

## Session Management

```json5
{
  session: {
    scope: "per-sender",            // "per-sender" | "global"
    reset: {
      mode: "idle",                  // "daily" | "idle"
      idleMinutes: 60
    },
    resetTriggers: ["/new", "/reset"]
  }
}
```

## Skills Configuration

```json5
{
  skills: {
    load: {
      extraDirs: ["/sandbox/.openclaw-data/skills"]
    },
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

Skills are loaded from `/sandbox/.openclaw-data/skills/<name>/SKILL.md`.

## Plugin Configuration (Memory & Dreaming)

Plugin settings go under `plugins.entries.<plugin-id>`. Since `openclaw.json` is Landlock read-only, plugin config must be set in the Dockerfile at build time.

### Enabling Dreaming (REM)

Dreaming promotes short-term recall entries to long-term memory (`MEMORY.md`) via a nightly cron job. Since `openclaw.json` is Landlock read-only, enabling dreaming requires a sandbox rebuild — see SKILL.md rule #19. Configure via the `memory-core` plugin:

```json5
{
  plugins: {
    entries: {
      "memory-core": {
        config: {
          dreaming: {
            enabled: true,
            frequency: "0 3 * * *"    // cron expression (default: 3 AM)
          }
        }
      }
    }
  }
}
```

**In the Dockerfile** (line that sets `plugins`):
```python
'plugins': {'entries': {'memory-core': {'config': {'dreaming': {'enabled': True, 'frequency': '0 3 * * *'}}}}}
```

When enabled, OpenClaw auto-creates a `Memory Dreaming Promotion` cron job. This is separate from user-defined cron jobs — it appears in `openclaw cron list` but is managed by the plugin.

### Memory CLI Commands

```bash
openclaw memory status              # Show index, dreaming state, recall stats
openclaw memory status --deep       # Probe embedding provider readiness
openclaw memory status --fix        # Repair stale recall locks
openclaw memory index --force       # Force full reindex of memory files
openclaw memory promote --dry-run   # Preview short-term → long-term promotions
openclaw memory promote --apply     # Execute promotions into MEMORY.md
openclaw memory rem-harness --json  # Preview REM reflections without writing
openclaw memory search "query"      # Search memory files
```

### Memory Architecture

| Component | Location | Purpose |
|-----------|----------|---------|
| `MEMORY.md` | workspace root | Long-term curated memory (core file, UI shows "missing" without it) |
| `memory/YYYY-MM-DD.md` | `workspace/memory/` | Daily session notes |
| `short-term-recall.json` | `workspace/memory/.dreams/` | Short-term recall store (concept tags, scores, query hashes) |
| Memory index | `~/.openclaw/memory/main.sqlite` | FTS + vector search index |

### Session Maintenance

Sessions accumulate in `sessions.json` (one per conversation/interaction). Clean up periodically:

```bash
openclaw sessions cleanup --dry-run          # Preview what would be pruned
openclaw sessions cleanup --enforce          # Execute cleanup
openclaw sessions cleanup --enforce --all-agents  # All agents
```

Recommended: add as a weekly cron job inside the sandbox:
```bash
openclaw cron add \
  --name "Session cleanup" \
  --cron "0 4 * * 0" \
  --tz "Europe/Helsinki" \
  --system-event "Run session maintenance: openclaw sessions cleanup --enforce --all-agents"
```

## Cron Jobs

Cron jobs are managed via `openclaw cron add` (not by editing config directly). They are stored in `/sandbox/.openclaw-data/cron-jobs.json`.

## Compaction (Context Management)

```json5
{
  agents: {
    defaults: {
      compaction: {
        mode: "default",             // "default" | "safeguard"
        reserveTokensFloor: 24000,
        timeoutSeconds: 900
      }
    }
  }
}
```

Use `/compact` inside the sandbox to manually trigger context compaction (recovers 80-90%).

## Heartbeat Configuration

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m",               // interval: ms/s/m/h
        target: "telegram",         // delivery target
        prompt: "Check system status",
        ackMaxChars: 300
      }
    }
  }
}
```

## Key Fields for NemoClaw Operations

| Field | Why It Matters | Impact of Change |
|-------|---------------|------------------|
| `contextTokens` | Controls model context window | Rebuild required |
| `maxTokens` (in model def) | Max output tokens per response | Rebuild required |
| `gateway.port` | Dashboard access port | Rebuild required |
| `gateway.bind` | LAN accessibility | Rebuild required |
| `gateway.controlUi.allowedOrigins` | Which hosts can access dashboard | Rebuild required |
| `agents.defaults.userTimezone` | Timezone for cron and timestamps | Rebuild required |
| `channels.*` | Messaging bridge configuration | Rebuild required |
| `skills.entries` | Skill API keys and settings | Rebuild required |

**All changes require rebuild** because `openclaw.json` is Landlock read-only inside NemoClaw sandboxes. This is why the Dockerfile patches in `~/.nemoclaw/source/` are the primary mechanism for configuration changes — they modify the file at build time before Landlock locks it down.

## Environment Variable Substitution

Config values support `${VARIABLE_NAME}` syntax for env var injection:

```json5
{
  channels: {
    telegram: {
      botToken: "${TELEGRAM_BOT_TOKEN}"
    }
  }
}
```

This allows sensitive values to be set via environment at container startup rather than hardcoded in the config file.
