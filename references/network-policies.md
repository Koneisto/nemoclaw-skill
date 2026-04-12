# NemoClaw Network Policies

## Core Concept

The OpenShell sandbox runs with a **deny-by-default** network policy. The sandbox can only reach endpoints explicitly allowed in the policy YAML. Any request to an unlisted destination is intercepted by OpenShell's proxy. NemoClaw provides presets and the TUI to manage these policies.

**NemoClaw's preset system defines policies for HTTPS on port 443; OpenShell's proxy enforces them.** All preset policies use `port: 443` with TLS termination. The OpenShell proxy at `10.200.0.1:3128` blocks unlisted traffic.

**Exception**: Direct `openshell policy set` with custom YAML can allow non-443 ports and protocols (e.g., direct access to `<HA_HOST>:8123`), but this bypasses preset enforcement and requires manual policy lifecycle management. Preset-managed policies are strongly recommended.

## How Enforcement Actually Works

**CRITICAL**: The **preset system** is the recommended enforcement mechanism over raw `openshell policy set`.

### The Preset Pipeline

1. Presets live in `nemoclaw-blueprint/policies/presets/*.yaml`
2. Applied via `nemoclaw <sandbox> policy-add` (interactive) or programmatically:
   ```bash
   cd ~/.nemoclaw/source && node -e "require('./bin/lib/policies.js').applyPreset('<sandbox>', '<preset-name>')"
   ```
3. `applyPreset()` merges with the current full policy and uses the `--wait` flag
4. Raw `openshell policy set <file>` applies policy metadata but may not open traffic immediately (see below)

### Why Raw Policy Set Is Unreliable

`openshell policy set` records the policy but lacks the preset system's merge logic and `--wait` flag. Without these, there is a race condition: the policy metadata is recorded but the proxy at `10.200.0.1:3128` may continue blocking traffic until the connection state catches up. The preset pipeline avoids this by waiting for enforcement confirmation.

**`openshell policy set` does work** — it applies session-scoped dynamic policy that resets when the sandbox stops. But it is unreliable for opening new endpoints because of the missing `--wait`. Use presets for any endpoint you need working immediately.

## Baseline Policy

Defined in `nemoclaw-blueprint/policies/openclaw-sandbox.yaml`.

### Filesystem Rules

| Path | Access |
|------|--------|
| `/sandbox`, `/tmp`, `/dev/null` | Read-write |
| `/usr`, `/lib`, `/proc`, `/dev/urandom`, `/app`, `/etc`, `/var/log` | Read-only |

### Default Network Endpoints

| Policy | Endpoints | Binaries | Rules |
|--------|-----------|----------|-------|
| `claude_code` | `api.anthropic.com:443`, `statsig.anthropic.com:443`, `sentry.io:443` | `/usr/local/bin/claude` | All methods |
| `nvidia` | `integrate.api.nvidia.com:443`, `inference-api.nvidia.com:443` | `/usr/local/bin/claude`, `/usr/local/bin/openclaw` | All methods |
| `github` | `github.com:443` | `/usr/bin/gh`, `/usr/bin/git` | All methods |
| `github_rest_api` | `api.github.com:443` | `/usr/bin/gh` | GET, POST, PATCH, PUT, DELETE |
| `clawhub` | `clawhub.com:443` | `/usr/local/bin/openclaw` | GET, POST |
| `openclaw_api` | `openclaw.ai:443` | `/usr/local/bin/openclaw` | GET, POST |
| `openclaw_docs` | `docs.openclaw.ai:443` | `/usr/local/bin/openclaw` | GET only |
| `npm_registry` | `registry.npmjs.org:443` | `/usr/local/bin/openclaw`, `/usr/local/bin/npm` | GET only |
| `telegram` | `api.telegram.org:443` | Any binary | GET, POST on `/bot*/**` |

All endpoints use TLS termination, port 443 only. Inference traffic uses `inference.local` route exclusively (OpenShell gateway handles upstream).

## Operator Approval Flow

When the agent tries to reach an unlisted host:

1. Agent makes network request to unlisted host
2. OpenShell **blocks** the connection and logs the attempt
3. TUI (`openshell term`) displays blocked request: host, port, requesting binary
4. Operator approves or denies
5. If approved: endpoint added to running policy **for current session only** (NOT persisted)

```bash
# Open TUI to monitor and approve
openshell term
```

## Modifying Policies

### Static Changes (persist across restarts)

Edit `nemoclaw-blueprint/policies/openclaw-sandbox.yaml`, then re-run:
```bash
nemoclaw onboard
```

### Dynamic Changes (session only)

Apply policy update to running sandbox:
```bash
openshell policy set <policy-file>
```
Resets to baseline when sandbox stops.

### Preset System (recommended)

```bash
# List available presets
nemoclaw <name> policy-list

# Add preset interactively
nemoclaw <name> policy-add

# Apply preset programmatically
cd ~/.nemoclaw/source && node -e "require('./bin/lib/policies.js').applyPreset('<sandbox>', '<preset-name>')"
```

## Creating Custom Presets

Create YAML in `nemoclaw-blueprint/policies/presets/<name>.yaml`:

```yaml
preset:
  name: my-service
  description: "My service API access"

network_policies:
  my_service_api:
    name: my-service-api
    endpoints:
      - host: api.example.com
        port: 443
        protocol: rest
        enforcement: enforce
        tls: terminate
        rules:
          - allow: { method: GET, path: "/api/v1/**" }
          - allow: { method: POST, path: "/api/v1/**" }
    binaries:
      - { path: /usr/local/bin/openclaw }
```

**Key rules for custom presets:**
- External HTTPS: use `port: 443` and `tls: terminate`
- Internal HTTP: use non-443 port with `access: full` (bypasses TLS relay)
- Scope `binaries` to limit which processes can use the endpoint
- Use specific `path` patterns with `**` wildcards

## Accessing Internal HTTP Services

For internal services (e.g., Home Assistant, local LLM endpoints):

1. Add a direct policy entry with `access: full` on the service's actual port
2. Access directly from sandbox: `curl http://<HOST>:<PORT>/api/...`

```yaml
network_policies:
  my_service:
    name: my_service
    endpoints:
      - host: '10.0.0.5'
        port: 8123
        access: full
    binaries:
      - { path: /usr/bin/curl }
```

**Do NOT** set up reverse proxies (Caddy, nginx) **inside the sandbox** to wrap HTTP services in TLS. Direct policy entries with `access: full` are simpler, faster, and survive rebuilds without extra infrastructure.

### When You DO Need a Host-Side Reverse Proxy

If an **external** HTTP-only service has no HTTPS endpoint and you can't add one, run a reverse proxy on the **host** (not in the sandbox):

```
Sandbox → policy allows proxy-host:443 → Host Caddy/nginx (TLS termination) → external HTTP service
```

This is rare — most services offer HTTPS. Only use this pattern when the external service genuinely has no TLS endpoint. For internal LAN services (like Home Assistant), use `access: full` on the actual port instead.

## Policy Schema (OpenShell Level)

Understanding the full OpenShell policy schema helps when going beyond presets.

### Top-Level Structure

A complete policy has four sections:

| Section | Scope | Modifiable at Runtime? |
|---------|-------|----------------------|
| `filesystem_policy` | Read-only / read-write paths | **Static** — locked at creation |
| `landlock` | Kernel LSM enforcement mode | **Static** — locked at creation |
| `process` | User/group identity for agent | **Static** — locked at creation |
| `network_policies` | Outbound network access | **Dynamic** — hot-reloadable |

Static sections require `nemoclaw <name> destroy` + recreate to change. Network policies can be updated on running sandboxes.

### Landlock Modes

| Mode | Behavior |
|------|----------|
| `best_effort` | Skips inaccessible paths silently (default) |
| `hard_requirement` | Aborts if any specified path doesn't exist |

### Network Policy Features

- **Wildcard domains**: `*.example.com` matches all subdomains
- **L7 enforcement**: Distinguishes HTTP methods (GET vs POST) — read-only policies block mutations
- **Access presets**: `read-only`, `read-write`, `full` (CONNECT tunnel)
- **Rule objects**: Match on `method`, `path` (glob patterns), and `query` parameters
- **Binary scoping**: Restrict which executables can use an endpoint

### Global Policies

Apply a policy to all sandboxes on a gateway:
```bash
openshell policy set --global --policy ./global-policy.yaml
openshell policy delete --global
```

### Policy Inspection

```bash
# Export current active policy (full, including all merged presets)
openshell policy get <sandbox> --full > current-policy.yaml

# List applied policy names
openshell policy list <sandbox>
```

### Process Restrictions

The `process` section specifies sandbox user identity. Neither `run_as_user` nor `run_as_group` may be set to `root` or `0`.

## Related References

- **`presets.md`** — Full list of official and community presets with YAML format details
- **`troubleshooting.md`** — Diagnosing blocked traffic and policy application issues

## Policy YAML Structure

Each policy entry has:

- **`endpoints`**: Host:port pairs the sandbox can reach
- **`binaries`**: Executables allowed to use this endpoint
- **`rules`**: HTTP methods and path patterns permitted
- **`enforcement`**: `enforce` (default) or `audit`
- **`tls`**: `terminate` (required for port 443)
