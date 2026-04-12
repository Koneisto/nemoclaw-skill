# Access Control & Audit (OpenShell + OpenClaw)

## Access Control Model

The NemoClaw stack does not have traditional RBAC with named roles and role assignments. Instead, access control is enforced by **three components at different layers**: OpenShell provides infrastructure-level enforcement (network + filesystem), OpenClaw provides application-level scoping (skill allowlist), and NemoClaw provides the operator interface to manage policies.

### 1. Binary Scoping (Network Layer — OpenShell)

OpenShell's network policies specify which **executable** can reach which **endpoint**. The binary identity acts as the role:

```yaml
# Only curl can reach the weather API
network_policies:
  weather_api:
    endpoints:
      - host: api.weatherapi.com
        port: 443
        tls: terminate
    binaries:
      - { path: /usr/bin/curl }

# Only openclaw can reach the inference endpoint
network_policies:
  inference:
    endpoints:
      - host: inference.local
        port: 443
    binaries:
      - { path: /usr/local/bin/openclaw }
```

When the agent calls `exec python3 helper.py` and the helper script runs `curl api.weatherapi.com`, OpenShell's proxy identifies the requesting binary as `/usr/bin/curl` and matches it against policy. The agent process (`/usr/local/bin/openclaw`) cannot directly reach `api.weatherapi.com` — only `curl` can.

This is the closest thing to "role-based" access in the stack: **binary identity = role, policy entries = permissions**. OpenShell enforces this; NemoClaw provides the preset system and TUI (`openshell term`) to manage it.

See `references/network-policies.md` for the full policy schema.

### 2. Filesystem Scoping (Kernel Layer — OpenShell)

OpenShell's Landlock LSM confines all tool execution to:

| Path | Access | Tools Affected |
|------|--------|---------------|
| `/sandbox`, `/tmp`, `/dev/null` | Read-write | `read`, `write`, `exec` (file output) |
| `/usr`, `/lib`, `/proc`, `/etc`, `/var/log`, `/app` | Read-only | `read` (can read), `write` (blocked) |
| Everything else | Denied | All tools |

Filesystem policy is **static** — locked at sandbox creation. Cannot be hot-reloaded. Changes require `nemoclaw <name> destroy` + recreate.

### 3. Skill Scoping (Application Layer — OpenClaw)

This is a **base OpenClaw feature**, not NemoClaw-specific. `openclaw.json` → `skills.allowBundled` controls which skill instruction files are loaded into the agent's context:

```json5
{
  skills: {
    allowBundled: ["healthcheck", "skill-creator"]  // only these bundled skills load
  }
}
```

If a skill is not in the allowlist, the agent never sees its instructions and won't generate the corresponding tool calls. This works in both plain OpenClaw and NemoClaw. It is **soft scoping** — the agent could theoretically attempt the tool call if it knows how from training data, but without skill instructions it rarely does. Smaller models (Qwen3.5-35B) are more likely to improvise; larger models follow skill instructions more reliably.

## Monitor-Before-Enforce Pattern

For new deployments or after significant policy changes, run in audit mode before enforcing:

1. Set `enforcement: audit` in policy entries (logs decisions but does **not** block traffic):
   ```yaml
   network_policies:
     new_service:
       endpoints:
         - host: api.newservice.com
           port: 443
           tls: terminate
           enforcement: audit    # <-- logs only, does not block
       binaries:
         - { path: /usr/bin/curl }
   ```

2. Monitor with `openshell logs` to see what would be blocked:
   ```bash
   openshell logs <sandbox> --level warn --tail
   ```

3. Review log output — each denied/audited request shows host, port, method, path, binary

4. Adjust policy entries based on observed traffic patterns

5. Switch to `enforcement: enforce` when confident the policy covers all legitimate traffic

**Recommended**: Run audit mode for 48 hours on new deployments before switching to enforce.

## Audit Trail

### What Gets Logged

Every outbound connection from the sandbox generates a log entry:

| Event | Log Level | Key Fields |
|-------|-----------|------------|
| Policy allow | DEBUG | host, port, binary, method, path |
| Policy deny | WARN | host, port, binary, method, path, reason |
| Inference route | INFO | provider, model |
| TLS termination | DEBUG | host, SNI |
| Operator approval | INFO | host, approved/denied, session |

### Viewing Audit Logs

```bash
# Stream all policy decisions (allow + deny)
openshell logs <sandbox> --tail

# Only denials (most useful for troubleshooting and auditing)
openshell logs <sandbox> --level warn --tail

# Filter by time window
openshell logs <sandbox> --since 1h --level warn

# Gateway/control plane diagnostic logs
openshell doctor logs --tail
```

### Tool Call History (Session Logs)

Policy logs show network-level decisions. For application-level tool call history, query session logs inside the sandbox:

```bash
# All tool calls made by the agent
ssh sandbox 'grep "tool_calls" /sandbox/.openclaw-data/agents/main/sessions/*.jsonl'

# Count tool calls per session
ssh sandbox 'grep -c "tool_calls" /sandbox/.openclaw-data/agents/main/sessions/*.jsonl'

# Exec commands only (most common tool type)
ssh sandbox 'grep "\"name\":\"exec\"" /sandbox/.openclaw-data/agents/main/sessions/*.jsonl'
```

### Example Log Patterns

```
# L7 policy denial — agent tried to reach an unlisted host
[sandbox] [WARN] l7_decision=deny method=POST path=/api/v1/send host=api.blocked.com binary=/usr/bin/curl

# Allowed connection
[sandbox] [DEBUG] l7_decision=allow method=GET path=/data/2.5/weather host=api.openweathermap.org binary=/usr/bin/curl

# Binary identification at proxy
[sandbox] [DEBUG] proxy_connect binary=/usr/bin/curl target=api.weatherapi.com:443

# Audit mode (would-deny but not blocked)
[sandbox] [WARN] l7_decision=audit method=GET path=/api/v1/status host=api.newservice.com action=would_deny
```

## Limitations (NemoClaw 0.1.0 Alpha)

| Feature | Status | Notes |
|---------|--------|-------|
| Binary scoping | Available | Full support via network policy YAML |
| Filesystem scoping | Available | Landlock LSM, static at creation |
| Skill scoping | Available | Via `skills.allowBundled` in openclaw.json |
| Audit logging | Available | Plaintext via `openshell logs` |
| Audit mode | Available | Per-policy-entry `enforcement: audit` |
| Per-tool rate limiting | Not available | Planned for future release |
| Structured log export (JSON/OTLP) | Not available | Logs are plaintext only |
| OpenTelemetry integration | Not available | No OTLP endpoint support |
| Aggregated dashboards | Not available | Use grep/jq on log output |
| Named RBAC roles | Not available | Binary identity is the role mechanism |
| Global audit mode toggle | Not available | Must set per-policy-entry |

## Related References

- `references/tool-calling.md` — Tool calling model, built-in tools, model compatibility
- `references/network-policies.md` — Full network policy schema and preset system
- `references/presets.md` — Official and community preset policies
- `references/troubleshooting.md` — Diagnosing blocked traffic
