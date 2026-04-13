# Signal Bridge for NemoClaw

## Why Signal Over Telegram

NemoClaw officially ships with a Telegram bridge, but Signal offers significant privacy advantages for sandboxed AI agents:

| Aspect | Telegram | Signal |
|--------|----------|--------|
| Message routing | Cloud — all messages pass through Telegram servers | Local — messages processed by a Docker container on your host |
| Authentication | Bot token from @BotFather (cloud-managed) | Phone number linked to your Signal account (locally managed) |
| Message protocol | HTTP polling to `api.telegram.org` | WebSocket to local `signal-cli-rest-api` container |
| Infrastructure | Telegram cloud servers | Self-hosted Docker container |
| Access control | Chat ID allowlist | Phone number allowlist |
| Message limit | 4096 characters | 2000 characters |
| Privacy model | Telegram sees all messages | Messages never leave your network |

**Key advantage:** With Signal, the entire message pipeline stays on your infrastructure. No third-party cloud service processes or stores your agent's conversations.

## Architecture

```
Signal App (phone)
    │
    ▼
Signal Servers (E2EE delivery only)
    │
    ▼
signal-cli-rest-api (Docker, your host)
    │ WebSocket
    ▼
signal-bridge.js (Node.js process)
    │ SSH tunnel
    ▼
OpenShell sandbox (OpenClaw agent)
    │ inference.local
    ▼
LLM provider (via OpenShell gateway)
    │
    ▼
Response flows back the same path
```

**Trust boundary:** The bridge connects to the sandbox via SSH — the same mechanism as `nemoclaw connect`. Messages enter and exit the sandbox through OpenShell's authenticated SSH proxy. The bridge process never bypasses the security boundary.

**Why SSH, not HTTP gateway:** The OpenClaw gateway exposes an HTTP API, but using it from outside the sandbox would bypass NemoClaw's trust model. SSH ensures all agent interactions are mediated by OpenShell's policy engine. This is a deliberate security design choice.

## Prerequisites

1. **Docker** running on the host
2. **Signal account** — a phone number registered with Signal (can be your personal number via linked device)
3. **NemoClaw sandbox** running and SSH-accessible via `ssh sandbox` (see SKILL.md "SSH Access to Sandbox" section for alias setup)
4. **Node.js 20+** on the host (already required by NemoClaw)
5. **qrencode** (optional) — for QR code generation during account linking: `sudo apt install qrencode`

## Setup

### Step 1: Run signal-cli-rest-api

The bridge requires [signal-cli-rest-api](https://github.com/bbernhard/signal-cli-rest-api) — a Docker container that provides a REST/WebSocket interface to the Signal protocol.

```yaml
# docker-compose.yml
services:
  signal-cli:
    image: bbernhard/signal-cli-rest-api:latest
    environment:
      - MODE=json-rpc
    ports:
      - "127.0.0.1:8080:8080"  # host:container — adjust host port if 8080 is taken
    volumes:
      - ./signal-data:/home/.local/share/signal-cli
    restart: unless-stopped
```

```bash
docker compose up -d
```

**Important:** Bind to `127.0.0.1` only — the Signal API should not be exposed to the network.

### Step 2: Link your Signal account

The recommended method is **linked device** — your phone keeps the number, the bridge gets a secondary device credential:

```bash
# Generate a QR code link
docker exec -it signal-cli signal-cli link -n "NemoClaw Bridge"
```

This outputs a `tsdevice:/` URI. Either:
- Convert it to a QR code: `echo "URI" | qrencode -t ANSIUTF8`
- Or paste it into an online QR generator

Then scan the QR code with your Signal app: **Settings → Linked Devices → Link New Device**.

Verify the link:

```bash
docker exec signal-cli signal-cli listAccounts
```

### Step 3: Set environment variables

```bash
export SIGNAL_PHONE_NUMBER="+15550100000"            # Your Signal number (E.164 format)
export ALLOWED_SIGNAL_NUMBERS="+15550200000"          # Comma-separated allowlist (optional)
export NVIDIA_API_KEY="your-api-key"                # Or inference provider key
export NEMOCLAW_SANDBOX="your-sandbox-name"         # Sandbox to connect to
export SIGNAL_API_URL="http://127.0.0.1:8080"       # signal-cli-rest-api URL (default)
```

If `ALLOWED_SIGNAL_NUMBERS` is unset, all incoming Signal messages are processed. Set it to restrict access.

### Step 4: Start the bridge

```bash
# Via NemoClaw service manager (if configured in start-services.sh)
nemoclaw start

# Or directly
node ~/.nemoclaw/source/scripts/signal-bridge.js
```

### Step 5: Test

Send a message from an allowed Signal number. The bridge should:
1. Show a typing indicator while processing
2. Respond within the sandbox's inference timeout
3. Log the interaction to stdout

## Bridge Features

The Signal bridge implements the same patterns as the Telegram bridge:

| Feature | Details |
|---------|---------|
| **Per-sender serialization** | Only one message per phone number processes at a time — prevents race conditions with the LLM |
| **Read receipts** | Sends read receipt immediately when a message is received — sender sees blue double-check marks |
| **Typing indicators** | Sends Signal typing indicators every 4 seconds while the agent is processing |
| **Message chunking** | Splits responses longer than 2000 characters into multiple messages |
| **Response filtering** | Strips LLM thinking blocks (`<think>...</think>`), boot noise, and reasoning leakage |
| **Auto-reconnect** | WebSocket reconnects automatically after disconnection (5-second backoff) |
| **Session isolation** | Each sender gets their own OpenClaw session (key: `sig-<phone-digits>`) |
| **SSH retry** | Retries sandbox connection up to 2 times with backoff on SSH exit code 255 |

## Security Considerations

- **Bind signal-cli-rest-api to localhost only** — never expose the Signal API to the network
- **Use `ALLOWED_SIGNAL_NUMBERS`** — without it, anyone who knows the bridge's Signal number can interact with your agent
- **SSH trust boundary** — the bridge authenticates to the sandbox via OpenShell's SSH proxy, same as manual terminal access
- **No credentials in sandbox** — the bridge passes messages, not API keys. Inference credentials are managed by OpenShell on the host
- **Linked device, not registration** — use `link` not `register`. Linking creates a secondary device credential; registration would take over the phone number

## Production Hardening

### Time Context Injection

The LLM inside the sandbox has no clock. The bridge prepends a system context line to each message:

```
[system: 2026-04-07 17:09. Use this for time awareness. Do not echo this line.]
<actual user message>
```

The format is neutral (ISO date + 24h time). Date/time formatting rules belong in the agent's USER.md, not the bridge — this ensures consistent formatting across all channels.

### Watchdog Auto-Start

The bridge is a long-running Node.js process that can crash or get killed. For always-on deployments, add it to the NemoClaw watchdog (cron every minute):

```bash
# In the watchdog script:
NODE="${HOME}/.nvm/versions/node/v$(node -v | tr -d v)/bin/node"

signal_bridge_running() {
    [[ -f "$PIDFILE" ]] && kill -0 "$(cat "$PIDFILE")" 2>/dev/null
}

if ! signal_bridge_running; then
    # Use your host port from docker-compose (8080 in example, may differ in production)
    if curl -sf http://127.0.0.1:${SIGNAL_API_PORT:-8080}/v1/about >/dev/null 2>&1; then
        # Kill orphans to prevent duplicate instances
        orphans=$(pgrep -f "node scripts/signal-bridge.js" 2>/dev/null || true)
        [[ -n "$orphans" ]] && echo "$orphans" | xargs kill 2>/dev/null && sleep 1

        cd "$NEMOCLAW_SOURCE"
        nohup "$NODE" scripts/signal-bridge.js >> "$LOG" 2>&1 &
        echo $! > "$PIDFILE"
    fi
fi
```

**Critical: use absolute path to `node`** — nvm binaries are not in cron's PATH.

### Orphan Process Cleanup

If the bridge crashes, the Node.js process may linger. Multiple instances cause **duplicate messages** (each instance has its own WebSocket to signal-cli, so each processes the same incoming message independently).

The watchdog handles this by killing orphans before starting a new instance (see above). The PID file alone isn't sufficient.

**Important (SKILL.md rule #17):** `pgrep -f "signal-bridge.js"` matches ANY process whose command line contains that string — including Claude Code or other diagnostic sessions. Filter by `/proc/PID/comm` to verify it's actually node:

```bash
for pid in $(pgrep -f "signal-bridge.js" 2>/dev/null); do
    comm=$(cat "/proc/$pid/comm" 2>/dev/null || true)
    if [[ "$comm" == node* ]]; then
        echo "$pid"  # Real bridge — safe to manage
    fi
done
```

### Log Rotation

The bridge logs to a single file that grows indefinitely. For production:

```bash
# Truncate on watchdog restart (bridge re-logs the banner on start)
truncate -s 0 "$SIGNAL_BRIDGE_LOG"
```

Or use `logrotate` for more sophisticated rotation.

## Troubleshooting

### Bridge can't connect to signal-cli-rest-api

```
Error: connect ECONNREFUSED 127.0.0.1:8080
```

Check that the Docker container is running:
```bash
docker ps | grep signal-cli
docker logs signal-cli
```

### WebSocket disconnects repeatedly

The signal-cli-rest-api container may need a restart if the WebSocket endpoint becomes unresponsive:
```bash
docker restart signal-cli
```

### Messages not arriving

1. Verify the account is linked: `docker exec signal-cli signal-cli listAccounts`
2. Check that `SIGNAL_PHONE_NUMBER` matches the linked account exactly (E.164 format with `+`)
3. If using `ALLOWED_SIGNAL_NUMBERS`, confirm the sender's number is in the allowlist

### Agent responds with errors or garbled output

The bridge filters LLM noise, but some model-specific output patterns may leak through. Check:
- Is the sandbox's inference provider running? (`nemoclaw <name> status`)
- Is the model loaded? Test directly: `nemoclaw <name> connect` then send a test message

### SSH connection fails

Same as any NemoClaw SSH issue — see the main troubleshooting guide (`references/troubleshooting.md`). Common causes:
- SSH handshake secret regenerated after container restart
- Sandbox not in `Ready` state
- Gateway not running

## Comparison with Official Telegram Bridge

The Signal bridge follows the same architectural pattern as NemoClaw's official Telegram bridge (`scripts/telegram-bridge.js`). Key differences in implementation:

| Aspect | Telegram Bridge | Signal Bridge |
|--------|----------------|---------------|
| Transport | HTTP long-polling (`getUpdates`) | WebSocket (persistent connection) |
| Message source | `message.text` + `message.chat.id` | `envelope.dataMessage.message` + `envelope.source` |
| Reply method | `sendMessage` REST call | `/v1/send` REST call |
| Session key | `tg-<chat-id>` | `sig-<phone-digits>` |
| Typing | `sendChatAction` | `/v1/typing-indicator` |
| Read receipts | Not implemented | `POST /v1/receipts/{number}` with `receipt_type: "read"` |

Both bridges use identical patterns for SSH sandbox execution, message chunking, and response filtering.

## Related

- `references/sessions.md` — Session management and reset procedures
- `references/troubleshooting.md` — General NemoClaw troubleshooting
- [signal-cli-rest-api](https://github.com/bbernhard/signal-cli-rest-api) — Docker image for Signal protocol access
- [signal-cli](https://github.com/AsamK/signal-cli) — Underlying Signal CLI tool
