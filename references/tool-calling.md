# Tool Calling in OpenClaw and NemoClaw

## Overview

OpenClaw is a **tool-calling agent**: the LLM generates structured `tool_call` outputs (JSON-schema function calls), and the OpenClaw runtime executes them. Tools are the execution primitives. Skills (SKILL.md files) teach the agent *when* and *how* to use tools — they do not replace tools.

OpenShell (the security runtime underneath NemoClaw) wraps every tool execution with sandbox policy enforcement. NemoClaw orchestrates the deployment of OpenClaw into OpenShell and provides the CLI/preset system to manage policies. The tool-calling interface is identical to plain OpenClaw; the difference is what OpenShell enforces between the tool call and its execution.

## Built-in Tools

OpenClaw provides these core tools to the agent (as of OpenClaw 2026.3.24):

| Tool | What It Does | NemoClaw Enforcement |
|------|-------------|---------------------|
| `exec` | Run a shell command or script | Binary identified by proxy; network policy scoped per binary |
| `read` | Read a file from disk | Landlock: `/sandbox`, `/tmp` (rw), system paths (ro) |
| `write` | Write/create a file | Landlock: `/sandbox`, `/tmp` only (rw) |
| `browser` | Fetch/render a URL | Network policy: host:port must be in active policy |
| `search` | Web search via configured provider | Network policy: search provider endpoint must be allowed |
| `mcp` | Call an MCP server tool | Supported but **not recommended** — see note below |

Additional tools (e.g., `image`, `code_interpreter`) may exist depending on OpenClaw version and plugin configuration. The table above covers the tools relevant to NemoClaw operations.

**MCP note:** OpenClaw supports MCP natively (`mcp.servers` in openclaw.json, `/mcp` slash command, stdio-to-HTTP bridge). NemoClaw can use MCP if the `mcp-services` preset is applied and `mcp.servers` is configured. However, **for 95% of use cases, native skills (exec + helper scripts via curl/python3) are simpler, faster, and more debuggable.** Only consider MCP if native skills genuinely can't solve your problem. Reasons to prefer native skills:
- One exec call, one response — no bridge layer, no stdio-to-HTTP proxy
- Direct HTTP endpoints with per-binary network policies are easier to audit
- MCP bridge (`mcp-http-bridge.cjs`) adds a failure point inside the locked-down sandbox
- Bridge startup is slow (~2-5s) compared to exec + curl (~0.2s)

## Tool Call Flow

```
1. User sends message (Signal, Telegram, dashboard)
2. LLM generates tool_call: { name: "exec", arguments: { command: "python3 /path/to/helper.py current" } }
3. OpenClaw runtime validates tool_call schema (name exists, arguments match)
4. OpenClaw dispatches to tool implementation
5. ─── NemoClaw enforcement layer ───
   a. exec → OpenShell identifies the target binary (e.g., /usr/bin/python3 → /usr/bin/curl)
   b. read/write → Landlock checks path against filesystem policy
   c. browser/search → Proxy evaluates destination host:port against network policy
   d. If denied: logged at WARN level, tool returns error to LLM
   e. If allowed: tool executes normally
6. Tool result returned to LLM for next turn
```

## Skills vs Tools

| Concept | What It Is | Example |
|---------|-----------|---------|
| **Tool** | Execution primitive the LLM can call | `exec`, `read`, `write`, `browser` |
| **Skill** | Markdown instruction file that teaches the agent | `weather/SKILL.md` — tells agent to run `exec python3 weather.py` |
| **Helper script** | Script invoked via `exec` tool | `weather.py` — aggregates 6 weather APIs in one call |

The relationship:
1. Agent discovers skills via snapshot (name + description from SKILL.md frontmatter)
2. When task matches, agent reads SKILL.md via `read` tool
3. SKILL.md instructs: "Run `exec python3 /path/to/weather.py current vantaa`"
4. Agent generates `tool_call` for the `exec` tool with that command
5. OpenShell enforces network policy on the resulting outbound connections

**Skills control agent behavior at the instruction level (OpenClaw). OpenShell controls tool execution at the infrastructure level.** Both layers matter — a permissive skill with a restrictive policy still blocks, and a restrictive skill with a permissive policy still limits (because the agent follows instructions).

## What Running Inside NemoClaw Changes

In plain OpenClaw, every tool call executes unrestricted. When OpenClaw runs inside NemoClaw (i.e., inside an OpenShell sandbox), **OpenShell** enforces five security layers around tool execution:

| Layer | Provided By | What It Controls |
|-------|-------------|-----------------|
| **Network boundary** | OpenShell (proxy at 10.200.0.1:3128) | Which hosts/ports `exec`/`browser`/`search` can reach |
| **Filesystem boundary** | OpenShell (Landlock LSM) | Which paths `read`/`write` can access |
| **Binary scoping** | OpenShell (per-binary policy entries) | Which executable is allowed to reach which endpoint |
| **Credential separation** | OpenShell (host-side storage + proxy injection) | Tools never see raw API keys |
| **Logging** | OpenShell (policy decision log) | Allow/deny with host, port, binary, method, path |

**NemoClaw's role** is orchestration: it deploys OpenClaw into OpenShell, provides the preset system for managing policies, and offers the CLI/TUI for operator interaction. The security enforcement itself is OpenShell.

See `references/access-control.md` for how to use these mechanisms as an access control model and how to view the audit trail.

## Model Compatibility for Tool Calling

OpenClaw (the base platform) requires models that support **structured function/tool calling** (generating valid JSON tool_call objects). This requirement applies whether running plain OpenClaw or inside NemoClaw — NemoClaw doesn't change the tool calling interface. Not all local inference backends handle this correctly:

| Backend | Tool Call Support | Notes |
|---------|------------------|-------|
| **vLLM** | Yes | Requires `--tool-call-parser` flag matching the model family (e.g., `qwen3_xml` for Qwen models) |
| **Ollama** | Yes | Wraps llama.cpp with proper tool call handling |
| **llama.cpp (raw)** | Unreliable | Grammar-constrained decoding cannot reliably parse NemoClaw's structured tool call format — tool calls may be malformed or incomplete |
| **NVIDIA NIM** | Yes | Native support via `nvidia` provider type |
| **Cloud providers** | Yes | Anthropic, OpenAI, Google all support function calling natively |

**If tool calls are failing or producing malformed JSON**, check the inference backend first. The `--tool-call-parser` vLLM flag must match the model family — using the wrong parser silently corrupts tool call output.

### Chat Template and Think-Block Interaction (SKILL.md Rule #15)

The vLLM `--chat-template` flag critically affects tool calling for models that use thinking:

- **Qwen3.5 MoE** (e.g., 35B-A3B): Requires `<think>\n\n</think>\n\n` injection at generation start or produces empty output. But this think-block puts the model in "text response mode" and **prevents tool call generation**. Tool calling is inherently unreliable with Qwen3.5.
- **Qwen3 dense** (e.g., 32B): Does NOT require think-block. Use `/no_think` in the system prompt and no think injection. Tool calling works reliably.
- The `qwen3_xml` tool parser correctly handles text before `<tool_call>` tags — the parser is not the problem when tool calling fails with Qwen3.5.

When switching models, always verify: (1) chat template matches model's thinking requirements, (2) tool calling works end-to-end through OpenClaw, (3) session cache cleared after switch.

## Session Log: Tool Call Audit Trail

Every tool call is recorded in the session log files at `/sandbox/.openclaw-data/agents/main/sessions/*.jsonl`. Each entry contains:

```jsonl
{"role":"assistant","content":null,"tool_calls":[{"id":"call_abc","type":"function","function":{"name":"exec","arguments":"{\"command\":\"python3 /path/to/weather.py current vantaa\"}"}}]}
{"role":"tool","tool_call_id":"call_abc","content":"Current weather in Vantaa: 8 degrees, partly cloudy..."}
```

To search tool call history from the host:

```bash
# All tool calls in current sessions
ssh sandbox 'grep "tool_calls" /sandbox/.openclaw-data/agents/main/sessions/*.jsonl'

# Only exec tool calls
ssh sandbox 'grep "\"name\":\"exec\"" /sandbox/.openclaw-data/agents/main/sessions/*.jsonl'

# Tool call errors (denied by policy or execution failures)
ssh sandbox 'grep -A1 "tool_calls" /sandbox/.openclaw-data/agents/main/sessions/*.jsonl | grep "error\|denied\|blocked"'
```

For network-level audit (policy allow/deny decisions), see `references/access-control.md`.

## Debugging Tool Call Failures

When a tool call silently fails or produces no output, check these layers in order:

| Check | Command | What It Tells You |
|-------|---------|-------------------|
| 1. Was the tool called? | `ssh sandbox 'grep "tool_calls" /sandbox/.openclaw-data/agents/main/sessions/*.jsonl \| tail -5'` | Whether the LLM generated a tool_call at all |
| 2. Was it policy-blocked? | `openshell logs <sandbox> --level warn --since 5m` | Network denials (host:port + binary) |
| 3. Is the skill loaded? | `ssh sandbox 'openclaw skills list'` | Whether the skill shows as "ready" vs "blocked" or "missing" |
| 4. Is the helper script present? | `ssh sandbox 'ls -la /sandbox/.openclaw-data/skills/<name>/'` | Whether the script file exists post-restore |
| 5. Does the script work standalone? | `ssh sandbox 'python3 /path/to/helper.py <args>'` | Isolates script bugs from agent/policy issues |
| 6. Is inference working? | `nemoclaw <name> status` | Provider reachable, model loaded |

**Common failure patterns:**

- **Agent fabricates data instead of calling tool** — Model skips the skill and generates plausible-looking output. Fix: add `MANDATORY` + exact exec command to skill description. See SKILL.md rule #14.
- **Tool call produces malformed JSON** — Inference backend can't handle structured tool calling. Fix: check model compatibility table above. Use vLLM with correct `--tool-call-parser`.
- **Tool called but returns empty** — Helper script crashes silently. Fix: run script manually via SSH (check 5 above).
- **Tool called but network blocked** — Policy doesn't allow the endpoint. The agent sees an error like `Connection refused` or `Connection denied to host:port` as the tool result in the session log. Meanwhile, `openshell logs --level warn` shows the L7 denial with host, port, and binary. Fix: add endpoint via preset.
- **Skill shows "blocked"** — Required binary missing from sandbox image (by design — reduced attack surface). Fix: install the binary in Dockerfile or switch to a helper script that uses available binaries (`curl`, `python3`).

## Related References

- `references/skills.md` — Skill file format, discovery, loading, and native skill design
- `references/access-control.md` — Binary scoping as RBAC, audit trail, monitor-before-enforce
- `references/network-policies.md` — Full network policy schema and enforcement details
- `references/openclaw-config.md` — Model and skill configuration in openclaw.json
