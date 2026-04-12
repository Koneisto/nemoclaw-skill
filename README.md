# NemoClaw Skill for Claude Code

A comprehensive Claude Code skill for managing [NVIDIA NemoClaw](https://github.com/NVIDIA/NemoClaw) sandboxed AI agents from the host side.

## Why This Skill Exists

LLMs don't natively understand the OpenShell security layer that NemoClaw adds on top of OpenClaw. Without this context, AI assistants will confidently give advice that **breaks against the security boundary**:

| What the model suggests | Why it fails in NemoClaw |
|------------------------|-------------------------|
| `curl http://api.example.com` | OpenShell proxy only allows HTTPS on port 443 through policy-enforced endpoints |
| `export OPENAI_API_KEY=sk-...` inside sandbox | Credentials never enter the sandbox -- OpenShell injects them via the proxy |
| Edit `openclaw.json` and restart | The file is **read-only** (Landlock LSM protected) -- changes require a full sandbox rebuild |
| `openshell sandbox upload file.md` | Creates wrapper directories (`file.md/file.md`) -- must use SSH instead |
| Add a reverse proxy or MCP bridge | Non-standard wiring breaks on rebuild/reboot -- use direct policy entries |

These aren't edge cases. They're the **first things you hit** when working with NemoClaw, and every one produces confusing errors that look like bugs but are actually the security model working as designed.

## Trust the Architecture

The single most important lesson from production NemoClaw operation:

**Do NOT deviate from the standard NVIDIA/OpenShell/NemoClaw architecture.** Custom proxies, protocol bridges, blanket network policies -- every "clever" workaround adds a fragile link that breaks on rebuild, reboot, or policy update. When something doesn't fit neatly, step back and find the native approach.

The architecture that works:

```
sandbox --> direct endpoint (policy-controlled) --> service
```

The architecture that breaks:

```
sandbox --> bridge --> proxy --> service
```

If the sandbox needs to call an API, add a direct network policy entry. If the agent needs functionality, build a native skill with a helper script. One exec call, one response. No chains.

## Files

| File | Purpose |
|------|---------|
| `SKILL.md` | Entry point -- 24 critical rules, native skill design, checklists, common operations |
| `references/architecture.md` | OpenShell internals: gateway, sandbox, policy engine, credential injection |
| `references/cli-commands.md` | CLI reference for `nemoclaw`, `openshell`, and sandbox commands |
| `references/network-policies.md` | Network policy enforcement: direct endpoints, L7 rules, operator approval |
| `references/inference-profiles.md` | Inference routing via `inference.local`, provider switching |
| `references/workspace-backup.md` | Overlay filesystem, automated watchdog restore, backup procedures |
| `references/sessions.md` | Session lifecycle, skill snapshot caching, invalidation |
| `references/skills.md` | Bundled vs custom skills, native skill design, SOUL.md priority |
| `references/signal-bridge.md` | Signal bridge: setup, SSH forwarding, watchdog auto-restart |
| `references/presets.md` | Official and community network policy presets (YAML) |
| `references/troubleshooting.md` | Real error patterns from logs, post-reboot recovery |
| `references/dgx-spark.md` | DGX Spark: cgroup v2, local vLLM, ARM64 patches |
| `references/openclaw-config.md` | `openclaw.json` schema: model, gateway, channels, skills, plugins |
| `references/tool-calling.md` | Tool calling model: built-in tools, NemoClaw enforcement, model compatibility |
| `references/access-control.md` | Access control (binary scoping as RBAC), audit trail, monitor-before-enforce |
| `references/monitoring.md` | Health checks, log monitoring, watchdog as monitor, alerting patterns |
| `LICENSE` | Apache 2.0 |

## Sandbox Workspace Files

The OpenClaw agent inside the sandbox uses these workspace files at `/sandbox/.openclaw-data/workspace/`:

| File | Purpose | Priority |
|------|---------|----------|
| `SOUL.md` | Agent identity, personality, behavioral rules, allowed/forbidden actions | Highest -- overrides everything |
| `USER.md` | User context: name, preferences, language, interaction style | High |
| `IDENTITY.md` | Agent name, role description | High |
| `TOOLS.md` | Available services, API endpoints, entity IDs, credentials | Reference |
| `MEMORY.md` | Long-term memory promoted from daily notes (dreaming system) | Context |
| `AGENTS.md` | Multi-agent definitions (if used) | Optional |
| `HEARTBEAT.md` | Periodic status written by the agent | Auto-managed |
| `memory/*.md` | Daily notes (short-term memory, one file per day) | Auto-managed |

**Critical:** `SOUL.md` has the highest priority. If it references a specific tool or API, the agent will follow that over any skill instructions. All skill references in SOUL.md must point to the correct helper scripts.

These files live on the **overlay filesystem** and are wiped on every pod restart. The watchdog restores them from backup automatically.

## Key Knowledge

SKILL.md contains **21 critical operational rules** learned from production failures. The most important:

- **Rule #1**: Use preset system for network policies, not raw `openshell policy set`
- **Rule #3**: SSH for file transfer — `openshell upload/download` creates wrapper directories
- **Rule #9**: Workspace is ephemeral (overlay) — wiped on every pod restart, watchdog restore is mandatory
- **Rule #10**: Clear sessions AFTER workspace restore — agent caches stale system prompts
- **Rule #14**: Native skills (exec + curl/python3) preferred over MCP bridges

See SKILL.md for all 21 rules with full explanations.

## Known Upstream Issues

| Issue | Title | Skill Workaround |
|-------|-------|-----------------|
| [#486](https://github.com/NVIDIA/NemoClaw/issues/486) | Sandbox not available after restart | Automated watchdog restore pattern |
| [#888](https://github.com/NVIDIA/NemoClaw/issues/888) | SSH handshake secret regenerated | Hardcode secret in helm template |
| [#759](https://github.com/NVIDIA/NemoClaw/issues/759) | openclaw.json unwritable | Rebuild required for config changes |

## Installation

```bash
# Clone and symlink
git clone https://github.com/Koneisto/nemoclaw-skill.git
ln -s /path/to/nemoclaw-skill ~/.claude/skills/nemoclaw
```

The skill activates automatically when you ask Claude Code about NemoClaw management.

The `SKILL.md` and reference files are plain Markdown. You can include them as context in any LLM-based tool.

## Prerequisites

| Requirement | Minimum | Notes |
|-------------|---------|-------|
| OS | Ubuntu 22.04 LTS+ | Linux only — Landlock requires kernel 5.13+ |
| CPU | 4 vCPU | 4+ recommended |
| RAM | 8 GB | 16 GB recommended for local inference |
| Disk | 20 GB free | 40 GB recommended (model weights + backups) |
| Docker | 28+ | Must be running, cgroup v2 configured |
| Node.js | 22.16+ | Via nvm (NemoClaw installs this) |
| Network | HTTPS port 443 outbound | For cloud providers; local inference needs no outbound |

## Compatibility

- **NemoClaw**: v0.1.0+ (alpha)
- **OpenShell**: v0.0.23+ (tested), v0.0.16+ (minimum)
- **OpenClaw**: 2026.3.24+ (tested)
- **Platform**: Linux (Ubuntu 22.04+), Docker 28+

## Sources

- [NVIDIA NemoClaw](https://github.com/NVIDIA/NemoClaw) official repository
- [NVIDIA OpenShell](https://docs.nvidia.com/openshell/latest/) documentation
- [OpenClaw](https://docs.openclaw.ai/gateway/configuration-reference) configuration reference
- Production operational experience with local vLLM inference

## License

Apache 2.0

## Contributing

Issues and PRs welcome. Most valuable: real error patterns, operational gotchas, corrections as NemoClaw evolves.

---

*People buy Macs to run OpenClaw. It runs. It runs without guardrails. NemoClaw adds them -- kernel sandboxing, network isolation, credential separation. It's the right answer. It's also the hard answer. This skill exists because someone already bled through all the ways it breaks, and wrote it down.*
