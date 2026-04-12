# NemoClaw Policy Presets

## Overview

Presets are named YAML files in `nemoclaw-blueprint/policies/presets/` that extend the baseline network policy with additional endpoints for specific services.

## Official NVIDIA Presets (9, as of NemoClaw 0.1.0)

Shipped with NemoClaw in the `nemoclaw-blueprint/policies/presets/` directory:

| Preset | Endpoints | Use Case |
|--------|-----------|----------|
| `discord` | Discord webhook API | Bot integrations |
| `docker` | Docker Hub, NVIDIA container registry | Container operations |
| `huggingface` | Hugging Face model registry | Model downloads |
| `jira` | Atlassian Jira API | Issue tracking |
| `npm` | npm and Yarn registries | Package management |
| `outlook` | Microsoft 365, Outlook | Email integration |
| `pypi` | Python Package Index | Python packages |
| `slack` | Slack API and webhooks | Chat integration |
| `telegram` | Telegram Bot API | Messaging bridge |

## Community Presets (18)

From the community repository [VoltAgent/awesome-nemoclaw](https://github.com/VoltAgent/awesome-nemoclaw) on GitHub. These are **example baselines** — review and customize before production use.

| Preset | Endpoints | Notes |
|--------|-----------|-------|
| `gitlab` | GitLab API (`/api/v4/**`) | Self-hosted: change host |
| `notion` | Notion API (`/v1/**`) | Integration token required |
| `linear` | Linear GraphQL (`/graphql`) | API key in header |
| `confluence` | Confluence/Atlassian APIs | Scope to your tenant |
| `teams` | Microsoft Teams, Graph API | OAuth required |
| `zendesk` | Zendesk API | Subdomain-specific |
| `sentry` | Sentry API | Error tracking |
| `stripe` | Stripe API (`/v1/**`) | PCI considerations |
| `cloudflare` | Cloudflare API | DNS/CDN management |
| `google-workspace` | OAuth, Gmail, Drive, Calendar | Multi-endpoint |
| `aws` | STS, S3, Bedrock | Bucket placeholders |
| `gcp` | GCP APIs | Service-specific scoping |
| `vercel` | Vercel API | Deployment management |
| `supabase` | Supabase API | Project-specific URL |
| `neon` | Neon Postgres API | Database management |
| `algolia` | Algolia Search API | App-specific |
| `airtable` | Airtable API | Base-specific |
| `hubspot` | HubSpot API | CRM integration |

## Example Custom Presets

Examples of custom presets you might create for your deployment:

| Preset | Endpoints | Purpose |
|--------|-----------|---------|
| `home-assistant` | HA via Caddy reverse proxy | Smart home control |
| `web-search` | DuckDuckGo, Google CSE | Web search capabilities |
| `caddy-proxy` | Caddy at `<PROXY_HOST>:443` | HTTPS proxy for HTTP services |
| `mcp-services` | MCP service endpoints | MCP tool access |

## Preset YAML Format

Two styles are available. Use **REST style** for APIs where you want to control which HTTP methods and paths are allowed (most common). Use **Full style** for WebSocket connections, long-lived tunnels, or internal services where method/path filtering doesn't apply.

### REST API Style (with method/path rules)

For precise HTTP method and path control:

```yaml
# SPDX header (required for official presets)
preset:
  name: service-name
  description: "Service API access (least-privilege example)"

network_policies:
  policy_id:
    name: policy-display-name
    endpoints:
      - host: api.example.com
        port: 443
        protocol: rest
        enforcement: enforce
        tls: terminate
        rules:
          - allow: { method: GET, path: "/api/v1/**" }
          - allow: { method: POST, path: "/api/v1/**" }
          - allow: { method: PATCH, path: "/api/v1/**" }
    binaries:
      - { path: /usr/local/bin/openclaw }
```

### Full Access Style (CONNECT tunnel)

For WebSocket connections or when you need unrestricted access to a host:

```yaml
preset:
  name: my-proxy
  description: "Full access via CONNECT tunnel"

network_policies:
  my_proxy:
    name: my-proxy
    endpoints:
      - host: <PROXY_HOST>
        port: 443
        access: full
    binaries:
      - { path: /usr/local/bin/node }
      - { path: /usr/local/bin/openclaw }
```

`access: full` creates a CONNECT tunnel that avoids HTTP-level timeouts — required for WebSocket and long-lived connections.

### Format Rules

- **Always** `port: 443` — OpenShell proxy only handles port 443
- **REST style**: `protocol: rest` + `enforcement: enforce` + `tls: terminate` + explicit `rules`
- **Full style**: `access: full` — no method/path filtering, used for tunnels and proxies
- **`binaries`** — scope to specific executables for least-privilege
- **`rules`** — use `method` + `path` patterns with `**` wildcards

## Applying Presets

### Interactive
```bash
nemoclaw <sandbox> policy-add
```

### Programmatic
```bash
cd ~/.nemoclaw/source && node -e "require('./bin/lib/policies.js').applyPreset('<sandbox>', '<preset-name>')"
```

### As Static Baseline
Merge preset entries into `nemoclaw-blueprint/policies/openclaw-sandbox.yaml` and re-run `nemoclaw onboard`.

## Creating a New Custom Preset

1. Create file: `nemoclaw-blueprint/policies/presets/<name>.yaml`
2. Follow the YAML format above
3. Use `port: 443` + `tls: terminate`
4. If target is HTTP: set up Caddy reverse proxy first
5. Scope `binaries` to limit access
6. Apply: `nemoclaw <sandbox> policy-add`

### Example: Custom API Preset

```yaml
preset:
  name: my-internal-api
  description: "Internal API behind Caddy TLS proxy"

network_policies:
  my_api:
    name: internal-api
    endpoints:
      - host: <PROXY_HOST>
        port: 443
        protocol: rest
        enforcement: enforce
        tls: terminate
        rules:
          - allow: { method: GET, path: "/myapi/**" }
          - allow: { method: POST, path: "/myapi/**" }
    binaries:
      - { path: /usr/local/bin/openclaw }
```

## Hardening Checklist

Before using any community preset in production:

- [ ] Review all `host` entries — verify you trust each domain
- [ ] Tighten `path` patterns — avoid `/**` wildcards where possible
- [ ] Scope `binaries` — don't use "any binary" unless necessary
- [ ] Check `method` restrictions — limit to methods you actually need
- [ ] Replace placeholder values (bucket names, tenant IDs, subdomains)
- [ ] Test in audit mode first (`enforcement: audit`) before enforcing
