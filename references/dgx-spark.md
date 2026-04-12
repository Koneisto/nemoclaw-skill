# NemoClaw on DGX Spark

## Overview

DGX Spark (Grace Hopper) runs Ubuntu 24.04 with Docker pre-installed. NemoClaw requires specific configuration for cgroup v2 compatibility and can use local vLLM for inference instead of cloud providers.

## Prerequisites

- Docker v28.x/29.x (pre-installed on Spark)
- Node.js 22.16+ (auto-installed by NemoClaw)
- OpenShell CLI
- For local inference: vLLM running on host

## Installation Steps

### 1. Install OpenShell CLI
```bash
curl -fsSL https://raw.githubusercontent.com/NVIDIA/OpenShell/main/install.sh | bash
```

### 2. Clone NemoClaw
```bash
git clone https://github.com/NVIDIA/NemoClaw.git ~/.nemoclaw/source
cd ~/.nemoclaw/source
```

### 3. Run Spark Setup (sudo required)
```bash
sudo nemoclaw setup-spark
```
This fixes:
- cgroup v2 delegation: sets `"default-cgroupns-mode": "host"` in Docker daemon config
- Docker permissions: adds user to docker group
- Other Ubuntu 24.04 specific issues

### 4. Run Installer
```bash
./install.sh
# or
curl -fsSL https://www.nvidia.com/nemoclaw.sh | bash
```

### 5. Onboard
```bash
nemoclaw onboard
```

## Cgroup v2 Fix Details

Ubuntu 24.04 uses cgroup v2 by default. NemoClaw's internal k3s requires `"default-cgroupns-mode": "host"` in `/etc/docker/daemon.json`:

```json
{
  "default-cgroupns-mode": "host"
}
```

Applied automatically by `setup-spark`. Requires Docker daemon restart.

## Local vLLM Inference

### vLLM Setup on Spark

```bash
# Verify NVIDIA runtime
docker run --rm --runtime=nvidia --gpus all ubuntu nvidia-smi

# vLLM runs as Docker container
# Example: Qwen3.5-35B-A3B MoE at port 8001
# Model ID: "model"
# Max model length: 65536
# GPU memory utilization: 0.50 (avoid OOM with CUDA graphs)
```

### NemoClaw with Local vLLM

```bash
cd ~/.nemoclaw/source && \
  NEMOCLAW_EXPERIMENTAL=1 \
  NEMOCLAW_GW_PORT=8088 \
  VLLM_PORT=8001 \
  NEMOCLAW_SANDBOX_NAME=my-assistant \
  NEMOCLAW_PROVIDER=vllm \
  NEMOCLAW_MODEL=model \
  nemoclaw onboard
```

Key environment variables:
- `NEMOCLAW_EXPERIMENTAL=1` — required to enable vLLM provider
- `VLLM_PORT` — port where vLLM is running (default: 8000)
- vLLM base URL becomes `http://host.openshell.internal:<VLLM_PORT>/v1`

### Custom Patches for Spark

Common modifications in `~/.nemoclaw/source`:

| File | Patch | Purpose |
|------|-------|---------|
| `Dockerfile` | `TZ=<YOUR_TIMEZONE>` | Timezone |
| `Dockerfile` | `CHAT_UI_EXTRA_ORIGINS` | Allow LAN dashboard access |
| `Dockerfile` | `maxTokens=8192` | Increase max output tokens |
| `Dockerfile` | `USER sandbox` before `.bashrc` | Fix file ownership |
| `bin/lib/onboard.js` | `VLLM_PORT` env var | Custom vLLM port detection |
| `bin/lib/local-inference.js` | `VLLM_PORT` env var | Custom vLLM port for base URL, validation, health check |
| `bin/nemoclaw.js` | `0.0.0.0:18789` | Bind dashboard to all interfaces |
| `openclaw-sandbox.yaml` | Comment out `deny` field | **Obsolete if running OpenShell 0.0.23+** — only needed for 0.0.16. Check your version with `openshell --version` before applying. |

## DGX Spark Specific Troubleshooting

### Dashboard not loading on Spark
**Cause**: Gateway not running, wrong port, or origin restriction.
**Fix**:
```bash
# Verify gateway is running inside sandbox
nemoclaw <name> connect
pgrep -f openclaw

# Re-forward with LAN binding
openshell forward start --background 0.0.0.0:18789 <sandbox>
```
If dashboard loads locally but not from LAN: check `CHAT_UI_EXTRA_ORIGINS` includes the accessing host's address in the Dockerfile patch.

### "Connection refused" to vLLM from sandbox
**Cause**: vLLM not running, or `VLLM_PORT` mismatch.
**Fix**:
```bash
# From host — verify vLLM responds
curl http://localhost:8001/v1/models

# Check sandbox can reach it via gateway
nemoclaw <name> status  # shows active inference endpoint
```

### CoreDNS crashes
**Cause**: k3s CoreDNS instability on Spark.
**Fix**: Restart the NemoClaw containers or re-run `nemoclaw onboard`.

### Port 3000 conflict
**Cause**: NVIDIA AI Workbench uses port 3000.
**Fix**: Configure NemoClaw to use different port or stop AI Workbench.

### OOM on Spark
**Fix**:
```bash
sudo sh -c 'sync; echo 3 > /proc/sys/vm/drop_caches'
```

### CUDA graphs + high gpu-mem cause OOM
**Cause**: CUDA graph compilation requires extra memory on top of model weights.
**Fix**: Set `--gpu-memory-utilization 0.50` (or lower) when launching vLLM to leave headroom for CUDA graphs. See also `references/inference-profiles.md` OOM Prevention section.

### Image pull failures
**Cause**: Network or registry issues.
**Fix**: Check Docker network connectivity and retry. May need proxy configuration.

### Sandbox creation fails with stale port forward
**Cause**: Previous onboard left a stale port forward.
**Fix**: Re-run `nemoclaw onboard` — it cleans up automatically.

## Architecture Notes

NemoClaw on Spark runs within Docker containers with embedded k3s:
- Gateway origin restrictions limit external dashboard connections
- Use `CHAT_UI_EXTRA_ORIGINS` to allow LAN access
- Port forwarding: `openshell forward start --background 0.0.0.0:18789 <sandbox>`

## Signal Bridge on Spark

SSH-based bridge (not HTTP gateway API — security requirement). See `references/signal-bridge.md` for full details.

**Critical for cron/watchdog:** Use absolute path to `node` — nvm is not in cron's PATH:

```bash
NODE="${HOME}/.nvm/versions/node/v$(node -v | tr -d v)/bin/node"
"$NODE" scripts/signal-bridge.js
```

**Do NOT use** `./scripts/start-services.sh` for production — it lacks orphan cleanup, PID management, and absolute paths. Use the watchdog pattern instead (see `references/signal-bridge.md` "Production Hardening").

- Script: `~/.nemoclaw/source/scripts/signal-bridge.js`
- Transport: SSH via `openshell ssh-proxy`
- Session ID: `--session-id sig-PHONENUMBER`
- Features: noise filtering, exit 255 retry, thinking stripping, busy number serialization, time context injection

## Workspace Persistence on Spark

**WARNING**: Workspace is on the overlay filesystem and gets **wiped on every pod restart** (reboot, crash). This is NOT a Spark-specific issue — it affects all NemoClaw deployments. See `references/workspace-backup.md` for the automated recovery (watchdog) pattern.

For always-on Spark deployments, a watchdog cron job (every minute) is essential to auto-restore workspace, gateway, tunnel, DNS, and Signal bridge after reboots.
