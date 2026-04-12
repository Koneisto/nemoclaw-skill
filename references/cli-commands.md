# NemoClaw CLI Commands Reference

## Host Commands

### Lifecycle Management

| Command | Description |
|---------|-------------|
| `nemoclaw onboard` | Interactive setup wizard — creates gateway, registers providers, builds sandbox image, creates sandbox |
| `nemoclaw list` | List all sandboxes with model, provider, and policy presets |
| `nemoclaw <name> connect` | Connect to sandbox (interactive shell) |
| `nemoclaw <name> status` | Show sandbox status, health, and inference configuration |
| `nemoclaw <name> status --json` | Machine-readable status output |
| `nemoclaw <name> logs [--follow]` | View/stream sandbox logs |
| `nemoclaw <name> destroy` | Delete sandbox permanently (**WARNING: deletes all workspace files**) |
| `nemoclaw setup-spark` | DGX Spark cgroup v2 fixes (run with `sudo`) |

### Policy Management

| Command | Description |
|---------|-------------|
| `nemoclaw <name> policy-add` | Add policy preset to sandbox (interactive) |
| `nemoclaw <name> policy-list` | List available and applied presets |

### Auxiliary Services

| Command | Description |
|---------|-------------|
| `nemoclaw start` | Start auxiliary services (Telegram/Signal bridge, cloudflared) |
| `nemoclaw stop` | Stop all auxiliary services |
| `nemoclaw status` | Show sandbox list + auxiliary service status |

### Deployment

| Command | Description |
|---------|-------------|
| `nemoclaw deploy <instance>` | Deploy to remote GPU via Brev (EXPERIMENTAL) |

## OpenShell Commands (Host)

### Sandbox Management

| Command | Description |
|---------|-------------|
| `openshell sandbox create -- claude` | Create sandbox (auto-bootstraps gateway if needed) |
| `openshell sandbox create --from openclaw` | Create sandbox from community image |
| `openshell sandbox create --policy ./policy.yaml -- claude` | Create with custom policy |
| `openshell sandbox list` | List underlying sandboxes |
| `openshell sandbox get <name>` | Inspect sandbox details |
| `openshell sandbox connect <name>` | Terminal access into sandbox |
| `openshell sandbox upload <name> <local> <remote>` | Upload file (**AVOID — creates wrapper dirs, use SSH**) |
| `openshell sandbox download <name> <remote> <local>` | Download file from sandbox |
| `openshell sandbox delete <name>` | Delete sandbox |
| `openshell sandbox ssh-config <name>` | Generate SSH config entry for sandbox |

### Gateway Management

| Command | Description |
|---------|-------------|
| `openshell gateway start` | Start local gateway (Docker, mTLS) |
| `openshell gateway start --remote user@host` | Start remote gateway via SSH |
| `openshell gateway start --recreate` | Reset gateway (**destroys all state**) |
| `openshell gateway stop` | Stop gateway (preserving state) |
| `openshell gateway destroy` | Destroy gateway permanently |
| `openshell gateway info` | Show gateway details |
| `openshell gateway select <name>` | Switch active gateway |
| `openshell status` | Health check for gateway + sandboxes |

### Provider Management

| Command | Description |
|---------|-------------|
| `openshell provider create --name <n> --type <t> --from-existing` | Create provider from env vars |
| `openshell provider create --name <n> --type generic --credential KEY=val` | Create with explicit value |
| `openshell provider list` | List all providers |
| `openshell provider get <name>` | Inspect provider |
| `openshell provider update <name> --type <t> --from-existing` | Update credentials |
| `openshell provider delete <name>` | Delete provider |

**Critical**: Providers cannot be added to a running sandbox. Requires delete + recreate.

### Inference Routing

| Command | Description |
|---------|-------------|
| `openshell inference set --provider <name> --model <model>` | Set active model (no restart needed) |
| `openshell inference get` | Show current inference config |
| `openshell inference update --timeout <ms>` | Update inference settings |

Changes propagate to all sandboxes on the gateway within ~5 seconds.

### Policy Management

| Command | Description |
|---------|-------------|
| `openshell policy set <sandbox> --policy <file> --wait` | Apply policy (wait for confirmation) |
| `openshell policy set --gateway <gw> <sandbox> --policy <file>` | Apply with explicit gateway |
| `openshell policy set --global --policy <file>` | Apply global policy to all sandboxes |
| `openshell policy get <sandbox> --full` | Export full merged policy |
| `openshell policy get --gateway <gw> <sandbox>` | Export policy with explicit gateway |
| `openshell policy list <sandbox>` | List applied policy names |
| `openshell policy delete --global` | Remove global policy |

### Monitoring & Logging

| Command | Description |
|---------|-------------|
| `openshell term` | TUI — network activity, approve/deny requests |
| `openshell logs <sandbox> --tail` | Stream logs |
| `openshell logs <sandbox> --source sandbox` | Filter by source |
| `openshell logs <sandbox> --level warn` | Filter by level |
| `openshell logs <sandbox> --since 5m` | Filter by time |
| `openshell doctor logs` | View gateway diagnostic logs |
| `openshell doctor logs --tail` | Stream gateway diagnostic logs |

### Port Forwarding

| Command | Description |
|---------|-------------|
| `openshell forward start <port> <sandbox>` | Start port forward |
| `openshell forward start --background <bind>:<port> <sandbox>` | Background port forward |
| `openshell forward list` | List active forwards |
| `openshell forward stop <port> <sandbox>` | Stop port forward |

## Inside Sandbox

| Command | Description |
|---------|-------------|
| `openclaw tui` | Interactive chat TUI |
| `openclaw agent --agent main --local -m "msg" --session-id <id>` | CLI single message |
| `openclaw cron add` | Add scheduled cron job |
| `openclaw cron list` | List cron jobs |
| `openclaw devices list --json` | List device pairings |
| `openclaw devices approve <requestId>` | Approve device pairing |
| `/nemoclaw status` | Slash command: show sandbox/inference state |
| `/compact` | Reclaim context window (80-90% recovery) |

## Non-Interactive Onboarding

For scripted/automated setup:

```bash
cd ~/.nemoclaw/source && \
  NEMOCLAW_EXPERIMENTAL=1 \
  NEMOCLAW_GW_PORT=8088 \
  VLLM_PORT=8001 \
  NEMOCLAW_SANDBOX_NAME=my-assistant \
  NEMOCLAW_RECREATE_SANDBOX=1 \
  NEMOCLAW_PROVIDER=vllm \
  NEMOCLAW_MODEL=model \
  CHAT_UI_EXTRA_ORIGINS="http://<HOST_LAN_IP>:18789,http://<HOSTNAME>.local:18789" \
  nemoclaw onboard --non-interactive
```

## Sandbox Name Rules

- RFC 1123 subdomain format
- Lowercase alphanumeric characters and hyphens only
- Must start and end with alphanumeric character
- Uppercase letters are automatically lowercased
- Examples: `my-assistant`, `dev1`, `home-agent`

## Important Notes

- `nemoclaw <name> destroy` **permanently deletes** all files inside the sandbox, including workspace files. Always back up first.
- `openshell sandbox upload` creates wrapper directories — use SSH `cat >` instead.
- Runtime model switching via `openshell inference set` does NOT rewrite credentials or restart the sandbox.
- `openshell policy set` applies only for the current session — changes reset when sandbox stops.
