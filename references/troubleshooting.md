# NemoClaw Troubleshooting Guide

## Quick Reference — Jump to Your Problem

| Symptom | Section |
|---------|---------|
| Agent has no personality / re-introduces itself | Post-Reboot: Workspace wiped |
| `BOOTSTRAP.md` keeps coming back | Post-Reboot: BOOTSTRAP.md reappears |
| Cron scripts fail silently | Post-Reboot: Cron scripts fail |
| Agent makes up weather/sensor data | Post-Reboot: Agent fabricates data |
| TTS reads symbols literally | Post-Reboot: TTS reads symbols |
| SSH broken after reboot | Runtime: SSH connection broken |
| Traffic blocked despite policy | Network: Policy applied but blocked |
| 403 from proxy | Network: Proxy returns 403 |
| Dashboard not loading | OpenClaw: Dashboard URL not loading |
| Model connection refused | OpenClaw: Connection refused to model |
| OOM / GPU memory | OpenClaw: Out of memory |
| `openshell` / `node` not found in cron | Post-Reboot: Cron scripts fail |
| Cron jobs missing after restore | Post-Reboot: Cron jobs disappeared |
| `nemoclaw` command not found | Installation: not found after install |

## Post-Reboot Issues

### `nemoclaw` not found after install
**Cause**: nvm/fnm PATH not updated in current shell session.
**Fix**: `source ~/.bashrc` or open a new terminal.

### Unsupported platform
**Cause**: NemoClaw requires Ubuntu 22.04 LTS+.
**Fix**: Verify you're on a supported Linux distribution.

### Node.js version too old
**Cause**: NemoClaw requires Node.js 22.16+.
**Fix**:
```bash
node --version
nvm install 22
nvm use 22
```

### Docker not running
**Fix**: `sudo systemctl start docker`

### npm EACCES permission error
**Cause**: Don't run npm with sudo.
**Fix**:
```bash
mkdir -p ~/.npm-global
npm config set prefix ~/.npm-global
export PATH=~/.npm-global/bin:$PATH  # add to ~/.bashrc
```

### Port 18789 in use
**Fix**:
```bash
sudo lsof -i :18789
kill <PID>
```

## Onboarding Issues

### Cgroup v2 errors (Ubuntu 24.04, DGX Spark, WSL2)
**Cause**: Docker not configured for cgroup v2 delegation.
**Fix**:
```bash
sudo nemoclaw setup-spark
nemoclaw onboard
```
This sets `"default-cgroupns-mode": "host"` in Docker daemon config.

### Invalid sandbox name
**Cause**: Name doesn't follow RFC 1123 (lowercase alphanumeric + hyphens).
**Fix**: Use names like `my-assistant`, `dev1`, `home-agent`.

### Sandbox creation fails on DGX
**Cause**: Stale port forward from previous onboard or DNS not propagated.
**Fix**: Re-run `nemoclaw onboard` — it cleans up stale port forwards automatically.

### Inference validation fails
**Cause**: Provider endpoint unreachable or invalid model name.
**Fix**: Verify provider credentials and endpoint accessibility from the host.

## Post-Reboot Issues

### Workspace wiped after reboot (NVIDIA/NemoClaw#486)
**Cause**: Workspace lives on the pod's overlay filesystem, NOT a PVC. Every pod restart resets it to image defaults.
**Symptoms**: Agent has no personality, re-introduces itself, doesn't know the user, default SOUL.md.
**Fix**: Restore workspace from backup and reset sessions. For always-on deployments, use the watchdog auto-restore pattern — see `references/workspace-backup.md` "Automated Recovery" section.

### Agent re-introduces itself after restore
**Cause**: `BOOTSTRAP.md` still exists in the workspace (either not deleted after initial setup, or restored from backup that contains it). Session reset causes agent to follow bootstrap instructions.
**Fix**:
```bash
ssh sandbox "rm -f /sandbox/.openclaw-data/workspace/BOOTSTRAP.md"
```
Also remove from ALL backups to prevent it coming back on next restore. See SKILL.md rule #11 for the complete procedure.

### Cron scripts fail: `openshell`/`node` not found
**Cause**: `openshell` and `node` are installed via nvm/npm into user-local paths. Cron uses a minimal PATH that doesn't include `~/.local/bin` or `~/.nvm/`.
**Symptoms**: Nightly backups silently produce empty output. Watchdog can't start Signal bridge. Log shows "command not found".
**Fix**: Use absolute paths in all cron-executed scripts (see SKILL.md rule #12):
```bash
OPENSHELL="${HOME}/.local/bin/openshell"
NODE="${HOME}/.nvm/versions/node/v$(node -v | tr -d v)/bin/node"
```

### Agent fabricates weather/sensor data instead of fetching it
**Cause**: Smaller LLMs (Qwen3.5-35B) skip skill-based curl commands and generate plausible-looking but wrong numbers. The model "knows" what a weather response looks like and fabricates one.
**Fix**: Update the skill to say "use the **exec tool** to run this curl". The exec tool is a framework-level tool — the LLM produces a tool_call and the runtime executes it. The model cannot fabricate the result. Also add a global rule in SOUL.md: "NEVER fabricate real-world data."

### TTS reads "C" or "m/s" instead of spoken words
**Cause**: Agent outputs symbols (°C, m/s, %) that TTS reads literally.
**Fix**: Add global unit rule in SOUL.md: "Always spell out units — write 'degrees' not '°C', 'meters per second' not 'm/s'". Apply to ALL channels (not just voice) so responses work everywhere.

### Watchdog reports "Gateway failed to start" but gateway is running
**Cause**: OpenClaw 2026.4.5+ auto-starts the gateway. The watchdog tries to start a second instance → "already running" error. Also, `ss -tlnp` inside the sandbox doesn't show process names (insufficient privileges), so checks for "openclaw" in the output always fail.
**Fix**: Update gateway check to look for LISTEN state only (`ss -tln sport = :18789 | grep -q LISTEN`), and skip `start_gateway` if port is already listening.

### BOOTSTRAP.md reappears after OpenClaw upgrade
**Cause**: New OpenClaw versions re-create default workspace files including BOOTSTRAP.md.
**Fix**: Delete after every upgrade, not just initial setup: `ssh sandbox "rm /sandbox/.openclaw-data/workspace/BOOTSTRAP.md"`. Also remove from backups.

### Cron jobs "disappeared" after restore
**Cause**: Cron jobs are stored in `/sandbox/.openclaw-data/cron-jobs.json`. Restoring the file from backup puts the JSON back, but OpenClaw's cron scheduler only reads it at startup. Jobs won't fire until re-registered.
**Fix**: After restoring `cron-jobs.json`, re-register each job via `openclaw cron add` — or restart the gateway process to force a reload. See `references/workspace-backup.md` Step 12.

## Runtime Issues

### SSH connection broken after reboot
**Cause**: OpenShell regenerates `SSH_HANDSHAKE_SECRET` on every container restart (NemoClaw#888, OpenShell#487). See also SKILL.md rule #4.
**Fix**: Hardcode the sandbox's secret in the container's helm template:
```bash
# Enter the openshell cluster container:
docker exec -it openshell-cluster-nemoclaw bash

# Edit the template to replace the placeholder with the actual secret:
# /opt/openshell/manifests/openshell-helmchart.yaml
# Find: __SSH_HANDSHAKE_SECRET__
# Replace with the sandbox's actual secret value (from initial creation)
```
The fix persists in the container's writable layer across restarts. Only resets if the container is `docker rm`'d + recreated.

**NEVER** use `openshell gateway start --recreate` without being prepared to recreate all sandboxes.

### Sandbox shows as stopped
**Fix**: `nemoclaw onboard` to recreate from same blueprint and policy.

### Inference requests time out
**Fix**:
```bash
nemoclaw <name> status  # check active provider and endpoint
```
Verify provider reachable from host. Check network policy rules. Verify credentials.

### Agent cannot reach external host
**Cause**: Host not in network policy.
**Fix**:
```bash
openshell term  # see blocked requests, approve/deny
```
For permanent access: add endpoint to network policy preset.

### DNS not working in sandbox (EAI_AGAIN)
**Cause**: DNS proxy not running after rebuild.
**Fix**:
```bash
bash scripts/setup-dns-proxy.sh nemoclaw <sandbox>
```

### Blueprint run failed
**Fix**: Check logs for error details:
```bash
nemoclaw <name> logs --follow
```

### TLS certificate issues after reboot
**Fix**:
```bash
openshell gateway destroy
nemoclaw onboard  # re-creates gateway with fresh certs
```

### `.bashrc`/`.profile` ownership issue
**Cause**: Files created during Docker build are root-owned; sandbox user can't write.
**Fix**:
```bash
ssh sandbox "rm -f /sandbox/.bashrc /sandbox/.profile"
```

### `openshell sandbox upload` creates wrapper directories
**Cause**: Known bug — `file.md` becomes `file.md/file.md`.
**Fix**: Always use SSH for file transfer:
```bash
# Upload
ssh sandbox "cat > /sandbox/path/to/file" < local/file

# Download
ssh sandbox "cat /sandbox/path/to/file" > local/file
```

### `openclaw.json` changes don't take effect
**Cause**: File is read-only (Landlock protected).
**Fix**: Changes require sandbox rebuild via `nemoclaw onboard`.

### Gateway token changed after rebuild
**Cause**: Gateway generates new token on rebuild.
**Fix**: Update any external systems that depend on the gateway token.

## OpenClaw / Dashboard Issues

### Dashboard URL not loading
**Cause**: Gateway not running or wrong host/port.
**Fix**:
```bash
# Check gateway is running
pgrep -f openclaw
ps aux | grep openclaw

# Restart gateway (inside sandbox)
nohup openclaw gateway run > /tmp/gateway.log 2>&1 &

# Re-forward dashboard from host
openshell forward start --background 0.0.0.0:18789 <sandbox>
```
Find URL/token: check original installer output or gateway logs (typically `~/.openclaw/logs/`).

### "Connection refused" to model
**Cause**: Model provider not running, or wrong port configured.
**Fix**: Verify the model provider is running and port matches `openclaw.json`:
```bash
# Check vLLM is running on expected port
curl http://localhost:8001/v1/models

# For Ollama: check port 11434
curl http://localhost:11434/api/tags
```

### OpenClaw says no model available
**Cause**: Model provider not configured or model not loaded.
**Fix**: Verify `openclaw.json` has correct model configuration. For local vLLM, ensure model is loaded and serving. Note: `openclaw.json` is read-only (Landlock) — changes require sandbox rebuild.

### Config changes not applied
**Cause**: Gateway not reloaded after config change.
**Fix**: Restart the gateway process, or rebuild sandbox if `openclaw.json` changed (Landlock read-only).

### Out of memory (OOM)
**Cause**: Model too large for available GPU memory, or other GPU workloads competing.
**Fix**:
```bash
# Check GPU usage
nvidia-smi

# Free page cache (temporary relief)
sudo sh -c 'sync; echo 3 > /proc/sys/vm/drop_caches'
```
Reduce context window (16384 tokens), choose smaller model, or lower `gpu-mem` utilization for vLLM.

### Context overflow
**Fix**: Use `/compact` to reclaim 80-90% of context window.

### Model switch causes crash
**Cause**: Current context exceeds new model's context limit.
**Fix**: Run `/compact` before switching to a model with a smaller context window.

### Install script fails or dependencies missing
**Cause**: Missing system packages on Linux.
**Fix**: Install `curl` and required build tools. Check OpenClaw documentation for current requirements.

## Network-Specific Issues

### Policy applied but traffic still blocked
**Cause**: Used raw `openshell policy set` which lacks the `--wait` flag and merge logic. See SKILL.md rule #1 and `references/network-policies.md` for details.
**Fix**: Use the preset pipeline:
```bash
nemoclaw <sandbox> policy-add  # interactive
# or programmatically:
cd ~/.nemoclaw/source && node -e "require('./bin/lib/policies.js').applyPreset('<sandbox>', '<preset>')"
```

### Non-443 port traffic blocked
**Cause**: OpenShell proxy tunnels HTTPS on port 443. Non-443 external endpoints need TLS termination.
**Fix**: For internal HTTP services (e.g., Home Assistant on port 8123), use `access: full` policy entries — see `references/network-policies.md` "Accessing Internal HTTP Services". For external HTTP services, a host-side reverse proxy (Caddy/nginx) can provide TLS termination at port 443. **Do NOT run Caddy inside the sandbox** — see SKILL.md rule about no proxies in sandbox.

### Proxy returns 403
**Cause**: Destination not in active policy. Proxy at `10.200.0.1:3128` blocks unlisted traffic.
**Fix**: Add endpoint to policy via preset system.

## Real Error Patterns (from `openshell logs`)

### Dashboard connection refused
```
[sandbox] [WARN] [openshell_sandbox::ssh] direct-tcpip: failed to connect addr=127.0.0.1:18789 error=Connection refused (os error 111)
```
**Cause**: OpenClaw gateway not running inside sandbox. **Fix**: Start gateway or re-forward port.

### SSH handshake accepted (normal)
```
[sandbox] [INFO] [openshell_sandbox::ssh] SSH handshake accepted peer=10.42.0.61:56036
```
This is normal — shows SSH proxy working correctly.

### Network policy denial (L7)
```
[sandbox] [WARN] l7_decision=deny method=POST path=/api/v1/resource host=api.example.com
```
**Cause**: Network policy blocks this method/path. **Fix**: Update policy or approve in TUI.

### `openshell sandbox download/upload` wrapper directory bug
```
tar: SOUL.md: Cannot stat: No such file or directory
```
Then the target path becomes a **directory** instead of a file. Both `download` and `upload` are affected. **Fix**: Always use SSH for file transfer.

## Diagnostic Commands

Quick diagnostic workflow when something isn't working:

```bash
# 1. Check sandbox status
nemoclaw <name> status

# 2. Stream logs for errors
nemoclaw <name> logs --follow

# 3. Monitor network activity
openshell term

# 4. Test inference from inside sandbox
nemoclaw <name> connect
openclaw agent --agent main --local -m "Test inference" --session-id debug

# 5. Check underlying sandbox state
openshell sandbox list
```

## Getting Help

- [NemoClaw Discord](https://discord.gg/XFpfPv9Uvx)
- [GitHub Issues](https://github.com/NVIDIA/NemoClaw/issues/new)
- [GitHub Discussions](https://github.com/NVIDIA/NemoClaw/discussions)
