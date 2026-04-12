---
name: nemoclaw
description: >
  Use when managing NVIDIA NemoClaw sandboxes, OpenShell security policies, inference routing,
  workspace backup/restore, or troubleshooting NemoClaw issues on the host side.
  Covers: sandbox lifecycle, network policy presets, operator approval via TUI, inference provider switching,
  blueprint management, DGX Spark deployment, Signal bridge, workspace files,
  Landlock/seccomp kernel isolation, sandbox hardening, native skill design,
  OpenShell CLI commands, openshell term, credential management, sandbox SSH access,
  OpenShell gateway management, provider credential injection, inference.local routing,
  policy schema (static vs dynamic), privacy router, L7 HTTP enforcement,
  openclaw.json configuration (model, gateway, channels, skills, contextWindow, maxTokens),
  tool calling model (JSON-schema tools, exec/read/write/browser), access control (binary scoping, RBAC),
  audit trail (openshell logs, policy decisions), model compatibility for tool calls.
  NOT for: plain OpenClaw without NemoClaw, Node-RED flows (use node-red skill),
  HA automations (use home-assistant-automation skill), n8n workflows (use n8n skills),
  ESPHome devices (use esphome-devices skill).
---

# NemoClaw Host Management

Skill for managing NVIDIA NemoClaw sandboxed AI agents from the host side.
NemoClaw = OpenClaw (agent) + OpenShell (security: Landlock, seccomp, network namespaces) + orchestration (CLI, presets, managed inference routing).

## Version Context

| Component | Version | Last Verified |
|-----------|---------|---------------|
| NemoClaw | 0.1.0 (alpha) | 2026-04-10 |
| OpenShell | 0.0.23 | 2026-04-10 |
| OpenClaw | 2026.3.24 | 2026-04-10 |
| This skill | Updated 2026-04-10 | Native skills, no MCP/Caddy, watchdog fixes, skill design rules |

Upstream: `github.com/NVIDIA/NemoClaw` — check `git log origin/main` for new commits before major operations.

## NemoClaw vs OpenClaw

| Aspect | OpenClaw | NemoClaw |
|--------|----------|----------|
| What | Local-first AI agent platform | OpenClaw + OpenShell security wrapper |
| Security | Application-layer | Kernel-level (Landlock, seccomp, namespaces) |
| Filesystem | Full access | `/sandbox` and `/tmp` only (read-write) |
| Network | Unrestricted | Deny-by-default, HTTPS port 443 only |
| Credentials | In-process | Host-side only, never in sandbox |
| Platform | macOS, Windows, Linux | Linux-only (Ubuntu 22.04+) |
| Install | `git clone` + `pnpm` | `curl` installer or `npm` |
| Tool calling | JSON-schema function calls, unrestricted execution | Same interface, OpenShell wraps execution with policy |
| Access control | Skill allowlist only (application-layer) | + Binary-scoped network + Landlock filesystem (OpenShell) |
| Audit/logging | Application-level only | + Every policy decision logged via OpenShell (allow/deny) |

They are complementary: OpenClaw is the base agent, NemoClaw is the secured wrapper.

## Architecture

```
Host CLI (nemoclaw) → Blueprint Runner → OpenShell CLI → Sandbox Container
                                                          ├── OpenClaw agent + native skills
                                                          ├── inference.local → OpenShell gateway → vLLM
                                                          ├── Deny-by-default network policy
                                                          ├── Direct API calls via native skills
                                                          └── Landlock filesystem isolation
```

Key boundaries:
- **Host**: `nemoclaw` CLI, `openshell` CLI, credentials, Signal bridge, watchdog
- **Sandbox**: OpenClaw agent, native skills (helper scripts), `/sandbox` workspace
- **Never crosses**: Raw API keys, unrestricted network, host filesystem access
- **Native skills preferred**: All skills use exec + curl/python3 directly. MCP is supported but not recommended (adds bridge layer). No Caddy proxy inside sandbox.

## Critical Rules

**Overarching principle: Do NOT deviate from standard NVIDIA/OpenShell/NemoClaw architecture.** Custom proxies (Caddy), protocol bridges (MCP-to-HTTP), blanket network policies — all caused persistent failures. If something requires non-standard wiring, step back and find the native approach. The architecture works when you work with it.

These are the non-obvious operational rules that cause failures if ignored:

### 1. Presets Are the Recommended Enforcement System

`openshell policy set <file>` applies policy metadata but lacks the merge logic and `--wait` flag that the preset pipeline uses. The result: the policy is recorded but traffic may remain blocked by the proxy until the connection state catches up. In practice, **always use the preset system** for reliable results.

- Presets live in `nemoclaw-blueprint/policies/presets/*.yaml`
- Apply via `nemoclaw <sandbox> policy-add` (interactive, recommended) or programmatically:
  ```bash
  cd ~/.nemoclaw/source && node -e "require('./bin/lib/policies.js').applyPreset('<sandbox>', '<preset-name>')"
  ```
- The `applyPreset()` function merges with current policy and uses `--wait` flag
- Note: the programmatic path depends on NemoClaw's internal file layout (`bin/lib/policies.js`). If it breaks after a NemoClaw update, use the interactive `policy-add` command instead
- See `references/network-policies.md` for full details

### 2. Port 443 HTTPS & Internal Service Access

The OpenShell proxy tunnels HTTPS on port 443. External APIs need `port: 443` + `tls: terminate` policy entries.

- Proxy at `10.200.0.1:3128` returns 403 for anything not in active policy
- External HTTPS: port 443 with `tls: terminate` (e.g., api.open-meteo.com, api.weatherapi.com)
- Internal HTTP: non-443 port with `access: full` (e.g., internal services on custom ports)
- Non-443 ports bypass TLS relay — the proxy controls host:port access only

**CRITICAL: No Caddy proxy in sandbox.** All services accessed directly with explicit policy entries. No MCP bridge, no reverse proxy layer. This is the standard OpenShell architecture.

### 3. SSH for File Transfer, Not `openshell upload/download`

Both `openshell sandbox upload` AND `openshell sandbox download` create wrapper directories (file.md becomes file.md/file.md directory). Always use SSH:

```bash
# Upload file to sandbox (always use .openclaw-data/ — that's the read-write overlay)
ssh sandbox "cat > /sandbox/.openclaw-data/workspace/FILE.md" < local/FILE.md

# Download file from sandbox
ssh sandbox "cat /sandbox/.openclaw-data/workspace/FILE.md" > local/FILE.md

# For directories, use tar over SSH:
ssh sandbox "tar cf - -C /sandbox/.openclaw-data/workspace ." | tar xf - -C local/workspace/
```

### 4. SSH Handshake Secret Bug (NemoClaw#888)

OpenShell regenerates `SSH_HANDSHAKE_SECRET` on every container restart, breaking SSH to existing sandboxes.

**Fix**: Hardcode the sandbox's secret in `/opt/openshell/manifests/openshell-helmchart.yaml` inside the `openshell-cluster-nemoclaw` container. This makes the entrypoint's `sed` a no-op.

**NEVER** use `openshell gateway start --recreate` without being prepared to recreate all sandboxes.

### 5. Sandbox Filesystem Restrictions

| Path | Access |
|------|--------|
| `/sandbox`, `/tmp`, `/dev/null` | Read-write |
| `/usr`, `/lib`, `/proc`, `/dev/urandom`, `/app`, `/etc`, `/var/log` | Read-only |
| `openclaw.json` | Read-only (Landlock) — changes require sandbox rebuild. See `references/openclaw-config.md` |

### 6. .bashrc/.profile Ownership

During Docker build, `.bashrc`/`.profile` are created as root-owned. The sandbox user can't write to them, causing `nemoclaw-start` failures. Fix: delete root-owned files and recreate as sandbox user.

### 7. Sandbox Hardening (Runtime)

The sandbox image has hardening that the Dockerfile alone cannot enforce. These are applied **automatically** by OpenShell at runtime — no manual configuration needed:

- **Purged tools**: `gcc`, `g++`, `make`, `netcat` removed from runtime image (automatic, in Dockerfile)
- **Process limit**: `ulimit -u 512` prevents fork bombs (automatic, via ENTRYPOINT)
- **Dropped capabilities**: `--cap-drop=ALL` at runtime (automatic, Docker runtime flag)
- **No privilege escalation**: `security_opt: no-new-privileges:true` (automatic, Docker runtime flag)

The following is **optional** and requires manual Docker Compose configuration:
- **Read-only rootfs**: `read_only: true` with tmpfs for `/tmp` — add to your Docker Compose override if desired

### 8. Providers Cannot Be Added to Running Sandboxes

OpenShell providers (credential entities) are bound at sandbox creation time. You cannot attach a new provider to a running sandbox — it requires delete and recreate.

### 9. Workspace Is Ephemeral (Overlay, NOT PVC)

Workspace files at `/sandbox/.openclaw-data/` live on the pod's **overlay filesystem**, NOT a Persistent Volume. They are **wiped on every pod restart** (reboot, crash, `openshell gateway start`). This is NVIDIA/NemoClaw#486 — no upstream fix as of April 2026.

- **Wiped on restart**: Every pod restart resets the overlay to image defaults — all workspace files, skills, cron jobs, sessions, and memory notes are lost
- **Wiped on `destroy`**: `nemoclaw <name> destroy` also deletes everything
- **Requires automated restore**: Use a watchdog with nightly backups to auto-restore after restart. See `references/workspace-backup.md` for the sentinel-based auto-restore pattern.
- **Back up frequently**: Nightly automated backup + manual backup before any destructive operation

### 10. Session Reset After Workspace Restore

When workspace files (SOUL.md, USER.md, TOOLS.md) are restored, existing sessions still cache the OLD system prompt (`systemSent: true` in `sessions.json`). The agent will ignore restored workspace files until sessions are cleared.

```bash
# List session files
ssh sandbox "ls /sandbox/.openclaw-data/agents/main/sessions/"

# Delete all session files (forces agent to re-read workspace on next message)
ssh sandbox "rm /sandbox/.openclaw-data/agents/main/sessions/sessions.json /sandbox/.openclaw-data/agents/main/sessions/*.jsonl"
```

Always clear sessions AFTER restoring ALL files (workspace AND skills), BEFORE starting services or sending messages. In a watchdog, clear sessions **twice**: once during initial restore, and again AFTER cron re-registration — because the gateway creates a stale skill snapshot between restores.

### 11. Delete BOOTSTRAP.md After Initial Setup AND After Every Upgrade

OpenClaw ships `BOOTSTRAP.md` which instructs the agent to introduce itself ("Hey. I just came online. Who am I?"). It gets **recreated on every OpenClaw upgrade** (new version re-creates default workspace files).

After initial setup AND after every upgrade:

1. Delete from sandbox: `ssh sandbox "rm /sandbox/.openclaw-data/workspace/BOOTSTRAP.md"`
2. Delete from ALL backups — if BOOTSTRAP.md is in backups, every workspace restore + session reset causes the agent to re-introduce itself
3. Reset sessions after deleting — otherwise the cached session may still have the bootstrap prompt

### 12. Cron PATH for nvm-Managed Binaries

`openshell` and `node` are installed via nvm/npm and are NOT in cron's minimal PATH. All scripts run from cron must use absolute paths:

```bash
OPENSHELL="${HOME}/.local/bin/openshell"
NODE="${HOME}/.nvm/versions/node/v$(node -v | tr -d v)/bin/node"
```

Without this, nightly backups silently fail and watchdog scripts cannot start the Signal bridge.

### 13. Gateway Check Must Not Rely on Process Name

Inside the sandbox, `ss -tlnp` does NOT show process names (sandbox user lacks privileges). Gateway health checks must check for LISTEN state only, not grep for "openclaw":

```bash
# WRONG (fails in sandbox — no process name visible):
ss -tlnp sport = :18789 | grep -q openclaw

# CORRECT:
ss -tln sport = :18789 | grep -q LISTEN
```

OpenClaw 2026.4.5+ auto-starts the gateway, so the watchdog's `start_gateway` will fail with "already running". The check-before-start pattern prevents this.

### 14. Native Skill Design Rules

Skills must be native (exec + helper script), NOT MCP-bridged. Key rules:

1. **One exec call, no chains.** Wrap logic in a helper script. Multiple sequential API calls are 4x slower.
2. **SOUL.md overrides SKILL.md.** Agent follows SOUL.md before skill instructions. All skill references must be consistent in SOUL.md.
3. **Agent takes shortcuts.** Smaller models skip reading SKILL.md if they think they know how. SKILL.md description must say MANDATORY + include the exact exec command.
4. **Description is the trigger.** Put the exec command IN the description: `"MANDATORY: Run /path/to/script"`, not just `"Get weather data"`.
5. **Session snapshot caching.** Clear sessions AFTER skill deployment — gateway caches skill list on pod startup.
6. **TOOLS.md shapes agent behavior.** The agent copies examples from TOOLS.md. If TOOLS.md shows entity IDs, agent uses entity IDs. If it shows friendly names, agent uses friendly names. Always use the same format you want the agent to produce.
7. **Never put API tokens in workspace .md files.** Helper scripts handle authentication internally. Bare tokens in TOOLS.md or SKILL.md get copied into curl commands by the agent, causing auth failures when copied wrong.
8. **All workspace files must be consistent.** SOUL.md, TOOLS.md, and SKILL.md must all point to the same scripts and use the same terminology. If SOUL.md says "use ha-control.sh with friendly names" but TOOLS.md shows entity IDs, the agent picks the familiar path and fails.
9. **Prefer native skills over MCP.** OpenClaw supports MCP natively, but the recommended NemoClaw pattern is exec + curl/python3 directly. MCP adds a bridge layer that is an extra failure point inside the locked-down sandbox.

### 15. Chat Template Must Match Model for Tool Calling

The chat template (`--chat-template` flag in vLLM) critically affects whether tool calling works:

- **Qwen3.5 MoE** (e.g., 35B-A3B): Requires `<think>\n\n</think>\n\n` injection at generation start or produces empty output. But this think-block puts the model in "text response mode" and **prevents tool call generation**. Tool calling is unreliable with Qwen3.5 — this is an inherent model limitation.
- **Qwen3 dense** (e.g., 32B): Does NOT require think-block. Use `/no_think` in the system prompt and NO think injection in the generation prompt. Tool calling works reliably.
- The `qwen3_xml` tool parser correctly handles text before `<tool_call>` tags — the parser is not the problem.

When switching models, always verify:
1. Chat template matches model's thinking requirements
2. Tool calling works end-to-end through OpenClaw (not just direct vLLM test)
3. Session cache cleared after model switch

### 16. Cron Restore Requires CLI Re-Registration

File copy to `/sandbox/.openclaw-data/cron/jobs.json` is NOT sufficient. The gateway holds cron state in memory and **overwrites the file** with its in-memory state. After restore:

1. Copy the file (for completeness)
2. Re-register each job via `openclaw cron add` with ALL fields including `--announce --channel last`
3. Use a Python parser to extract all fields from the backup JSON — shell parsing loses delivery config

### 17. Signal Bridge Detection Must Filter by Process Name

`pgrep -f "signal-bridge.js"` matches ANY process whose command line contains that string — including Claude Code bash sessions running diagnostics. This causes false "duplicate bridge" detection → kills the working bridge.

Fix: check `/proc/PID/comm` to verify the process is actually node:
```bash
for pid in $(pgrep -f "signal-bridge.js" 2>/dev/null); do
    comm=$(cat "/proc/$pid/comm" 2>/dev/null || true)
    if [[ "$comm" == node* ]]; then
        echo "$pid"  # Real bridge
    fi
done
```

### 18. TTS-Safe Output — No Unit Symbols

SOUL.md must enforce spelled-out units in all responses. Symbols like `°C`, `%`, `m/s`, `hPa` are spoken literally by TTS ("C", "percent sign", "m slash s") which sounds wrong.

SOUL.md must contain:
```
## Units
Always spell out units. No symbols. Critical for TTS.
- "5 degrees" not "5°C"
- "4 meters per second" not "4 m/s"
- "50 percent" not "50%"
- "1022 hectopascals" not "1022 hPa"
```

Helper scripts should also output spelled-out units. The agent sometimes ignores SOUL.md and copies raw API data (which contains symbols). Both layers matter.

### 19. No Duplicate Responses

The agent (especially Qwen3.5-35B) may output the raw tool result AND then rephrase it — producing two messages for one question. SOUL.md must contain:

```
NEVER send two messages for one question. When a tool returns data, present it ONCE in your own words.
```

### 20. OpenClaw Version Is Pinned in Dockerfile.base

OpenClaw is installed via `npm install -g openclaw@<version>` in `Dockerfile.base`, NOT the main Dockerfile. To upgrade OpenClaw:

1. Edit `~/.nemoclaw/source/Dockerfile.base` — change the version in `npm install -g openclaw@X.Y.Z`
2. Build the base image: `docker build -f Dockerfile.base -t ghcr.io/nvidia/nemoclaw/sandbox-base:latest .`
3. Then run `nemoclaw onboard` — it uses the locally-built base image

Without step 2, `nemoclaw onboard` pulls the cached/remote base image which has the OLD OpenClaw version. Docker build cache pruning alone does NOT help — the version pin is in the base image.

### 21. MEMORY.md Is a Required Core Workspace File

The OpenClaw UI expects `MEMORY.md` alongside other core files (SOUL.md, USER.md, etc.). If missing, the UI shows a "MEMORY missing" error on the Core Files panel. Create it during initial setup or restore it from backup.

### 22. Dreaming Requires Dockerfile Rebuild

The memory dreaming/REM system is configured via `plugins.entries.memory-core.config.dreaming` in `openclaw.json`. Since this file is Landlock read-only, enabling dreaming requires modifying the Dockerfile's `plugins` line and rebuilding the sandbox. See `references/openclaw-config.md#plugin-configuration-memory--dreaming` for the exact config.

When enabled, OpenClaw auto-creates a `Memory Dreaming Promotion` cron job that promotes short-term recall entries to `MEMORY.md` on the configured schedule.

### 23. `nemoclaw onboard` Binds Dashboard to 127.0.0.1

`nemoclaw onboard` starts the port forward with `127.0.0.1` (localhost only). The dashboard is unreachable from the LAN until the watchdog replaces it with `0.0.0.0`.

Add this check to your watchdog script to auto-fix the binding:
```bash
# Add to watchdog — detects 127.0.0.1 binding and restarts with 0.0.0.0
FORWARD_BIND=$(ss -tlnp sport = :${GATEWAY_PORT} | grep -o '[0-9.*]*:'"${GATEWAY_PORT}" | head -1)
if [[ "$FORWARD_BIND" == "127.0.0.1:${GATEWAY_PORT}" ]]; then
    openshell forward stop "${GATEWAY_PORT}" "$SANDBOX"
    openshell forward start --background "0.0.0.0:${GATEWAY_PORT}" "$SANDBOX"
fi
```

Without this watchdog check, manually fix after every onboard: `openshell forward start --background 0.0.0.0:18789 <sandbox>`

### 24. Optimized Build Context Only Copies Whitelisted Files

`sandbox-build-context.js` stages only specific files for Docker build: `Dockerfile`, `nemoclaw/`, `nemoclaw-blueprint/`, `scripts/nemoclaw-start.sh`. Extra files in the source root or scripts/ are NOT included.

To add files to the build context, either:
- Place them in a whitelisted directory (e.g., `scripts/`) AND add a copy line to `sandbox-build-context.js`
- Or use the legacy build context (`stageLegacySandboxBuildContext`) which copies everything

## Quick Reference

| Topic | Reference File |
|-------|---------------|
| Architecture & components | `references/architecture.md` |
| CLI commands (host + sandbox) | `references/cli-commands.md` |
| Network policies & enforcement | `references/network-policies.md` |
| Inference providers & switching | `references/inference-profiles.md` |
| Workspace files & backup/restore | `references/workspace-backup.md` |
| Sessions & session management | `references/sessions.md` |
| Skills system (bundled & custom) | `references/skills.md` |
| Signal bridge setup | `references/signal-bridge.md` |
| Policy presets (official + community) | `references/presets.md` |
| Troubleshooting guide | `references/troubleshooting.md` |
| DGX Spark deployment | `references/dgx-spark.md` |
| openclaw.json config reference | `references/openclaw-config.md` |
| Memory, dreaming & session cleanup | `references/openclaw-config.md` (Plugin Configuration section) |
| MCP servers (config, proxy, limitations) | `references/openclaw-config.md` (MCP section) |
| Tool calling model & OpenShell enforcement | `references/tool-calling.md` |
| Access control & audit trail | `references/access-control.md` |
| Monitoring & health checks | `references/monitoring.md` |
| Real-world deployment example (Koneisto) | `references/koneisto-deployment.md` |

## SSH Access to Sandbox

All SSH examples in this skill use `ssh sandbox` as shorthand. This requires OpenShell's SSH proxy.

### Inline usage (no config needed)

```bash
ssh -o ProxyCommand="openshell ssh-proxy --gateway-name <gateway> --name <sandbox>" sandbox "<command>"
```

The proxy flags are `--gateway-name` and `--name` (name mode). Do NOT use `--gateway`/`--sandbox-id` (token mode) — that requires a session token.

### Setting up a persistent SSH alias

```bash
# Option 1: Auto-generate SSH config (recommended)
openshell sandbox ssh-config <sandbox-name> >> ~/.ssh/config
# Creates host alias: openshell-<sandbox-name> (e.g., openshell-my-assistant)
# Usage: ssh openshell-my-assistant

# Option 2: Manual SSH config with custom alias
cat >> ~/.ssh/config << 'EOF'
Host sandbox
  User sandbox
  StrictHostKeyChecking no
  UserKnownHostsFile /dev/null
  LogLevel ERROR
  ProxyCommand openshell ssh-proxy --gateway-name <gateway-name> --name <sandbox-name>
EOF
# Usage: ssh sandbox
```

After setup, use the configured host alias in all SSH commands throughout this skill.

### Interactive shell (alternative to SSH)
```bash
nemoclaw <name> connect
```

### Host-side container access
To enter the OpenShell cluster container (for fixes like SSH handshake secret):
```bash
docker exec -it openshell-cluster-nemoclaw bash
```

## Common Operations

### Check sandbox status
```bash
nemoclaw <name> status
nemoclaw <name> status --json   # machine-readable
```

### Connect to sandbox
```bash
nemoclaw <name> connect
```

### Switch inference model at runtime
```bash
openshell inference set --provider <provider-name> --model <model>
# Does NOT restart sandbox or rewrite credentials
```

### Add a policy preset
```bash
nemoclaw <name> policy-add      # interactive
nemoclaw <name> policy-list     # see available/applied presets
```

### Monitor and approve network requests
```bash
openshell term                  # TUI: see blocked requests, approve/deny
```

### Backup/restore workspace
See `references/workspace-backup.md` for full procedures. Quick SSH backup:
```bash
BACKUP_DIR=~/.nemoclaw/backups/$(date +%Y%m%d-%H%M%S) && mkdir -p "$BACKUP_DIR"
for f in SOUL.md USER.md IDENTITY.md MEMORY.md TOOLS.md AGENTS.md; do
  ssh sandbox "cat /sandbox/.openclaw-data/workspace/$f" > "$BACKUP_DIR/$f" 2>/dev/null
done
```

### View/stream logs
```bash
nemoclaw <name> logs            # one-shot
nemoclaw <name> logs --follow   # stream
```

### Recreate sandbox (full rebuild)
```bash
cd ~/.nemoclaw/source && \
  NEMOCLAW_EXPERIMENTAL=1 \
  NEMOCLAW_RECREATE_SANDBOX=1 \
  nemoclaw onboard
```

## Pre-Completion Checklist

Before declaring any NemoClaw operation complete, verify:

### Network Policy Changes
- [ ] External endpoints use port 443 with `tls: terminate`
- [ ] Internal services: non-443 port with `access: full`
- [ ] No Caddy/MCP/blanket entries — direct endpoints only
- [ ] Tested connectivity from inside sandbox

### Inference Changes
- [ ] Verified with `nemoclaw <name> status` after switching
- [ ] Confirmed agent can reach new model via `inference.local`
- [ ] No credential changes needed (host-side credentials persist)

### Workspace/Backup Operations
- [ ] Used SSH `cat >` method (not `openshell sandbox upload`)
- [ ] Verified file ownership (sandbox user, not root)
- [ ] Confirmed files readable inside sandbox

### After Rebuild/Restart
- [ ] Dashboard forwarded on **0.0.0.0** (watchdog auto-fixes within 1 min)
- [ ] DNS proxy running: `bash scripts/setup-dns-proxy.sh nemoclaw <sandbox>`
- [ ] SSH handshake secret hardcoded (if container was recreated)
- [ ] Workspace restored (all core .md files + SOUL.md references correct skill scripts)
- [ ] Custom skills restored via SSH tar
- [ ] BOOTSTRAP.md deleted from workspace
- [ ] Sessions cleared AFTER skill restore (critical for skill snapshot)
- [ ] Network policy applied (direct endpoints, no Caddy/MCP)
- [ ] Cron jobs re-registered via CLI with `--announce --channel last` (file copy alone doesn't work)
- [ ] Sessions cleared AGAIN after cron registration (second clear for fresh skill snapshot)
- [ ] Signal bridge started and verified (check `/proc/PID/comm == node*`, not just pgrep)
- [ ] Chat template matches model (Qwen3.5 needs think-block; Qwen3 dense needs /no_think)
- [ ] No Bearer tokens in TOOLS.md or SKILL.md files (use helper scripts)
- [ ] TOOLS.md examples use friendly names, not entity IDs
- [ ] Custom skill E2E test: verify agent uses exec with helper script, not shortcuts

## Updating NemoClaw

### Pull New Version
```bash
cd ~/.nemoclaw/source
git stash           # preserve custom patches
git pull origin main
git stash pop       # re-apply patches (resolve conflicts if any)
```

### Re-apply Custom Patches
After `git pull`, custom patches (Dockerfile, onboard.js, local-inference.js, etc.) may be overwritten. Check for conflicts and re-apply. See `references/dgx-spark.md` for the patch table.

### Rebuild Sandbox
```bash
nemoclaw onboard    # rebuilds with updated code
```

After rebuild, follow the full restore procedure in `references/workspace-backup.md`.

## Credential Management

Credentials are stored at `~/.nemoclaw/credentials.json` on the host. The sandbox never sees raw API keys.

```bash
# View current credentials (sensitive!)
cat ~/.nemoclaw/credentials.json

# After rotating an API key:
# 1. Update credentials.json with new key
# 2. Re-run onboard to propagate
nemoclaw onboard
```

### Credential Rotation Without Losing State

Re-running `nemoclaw onboard` rebuilds the sandbox, which **wipes the overlay filesystem**. To rotate credentials without losing workspace state:

1. **Backup first**: Run a full workspace backup (see `references/workspace-backup.md`)
2. **Update credentials**: Edit `~/.nemoclaw/credentials.json` with the new key
3. **Re-onboard**: `nemoclaw onboard` (rebuilds sandbox with new credentials)
4. **Restore workspace**: Follow the full restore procedure (workspace files, skills, cron jobs, sessions)
5. **Verify**: `nemoclaw <name> status` to confirm new credentials work

The gateway token also changes on rebuild — update any external systems that depend on it.

## Environment Variables

| Variable | Purpose | Default |
|----------|---------|---------|
| `NEMOCLAW_EXPERIMENTAL` | Enable vLLM/NIM providers | unset |
| `NEMOCLAW_GW_PORT` | Gateway port | 18789 |
| `VLLM_PORT` | vLLM port for local inference | 8000 |
| `NEMOCLAW_SANDBOX_NAME` | Non-interactive sandbox name | — |
| `NEMOCLAW_RECREATE_SANDBOX` | Force sandbox recreation | — |
| `NEMOCLAW_PROVIDER` | Non-interactive provider selection | — |
| `NEMOCLAW_MODEL` | Non-interactive model selection | — |
| `CHAT_UI_EXTRA_ORIGINS` | Additional allowed dashboard origins | — |
| `TELEGRAM_BOT_TOKEN` | Telegram bridge token | — |
| `ALLOWED_CHAT_IDS` | Restrict Telegram access | unset (all) |
| `SIGNAL_PHONE_NUMBER` | Signal bridge sender number | — |
| `ALLOWED_SIGNAL_NUMBERS` | Restrict Signal access | — |
| `NEMOCLAW_GPU` | Remote GPU type for Brev deploy | `a2-highgpu-1g:nvidia-tesla-a100:1` |

## Known Compatibility Issues

| Issue | Impact | Workaround |
|-------|--------|-----------|
| llama.cpp grammar constraints vs tool call format | Tool calls fail or produce malformed JSON | Use Ollama or vLLM with `--tool-call-parser` flag matching model family |
| Kernel 5.13+ required for Landlock | Landlock silently degrades on older kernels | Verify: `cat /sys/kernel/security/lsm` should include `landlock` |
| Overlay filesystem wipe on restart (#486) | All workspace, skills, sessions lost | Automated watchdog restore (see `references/workspace-backup.md`) |

## Key Constants

- Default gateway port: **18789**
- Credentials file: `~/.nemoclaw/credentials.json`
- Baseline policy: `nemoclaw-blueprint/policies/openclaw-sandbox.yaml`
- Sandbox image: `ghcr.io/nvidia/openshell-community/sandboxes/openclaw`
- Inference endpoint in sandbox: `inference.local`
- Sandbox name rules: RFC 1123 (lowercase alphanumeric + hyphens)
- NemoClaw source: `~/.nemoclaw/source`
- Blueprint policies: `~/.nemoclaw/source/nemoclaw-blueprint/policies/`
- Backups: `~/.nemoclaw/backups/`

## Multi-Sandbox Considerations

Most examples in this skill assume a single sandbox ("my-assistant"). If adding a second sandbox:

### Per-Sandbox Resources (must be separate)
- **Gateway name**: each sandbox needs its own (e.g., `nemoclaw-dev`, `nemoclaw-prod`)
- **Network policy**: applied per-sandbox via `openshell policy set --gateway <gw> <sandbox>`
- **Workspace backups**: separate `latest-good` symlink per sandbox (e.g., `latest-good-dev`, `latest-good-prod`)
- **Watchdog config**: per-sandbox variables — don't hardcode sandbox name. Use a config file or loop over sandboxes.
- **PID files**: separate directories (`/tmp/nemoclaw-services-<sandbox>/`)
- **Signal bridge**: one phone number per sandbox (each bridge binds to one WebSocket)
- **Dashboard port**: each sandbox needs its own forwarded port (e.g., 18789, 18790)
- **Sessions**: completely independent per sandbox

### Shared Resources (single instance, all sandboxes use)
- **vLLM**: one model server, all sandboxes route through `inference.local` → same provider
- **signal-cli-rest-api**: one Docker container, one phone number — can only serve one bridge
- **OpenShell cluster container**: one `openshell-cluster-*` container per gateway
- **NemoClaw source**: one checkout in `~/.nemoclaw/source` — Dockerfile patches affect all sandboxes built from it. To differentiate sandbox configs (e.g., different models), use per-sandbox env vars during `nemoclaw onboard`, not Dockerfile branches

### Watchdog Template

To support multiple sandboxes, parameterize the watchdog instead of hardcoding:

```bash
for SANDBOX in my-assistant dev-sandbox; do
    GATEWAY="nemoclaw"  # or per-sandbox gateway
    RESTORE_MARKER="/tmp/nemoclaw-workspace-restored-${SANDBOX}"
    SIGNAL_BRIDGE_PIDDIR="/tmp/nemoclaw-services-${SANDBOX}"
    # ... rest of watchdog logic per sandbox
done
```

## Related Skills

- **home-assistant-automation** — HA YAML automations and config
- **node-red** — Node-RED flow development
- **esphome-devices** — ESP32/ESP8266 device firmware

---

For detailed documentation on any topic, read the appropriate reference file from the Quick Reference table.
