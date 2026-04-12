# NemoClaw Workspace Files & Backup/Restore

## Persistence Model

**WARNING**: Workspace files are on the pod's **overlay filesystem**, NOT a Persistent Volume. They are **wiped on every pod restart** (reboot, crash, gateway restart). This is a known issue (NVIDIA/NemoClaw#486).

- **Wiped on pod restart**: Every restart resets the overlay — workspace, skills, cron, sessions, memory notes all lost
- **Wiped on `destroy`**: `nemoclaw <name> destroy` also deletes everything
- **Not auto-recovered**: Without a watchdog, workspace must be manually restored after every reboot

**Automated backup/restore is essential.** See "Automated Recovery (Watchdog Pattern)" section below.

**Back up before:**
- Any server reboot or maintenance
- Running `nemoclaw <name> destroy`
- Major version upgrades
- Periodically after significant customization

## Workspace Files

### Path: Overlay Filesystem

NemoClaw uses an **overlay** for the `.openclaw` directory:

| Path | Access | Role |
|------|--------|------|
| `/sandbox/.openclaw/` | **Read-only** (Landlock) | Base layer — contains `openclaw.json`, default config |
| `/sandbox/.openclaw-data/` | **Read-write** | Overlay layer — runtime writes go here |

Both directories have identical structure (`workspace/`, `skills/`, `agents/`, `memory/`, etc.). When the agent writes a file, it goes to `.openclaw-data/`. When reading, `.openclaw-data` is checked first, then falls through to `.openclaw`.

**For backup/restore operations, always use `.openclaw-data/`** — that's where user-created content lives.

### Workspace Files

Located at `/sandbox/.openclaw-data/workspace/`:

| File | Purpose |
|------|---------|
| `SOUL.md` | Core personality, tone, behavioral rules |
| `USER.md` | Preferences, context, facts about user |
| `IDENTITY.md` | Agent name, creature type, emoji |
| `AGENTS.md` | Multi-agent coordination, safety guidelines |
| `MEMORY.md` | Curated long-term memory (required — UI shows "missing" without it) |
| `TOOLS.md` | Available tools and capabilities |
| `HEARTBEAT.md` | Heartbeat/status configuration |
| `memory/` | Daily notes (`YYYY-MM-DD.md`) — session continuity |
| `memory/.dreams/` | Short-term recall store (`short-term-recall.json`) — used by dreaming/REM |

### Editing Workspace Files

Two approaches:
1. **Agent-driven**: Ask the agent to update its own configuration during a chat session
2. **Manual**: Use `openshell sandbox connect` for terminal access, or transfer files via SSH

### SSH Access

All SSH examples use `ssh sandbox` as shorthand. Set up an alias first — see the main SKILL.md SSH section. Inline usage without an alias:
```bash
ssh -o ProxyCommand="openshell ssh-proxy --gateway-name <gateway> --name <sandbox>" sandbox "<command>"
```

## Additional Data to Back Up

| Path | Content | Notes |
|------|---------|-------|
| `/sandbox/.openclaw-data/skills/` | Custom skills (SKILL.md files) | User-created capabilities |
| `/sandbox/.openclaw-data/cron-jobs.json` | Cron job definitions | Must re-add via `openclaw cron add` on restore |
| Network policy | Applied via host-side commands | Export from host |
| `openclaw.json` | Agent configuration | **Read-only** (Landlock) — can't modify in place |
| `credentials.json` | Agent credentials | Sensitive — handle with care |

## Backup Methods

### Official Method (openshell download) — NOT RECOMMENDED

The official docs recommend `openshell sandbox download`, but it has a known bug: downloaded files are wrapped in a directory (e.g., `SOUL.md` becomes `SOUL.md/SOUL.md`). This affects both `upload` AND `download`.

**Use SSH instead** (see below).

### SSH Method (Recommended for Full Backup)

All SSH examples below use `ssh sandbox` as shorthand. This requires an SSH alias — see SKILL.md "SSH Access to Sandbox" section for setup.

SSH is more reliable and avoids the `openshell upload` bug (see Critical Pitfalls below). Use SSH for comprehensive backups including skills, cron, and policy:

```bash
SANDBOX=my-assistant
BACKUP_DIR=~/.nemoclaw/backups/$(date +%Y%m%d-%H%M%S)
mkdir -p "$BACKUP_DIR"

# Workspace files
for f in SOUL.md USER.md IDENTITY.md MEMORY.md TOOLS.md AGENTS.md HEARTBEAT.md; do
  ssh sandbox "cat /sandbox/.openclaw-data/workspace/$f" > "$BACKUP_DIR/$f" 2>/dev/null
done

# Memory directory
ssh sandbox "tar czf - /sandbox/.openclaw-data/workspace/memory/" > "$BACKUP_DIR/memory.tar.gz" 2>/dev/null

# Skills
ssh sandbox "tar czf - /sandbox/.openclaw-data/skills/" > "$BACKUP_DIR/skills.tar.gz" 2>/dev/null

# Cron jobs
ssh sandbox "cat /sandbox/.openclaw-data/cron-jobs.json" > "$BACKUP_DIR/cron-jobs.json" 2>/dev/null

# Network policy (from host side)
openshell policy get --gateway nemoclaw $SANDBOX > "$BACKUP_DIR/network-policy.yaml" 2>/dev/null

# Credentials
ssh sandbox "cat /sandbox/.openclaw-data/credentials.json" > "$BACKUP_DIR/credentials.json" 2>/dev/null
```

### Script Method

NemoClaw ships a convenience script:

```bash
# Backup
./scripts/backup-workspace.sh backup my-assistant

# Restore (latest or specific timestamp)
./scripts/backup-workspace.sh restore my-assistant [timestamp]
```

### Automated Backup (Cron — Every 6 Hours)

```bash
# crontab -e
# backup-full.sh MUST use absolute paths for openshell/node (see SKILL.md rule #12)
0 */6 * * * ~/.nemoclaw/backup-full.sh <sandbox-name> <gateway-name> >> /tmp/nemoclaw-backup.log 2>&1
```

Runs at 00:00, 06:00, 12:00, 18:00. Retains 28 backups (7 days). The script should be SSH-based to avoid the upload wrapper-directory bug.

### Verification After Backup

```bash
# Check backup directory has all files
ls -la "$BACKUP_DIR/"

# Or connect directly to sandbox to inspect (use your actual workspace path)
openshell sandbox connect $SANDBOX
ls -la /sandbox/.openclaw-data/workspace/    # or /sandbox/.openclaw/workspace/
```

## Restore Methods

### Official Method (openshell upload)

The official docs suggest `openshell sandbox upload`, but **this has a known bug** — it creates wrapper directories (`file.md` becomes `file.md/file.md`).

If you must use it, verify file placement immediately after:
```bash
openshell sandbox connect $SANDBOX
ls -la /sandbox/.openclaw-data/workspace/SOUL.md  # should be a file, not directory
```

### SSH Method (Recommended)

Always use SSH for restore to avoid the wrapper directory bug:

```bash
BACKUP_DIR=~/.nemoclaw/backups/<latest>
for f in SOUL.md USER.md IDENTITY.md MEMORY.md TOOLS.md HEARTBEAT.md AGENTS.md; do
  ssh sandbox "cat > /sandbox/.openclaw-data/workspace/$f" < "$BACKUP_DIR/$f"
done
```

## Full Restore After Rebuild

After running `nemoclaw onboard` (which recreates the sandbox), follow these steps **in order**:

### Step 1: Forward Dashboard
```bash
openshell forward start --background 0.0.0.0:18789 <sandbox>
```

### Step 2: Start Gateway (if not auto-started)
```bash
ssh sandbox "nohup openclaw gateway run > /tmp/gateway.log 2>&1 &"
```

### Step 3: Approve Device Pairing
These commands run **inside the sandbox** (connect first with `nemoclaw <sandbox> connect` or `ssh sandbox`):
```bash
# Inside sandbox:
openclaw devices list --json
openclaw devices approve <requestId>
```

### Step 4: Fix File Ownership
Delete root-owned `.bashrc`/`.profile` (created during Docker build as root):
```bash
ssh sandbox "rm -f /sandbox/.bashrc /sandbox/.profile"
```

### Step 5: DNS Proxy
```bash
bash scripts/setup-dns-proxy.sh nemoclaw <sandbox>
```
Without this, Node.js `web_fetch` fails with `EAI_AGAIN`.

### Step 6: Restore Workspace via SSH
```bash
BACKUP_DIR=~/.nemoclaw/backups/<latest>
for f in SOUL.md USER.md IDENTITY.md MEMORY.md TOOLS.md HEARTBEAT.md AGENTS.md; do
  ssh sandbox "cat > /sandbox/.openclaw-data/workspace/$f" < "$BACKUP_DIR/$f"
done
```

### Step 7: Create Memory Directory
```bash
ssh sandbox "mkdir -p /sandbox/.openclaw-data/workspace/memory"
```

### Step 8: Restore Memory Files
```bash
ssh sandbox "tar xzf - -C /" < "$BACKUP_DIR/memory.tar.gz"
```

### Step 9: Restore Skills
```bash
ssh sandbox "tar xzf - -C /" < "$BACKUP_DIR/skills.tar.gz"
```

### Step 10: Apply Network Policy
```bash
openshell policy set --gateway nemoclaw <sandbox> --policy "$BACKUP_DIR/network-policy.yaml"
```

### Step 11: Remove Stale Provider
```bash
openshell provider delete --gateway nemoclaw compatible-endpoint
```

### Step 12: Re-add Cron Jobs
Add cron jobs via `openclaw cron add` (NOT by editing `cron-jobs.json` directly — the file format includes runtime state that `cron add` manages).

### Step 13: Reset Sessions
After restoring workspace files, clear all sessions so the agent re-reads the updated SOUL.md/USER.md:
```bash
ssh sandbox "rm /sandbox/.openclaw-data/agents/main/sessions/sessions.json /sandbox/.openclaw-data/agents/main/sessions/*.jsonl 2>/dev/null"
```
Without this, the agent uses cached system prompts from before the restore.

### Step 14: Start Services
```bash
cd ~/.nemoclaw/source && ./scripts/start-services.sh
```

## Critical Pitfalls

| Pitfall | Impact | How to Avoid |
|---------|--------|--------------|
| `openshell sandbox upload/download` wrapper dirs | `file.md` becomes `file.md/file.md` directory (both upload AND download) | Use SSH `cat >` / `cat` or tar for all file transfers |
| `.bashrc`/`.profile` root ownership | Sandbox user can't write → nemoclaw-start fails | Delete root-owned copies after rebuild |
| Missing DNS proxy | `web_fetch` fails with `EAI_AGAIN` | Run `setup-dns-proxy.sh` after every rebuild |
| `openclaw.json` read-only | Landlock prevents modification at runtime | Changes require full sandbox rebuild |
| Gateway token changes on rebuild | External systems using token break | Update any token references after rebuild |
| SSH handshake secret regeneration | SSH breaks on container restart (NemoClaw#888) | Hardcode secret in helm template |
| Editing `cron-jobs.json` directly | Runtime state corrupted | Always use `openclaw cron add` |
| Forgetting memory directory | Daily notes lost | Create `memory/` dir explicitly, then restore tar |
| Not verifying after restore | Silent failures go unnoticed | Connect to sandbox and `ls` workspace after restore |
| BOOTSTRAP.md in backups | Agent re-introduces itself after every restore + session reset | Delete BOOTSTRAP.md from sandbox AND backups after initial setup |
| Not resetting sessions after restore | Agent ignores restored SOUL.md/USER.md (stale `systemSent: true`) | Always clear sessions after workspace restore |
| Cron backup uses bare `openshell`/`node` | Silently fails — binaries not in cron PATH (nvm) | Use absolute paths: `$HOME/.local/bin/openshell`, `$HOME/.nvm/versions/node/vXX/bin/node` |

## Automated Recovery (Watchdog Pattern)

Since workspace is wiped on every pod restart, automated recovery is essential for always-on deployments. This pattern uses a cron watchdog that detects wipes and restores from backup.

### Components

1. **6-hourly backup** (`backup-full.sh`): SSH+tar backup of workspace, skills, cron to host disk. Maintains a `latest-good` symlink pointing to the most recent backup with valid content (≥3 .md files). Retains 28 backups (7 days).

2. **Watchdog** (cron every minute): Detects workspace wipe via sentinel file, restores from `latest-good`, resets sessions.

### Sentinel-Based Wipe Detection

Instead of checking file content (fragile), write a sentinel file after successful restore:

```bash
SANDBOX_SENTINEL="/sandbox/.openclaw-data/.nemoclaw-restored"
```

- Sentinel lives on the overlay → wiped on pod restart (exactly the behavior we need)
- Host-side marker in `/tmp/` prevents re-running restore every minute after first detection
- `/tmp/` marker clears on host reboot → triggers restore check on next boot

### Watchdog Restore Flow

```
Cron (every minute)
  → Is sandbox SSH-reachable? (if not, skip)
  → Does host marker exist? (if yes, skip — already restored this boot)
  → Does sentinel exist inside sandbox? (if yes, create host marker, skip)
  → Sentinel missing = workspace wiped → RESTORE:
      1. tar workspace from latest-good → SSH → sandbox
      2. tar skills from latest-good → SSH → sandbox
      3. Restore cron-jobs.json file
      4. Clear sessions (rm sessions/*.json *.jsonl)
      5. Create memory/ directory
      6. Write sentinel file
      7. Create host marker
  → After gateway is up:
      8. Re-register cron jobs via `openclaw cron add` (file alone is NOT enough — gateway doesn't load from file)
```

**CRITICAL: Cron job re-registration.** Restoring `cron-jobs.json` to the filesystem is not sufficient. The OpenClaw gateway maintains its own in-memory cron state and does NOT read from the file on startup. Cron jobs must be re-added via `openclaw cron add` after every restore. The watchdog (step 0.7) handles this automatically by parsing `cron-jobs.json` from backup and calling `openclaw cron add` for each job.

### Key Design Decisions

- **tar over SSH** for bulk restore (~12s total vs ~30s for per-file SSH)
- **latest-good symlink** only updated when backup has ≥3 workspace .md files (protects against backing up a wiped sandbox)
- **Session reset included** in restore flow — agent always gets fresh system prompt from restored workspace
- **BOOTSTRAP.md excluded** from backups — prevents re-introduction behavior
- **Backward compatible**: Watchdog detects whether latest-good points to full backup dir (new) or workspace subdir (old)

## Disaster Recovery

### Backup directory fills up

Old backups accumulate in `~/.nemoclaw/backups/`. Each backup is ~1-5 MB (text files + YAML). At 32 backups × 5 MB = ~160 MB, this is rarely an issue, but for long-running systems:

```bash
# Keep only the 10 most recent backups (excluding latest-good symlink)
cd ~/.nemoclaw/backups
ls -dt 2026*/ | tail -n +11 | xargs rm -rf
```

Add to crontab for automatic cleanup: `0 5 * * 0 cd ~/.nemoclaw/backups && ls -dt 2026*/ | tail -n +11 | xargs rm -rf`

### latest-good symlink is corrupted or points to empty backup

```bash
# Check what latest-good points to
ls -la ~/.nemoclaw/backups/latest-good

# If empty or corrupted, find the most recent backup with workspace files
for d in $(ls -dt ~/.nemoclaw/backups/2026*/); do
  count=$(ls "$d/workspace/"*.md 2>/dev/null | wc -l)
  echo "$d: $count .md files"
done

# Re-link to a known-good backup
ln -sfn ~/.nemoclaw/backups/20260412-120001 ~/.nemoclaw/backups/latest-good
```

### Nuclear option: full sandbox rebuild from scratch

If backups are lost and the sandbox is corrupted:

1. Export anything still readable from the sandbox: `ssh sandbox "tar cf - -C /sandbox/.openclaw-data ." > emergency-dump.tar 2>/dev/null`
2. Destroy and recreate: `nemoclaw <name> destroy && nemoclaw onboard`
3. Manually recreate SOUL.md, USER.md, IDENTITY.md, TOOLS.md from memory or version control
4. Redeploy skills from their source (e.g., git repo)
5. Re-register cron jobs
6. Clear sessions and delete BOOTSTRAP.md

**Prevention:** Keep workspace files in version control (git) separately from NemoClaw backups. Backups handle fast recovery; git handles disaster recovery.
