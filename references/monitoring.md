# NemoClaw Monitoring & Health Checks

## Overview

NemoClaw 0.1.0 does not ship with a monitoring dashboard or structured metrics export. Health monitoring relies on CLI commands, log inspection, and the watchdog pattern. This reference documents what to check and how.

## Quick Health Check

Run from the host to verify all critical components:

```bash
#!/bin/bash
# nemoclaw-healthcheck.sh — run manually or from cron
SANDBOX="my-assistant"

echo "=== Sandbox Status ==="
nemoclaw "$SANDBOX" status

echo "=== Gateway Port ==="
ss -tln sport = :18789 | grep -q LISTEN && echo "OK: Gateway listening" || echo "FAIL: Gateway not listening"

echo "=== OpenShell Cluster ==="
docker ps --filter name=openshell-cluster --format '{{.Names}} {{.Status}}'

echo "=== Inference ==="
curl -sf http://localhost:8001/v1/models >/dev/null && echo "OK: vLLM responding" || echo "FAIL: vLLM not responding"

echo "=== Signal Bridge ==="
pgrep -f "signal-bridge.js" >/dev/null && echo "OK: Bridge running" || echo "WARN: Bridge not running"

echo "=== Recent Policy Denials ==="
openshell logs "$SANDBOX" --level warn --since 1h 2>/dev/null | tail -5
```

## What to Monitor

| Component | Check | Command | Healthy State |
|-----------|-------|---------|---------------|
| Sandbox | Running and ready | `nemoclaw <name> status` | Status: `Running` |
| Gateway port | Listening on host | `ss -tln sport = :18789 \| grep LISTEN` | LISTEN present |
| OpenShell cluster | Docker container running | `docker ps --filter name=openshell-cluster` | Up, not restarting |
| Inference (vLLM) | Model loaded and serving | `curl -sf http://localhost:8001/v1/models` | Returns model list |
| Signal bridge | Process alive | `pgrep -f signal-bridge.js` | PID found |
| Workspace sentinel | Not wiped by restart | `ssh sandbox 'test -f /sandbox/.openclaw-data/.nemoclaw-restored && echo OK'` | "OK" |
| Dashboard forward | Bound to 0.0.0.0 | `ss -tln sport = :18789 \| grep '0.0.0.0'` | 0.0.0.0 (not 127.0.0.1) |
| Disk space | Backup dir not full | `df -h ~/.nemoclaw/backups` | Sufficient free space |
| GPU memory | Not at capacity | `nvidia-smi --query-gpu=memory.used,memory.total --format=csv` | Used < 90% of total |

## Log Monitoring

### Policy Decisions (OpenShell)

```bash
# Stream all policy decisions (most verbose)
openshell logs <sandbox> --tail

# Only denials — the most actionable signal
openshell logs <sandbox> --level warn --tail

# Denials in the last hour (for periodic checks)
openshell logs <sandbox> --level warn --since 1h

# Count denials per hour (rough rate)
openshell logs <sandbox> --level warn --since 1h 2>/dev/null | wc -l
```

### Agent Activity (Session Logs)

```bash
# Count tool calls today
ssh sandbox 'grep -c "tool_calls" /sandbox/.openclaw-data/agents/main/sessions/*.jsonl 2>/dev/null'

# Last 5 tool calls
ssh sandbox 'grep "tool_calls" /sandbox/.openclaw-data/agents/main/sessions/*.jsonl 2>/dev/null | tail -5'

# Check for tool errors
ssh sandbox 'grep -A1 "tool_calls" /sandbox/.openclaw-data/agents/main/sessions/*.jsonl 2>/dev/null | grep -i "error\|denied\|failed"'
```

### Gateway Diagnostics

```bash
# Control plane logs
openshell doctor logs --tail

# Gateway health
openshell doctor status
```

## Watchdog as Monitor

In NemoClaw 0.1.0, the **watchdog script is the monitoring system**. There is no separate monitoring stack — the watchdog (cron every minute) both detects problems and auto-recovers:

| Check | Detection | Recovery |
|-------|-----------|----------|
| Workspace sentinel | Missing sentinel file = overlay wipe | Full workspace restore from backup |
| Gateway port | Port not in LISTEN state | Restart gateway process |
| SSH tunnel | SSH tunnel PID gone | Reconnect tunnel |
| DNS proxy | DNS not resolving inside sandbox | Restart DNS proxy script |
| Signal bridge | Bridge PID gone or orphaned | Kill orphans, restart bridge |
| Dashboard bind | Bound to 127.0.0.1 instead of 0.0.0.0 | Stop and restart port forward |

The watchdog runs via cron every minute. It is idempotent — running it when everything is healthy is a no-op. See `references/workspace-backup.md` for the full watchdog implementation.

**Key insight:** The watchdog is both monitor and recovery mechanism. If the watchdog itself fails (e.g., cron not running, nvm PATH issue), everything downstream breaks silently. Monitor the watchdog's cron log as your meta-health check:

```bash
# Verify watchdog ran recently (should show entries every minute)
grep "nemoclaw-watchdog" /var/log/syslog | tail -5
# Or check cron log directly
crontab -l | grep watchdog
```

## Alerting Patterns

No built-in alerting exists in NemoClaw 0.1.0. Implement with standard tools:

### Simple: Cron + Email/Signal

```bash
# Add to crontab — runs every 15 minutes, alerts on failure
*/15 * * * * /path/to/nemoclaw-healthcheck.sh 2>&1 | grep "FAIL" && echo "NemoClaw health issue" | signal-cli send -m - +YOUR_NUMBER
```

### Integration with Home Assistant

If your HA instance monitors the host, create a command-line sensor:

```yaml
# configuration.yaml
command_line:
  - sensor:
      name: "NemoClaw Status"
      command: "nemoclaw my-assistant status --json 2>/dev/null | jq -r '.status // \"unknown\"'"
      scan_interval: 300
```

Then create an automation to alert when status is not "Running".

## Limitations (NemoClaw 0.1.0)

| Feature | Status |
|---------|--------|
| Structured metrics (Prometheus) | Not available |
| OpenTelemetry traces | Not available |
| Built-in dashboard/UI metrics | Not available |
| Log export (JSON/syslog) | Not available — logs are plaintext |
| Per-tool rate limiting | Not available |
| Aggregated multi-sandbox view | Not available |

All monitoring relies on CLI commands and log parsing. For production deployments, wrap the health check script in your existing monitoring stack (Grafana Agent, Datadog, etc.) by parsing the CLI output.

## Related References

- `references/access-control.md` — Audit trail and log patterns
- `references/workspace-backup.md` — Watchdog auto-restore pattern
- `references/troubleshooting.md` — Diagnostic commands
