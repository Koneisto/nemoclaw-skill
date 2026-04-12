# NemoClaw Architecture

## Overview

NemoClaw is an orchestration layer that deploys OpenClaw inside NVIDIA OpenShell sandboxes. OpenShell provides kernel-level isolation (Landlock LSM, seccomp, network namespaces); NemoClaw provides the CLI, presets, and blueprint system to manage the deployment.

## Component Glossary

Four components form the stack. Documentation throughout this skill attributes features to the correct component:

| Component | What It Is | What It Provides | Runs Where |
|-----------|-----------|-----------------|------------|
| **OpenClaw** | Base AI agent platform | Tool calling (exec/read/write/browser), skills (SKILL.md), session management, memory/dreaming, cron, gateway, dashboard, channels (Signal/Telegram/Discord), MCP support | Inside sandbox |
| **OpenShell** | Security runtime | Landlock filesystem isolation, seccomp syscall filtering, network namespaces, deny-by-default proxy, binary scoping, credential injection, policy engine, audit logging | Sandbox container + host gateway |
| **NemoClaw** | Orchestration CLI + blueprint | `nemoclaw` CLI commands, onboard wizard, policy preset system, `openshell term` TUI, sandbox lifecycle, inference routing setup, blueprint versioning | Host |
| **nemoclaw-blueprint** | Versioned policy artifact | Baseline policy YAML (`openclaw-sandbox.yaml`), policy presets (`presets/*.yaml`), sandbox image config | Consumed by NemoClaw at deploy time |

**Rule of thumb:** If it's about *what the agent can do*, that's OpenClaw. If it's about *what the agent is allowed to do*, that's OpenShell. If it's about *deploying and managing* the agent, that's NemoClaw.

## Component Stack

```
┌─────────────────────────────────────────────────────────┐
│ Host                                                     │
│  nemoclaw CLI ──► Blueprint Runner ──► openshell CLI     │
│  Credentials (~/.nemoclaw/credentials.json)              │
│  Gateway process (cluster port configurable, default 18789)│
│  Dashboard forward (SSH tunnel to sandbox gateway port)   │
├─────────────────────────────────────────────────────────┤
│ OpenShell Sandbox (Docker + k3s)                         │
│  ┌─────────────────────────────────────────────────┐     │
│  │ OpenClaw agent + NemoClaw plugin                 │     │
│  │ inference.local ──► OpenShell gateway ──► Provider│    │
│  │ Landlock filesystem isolation                    │     │
│  │ seccomp syscall filtering                        │     │
│  │ Network namespace (deny-by-default)              │     │
│  │ /sandbox (rw), /tmp (rw), /usr (ro), /etc (ro)  │     │
│  └─────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────┘
```

## OpenShell Layer (the Security Runtime)

OpenShell is the runtime underneath NemoClaw. Understanding its internals is essential for troubleshooting.

### Four Core Components

| Component | Role | Details |
|-----------|------|---------|
| **Gateway** | Control plane | Manages sandbox lifecycle, credential storage, policy delivery, inference config. Runs in Docker with mTLS. |
| **Sandbox** | Data plane | Isolated runtime with container supervision and policy-enforced egress. Each sandbox = k3s pod. |
| **Policy Engine** | Defense-in-depth | Enforces filesystem, network, and process constraints at infrastructure layer. NemoClaw provides presets and TUI to manage policies. |
| **Privacy Router** | LLM routing | Keeps sensitive context on sandbox compute. Routes based on cost and privacy policy. Strips sandbox creds, injects provider creds. |

### Request Processing Flow

When the sandbox makes an outbound connection, OpenShell's proxy:

1. **Identifies** the initiating binary (e.g., `/usr/local/bin/openclaw`, `/usr/bin/curl`)
2. **Routes** `https://inference.local` requests as managed inference (credential injection)
3. **Queries** policy engine for all other destinations
4. **Returns** Allow (matching policy) or Deny (logged, blocked)
5. **Decrypts** TLS and validates against per-method, per-path rules (L7 enforcement)

### Tool Call Enforcement

When the agent makes a tool call, enforcement depends on the tool type:

| Tool | Enforcement Point | Policy Layer |
|------|-------------------|-------------|
| `exec` (shell command) | Binary identification + network policy | Network namespace |
| `read` / `write` | Filesystem path check | Landlock LSM |
| `browser` / `search` | Proxy evaluates destination host:port | Network namespace + L7 |
| `inference.local` | Managed routing, credential injection | Gateway |

The proxy identifies which binary initiated the connection (step 1 in Request Processing Flow above). This binary identity is matched against the `binaries` field in network policy entries. A `curl` call from a helper script is identified as `/usr/bin/curl`, not as the agent process — this is how binary scoping provides tool-level access control.

See `references/tool-calling.md` for the full tool calling model and `references/access-control.md` for the complete access control framework.

### Gateway Deployment Modes

| Mode | Command | Use Case |
|------|---------|----------|
| Local | `openshell gateway start` | Development, iteration |
| Remote | `openshell gateway start --remote user@host` | Powerful machines via SSH |
| Cloud | Behind reverse proxy | Production, external access |

All three expose identical API surfaces. Running `openshell sandbox create` without a gateway auto-bootstraps a local one.

### Credential Injection (Provider System)

OpenShell treats credentials as managed **providers**. The proxy replaces agent-visible opaque tokens with real credentials before upstream transmission:

- Sandbox environment gets placeholder tokens (never raw keys)
- Proxy resolves placeholders to real values at request time
- Supported injection: HTTP headers, basic auth, query params, URL segments
- Requests with unresolvable placeholders get HTTP 500 (never forwarded)

**Critical**: Providers cannot be added to a running sandbox — requires delete and recreate.

### Supported Provider Types

| Type | Auto-discovered Variables | Purpose |
|------|--------------------------|---------|
| `claude` | `ANTHROPIC_API_KEY`, `CLAUDE_API_KEY` | Claude Code, Anthropic API |
| `github` | `GITHUB_TOKEN`, `GH_TOKEN` | GitHub operations |
| `openai` | `OPENAI_API_KEY` | OpenAI-compatible endpoints |
| `nvidia` | `NVIDIA_API_KEY` | NVIDIA API Catalog |
| `generic` | User-defined | Custom services |

### Platform Support

| Platform | Architecture | Status |
|----------|-------------|--------|
| Linux (Debian/Ubuntu) | x86_64, aarch64 | Fully supported |
| macOS + Docker Desktop | arm64 (Apple Silicon) | Supported |
| Windows + WSL 2 | x86_64 | Experimental |

Docker 28.04+ required. Landlock and seccomp operate within Docker's Linux VM on macOS.

## Two Main NemoClaw Components

### 1. TypeScript Plugin

A thin package that integrates with OpenClaw CLI. Runs in-process with the OpenClaw gateway inside the sandbox.

```
nemoclaw/
├── src/
│   ├── index.ts                    Plugin entry — registers all commands
│   ├── cli.ts                      Commander.js subcommand wiring
│   ├── commands/
│   │   ├── launch.ts               Fresh install into OpenShell
│   │   ├── connect.ts              Interactive shell into sandbox
│   │   ├── status.ts               Blueprint run state + sandbox health
│   │   ├── logs.ts                 Stream blueprint and sandbox logs
│   │   └── slash.ts                /nemoclaw chat command handler
│   └── blueprint/
│       ├── resolve.ts              Version resolution, cache management
│       ├── fetch.ts                Download blueprint from OCI registry
│       ├── verify.ts               Digest verification, compatibility checks
│       ├── exec.ts                 Subprocess execution of blueprint runner
│       └── state.ts                Persistent state (run IDs)
├── openclaw.plugin.json            Plugin manifest
└── package.json
```

### 2. Python Blueprint

A versioned artifact with its own release stream. The plugin resolves, verifies, and executes it as a subprocess. Drives all interactions with the OpenShell CLI.

```
nemoclaw-blueprint/
├── blueprint.yaml                  Manifest — version, profiles, compatibility
├── policies/
│   ├── openclaw-sandbox.yaml       Default network + filesystem policy
│   └── presets/                    Named preset policies
│       ├── discord.yaml
│       ├── docker.yaml
│       ├── huggingface.yaml
│       ├── jira.yaml
│       ├── npm.yaml
│       ├── outlook.yaml
│       ├── pypi.yaml
│       ├── slack.yaml
│       └── telegram.yaml
```

## Blueprint Lifecycle

```
Resolve → Verify digest → Plan → Apply → Status
```

1. **Resolve**: Locate artifact, check version against `min_openshell_version` and `min_openclaw_version`
2. **Verify**: Check artifact digest against expected value
3. **Plan**: Determine what OpenShell resources to create/update (gateway, providers, sandbox, inference route, policy)
4. **Apply**: Execute the plan via `openshell` CLI commands
5. **Status**: Report current state

## Inference Routing

Agent inference requests never leave the sandbox directly. OpenShell intercepts and routes them:

```
Agent (sandbox) ──► inference.local ──► OpenShell gateway ──► Provider (cloud/local)
```

- The sandbox knows which model family to use
- OpenShell owns the actual provider credential and upstream endpoint
- Credentials are stored on host in `~/.nemoclaw/credentials.json`
- The sandbox never receives raw API keys

## Security Layers

| Layer | Mechanism | Purpose | Flexibility |
|-------|-----------|---------|-------------|
| Network | Network namespace | Deny-by-default egress, YAML-defined allow rules | **Hot-reloadable** |
| Inference | Transparent routing | All model calls go through gateway | **Hot-reloadable** |
| Filesystem | Landlock LSM | Confine to `/sandbox` + `/tmp` (rw), system paths (ro) | Locked at creation |
| Process | seccomp + ulimit | Filter syscalls, prevent fork bombs (`ulimit -u 512`) | Locked at creation |
| Capabilities | `--cap-drop=ALL` | Drop all Linux capabilities at runtime | Runtime flag |
| Credentials | Host-side storage | Sandbox never sees raw API keys | — |
| Policy | Operator approval | Unknown hosts blocked, surfaced in TUI | Session-scoped |

### Sandbox Hardening Details

- Build toolchains (`gcc`, `g++`, `make`) and network probes (`netcat`) **purged** from runtime image
- Process limit: `ulimit -u 512` via ENTRYPOINT prevents fork bombs
- `--cap-drop=ALL` enforced at runtime (Dockerfile cannot enforce this)
- Optional: read-only rootfs with tmpfs for `/tmp`
- `security_opt: no-new-privileges:true` prevents privilege escalation

## Messaging Bridges

NemoClaw supports host-side messaging bridges that forward messages to/from the sandboxed agent:

| Bridge | Transport | Status |
|--------|-----------|--------|
| Telegram | Bot API via `TELEGRAM_BOT_TOKEN` | Official, `nemoclaw start` |
| Signal | SSH-based (NOT HTTP gateway) | Custom script |
| Discord | Discord webhook API | Via preset |
| Slack | Slack API and webhooks | Via preset |

Bridges run on the host, outside the sandbox. They connect to the agent via SSH (Signal) or through the gateway API (Telegram).

## Design Principles

- **Thin plugin, versioned blueprint**: Plugin stays small and stable. Orchestration logic lives in the blueprint.
- **Respect CLI boundaries**: `nemoclaw` CLI is the primary interface for sandbox management.
- **Supply chain safety**: Blueprint artifacts are immutable, versioned, and digest-verified.
- **Deny by default**: No access is granted unless explicitly allowed in policy.

## Sandbox Environment

The sandbox runs the `ghcr.io/nvidia/openshell-community/sandboxes/openclaw` container image. Inside:

- OpenClaw runs with NemoClaw plugin pre-installed
- Inference calls routed through OpenShell to configured provider
- Network egress restricted by baseline policy in `openclaw-sandbox.yaml`
- Filesystem access confined by Landlock
- Sandbox process runs as dedicated `sandbox` user and group

### Overlay Filesystem

The `.openclaw` directory uses an overlay pattern:

| Path | Access | Purpose |
|------|--------|---------|
| `/sandbox/.openclaw/` | Read-only (Landlock) | Base layer — `openclaw.json`, default agents, bundled skills |
| `/sandbox/.openclaw-data/` | Read-write | Overlay — runtime writes (workspace, user skills, cron, memory) |

Both have identical directory structure. Writes go to `.openclaw-data/`, reads fall through to `.openclaw/`. For backup/restore, always target `.openclaw-data/`.

**CRITICAL: The overlay is ephemeral.** Every pod restart (reboot, crash, `openshell gateway start`) wipes `/sandbox/.openclaw-data/` back to image defaults. All workspace files, custom skills, cron jobs, sessions, and memory notes are lost. This is NVIDIA/NemoClaw#486 — no upstream fix as of April 2026. Automated watchdog restore is mandatory for always-on deployments. See `references/workspace-backup.md`.

## Requirements

| Resource | Minimum | Recommended |
|----------|---------|-------------|
| CPU | 4 vCPU | 4+ vCPU |
| RAM | 8 GB | 16 GB |
| Disk | 20 GB free | 40 GB free |
| OS | Ubuntu 22.04 LTS+ | — |
| Node.js | 22.16+ | — |
| npm | 10+ | — |
| Docker | Running | cgroup v2 configured |
