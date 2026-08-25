---
name: vps-disk-cleanup
description: Safe VPS disk cleanup when the disk is near full - scan space hogs, delete only caches and stale temp files, keep business data and required ML models. Trigger when the disk is full, "clean the cache", or df -h shows 85%+ usage.
version: 1.1.0
author: Hermes Agent
metadata:
  hermes:
    tags: [vps, disk, cleanup, cache, maintenance]
---

# VPS Disk Cleanup

Safe cleanup procedure for a containerized VPS (e.g. Hostinger, 48G overlay disk). Freed ~4.2G in one pass in Aug 2026; a later pass freed ~10.8G total (46G -> 36G used, 2.2G -> 13G free, 96% -> 75%).

## CRITICAL: df vs du discrepancy - the invisible base image + swapfile

When the user asks "why is it still showing X GB?" after a cleanup, the answer is usually the container base image + the host swapfile, not leftover waste:

- `df -h /` may report ~36G used while `du -x -s /` sees only ~10-14G of visible files. The gap has TWO parts:
  - **~14G container base image**: the root filesystem is an overlay (`grep " / " /proc/mounts`) of many read-only lowerdir layers (preinstalled OS, Python, Node, Chromium, system libs) + one writable upperdir (user files).
  - **~8G host-level swapfile**: `/proc/swaps` shows `/swapfile` active, but `ls /swapfile` = not found and `find /` finds nothing. The swapfile lives on the host disk and counts against the quota, but is invisible/untouchable from inside the container.
- The base layers live at `/var/lib/containerd/...` on the HOST - the path does not exist inside the container namespace. **They cannot be seen or deleted from inside.**
- **Deleting files that came from the base image frees NOTHING** - overlayfs whiteouts them but the lower-layer data stays on the host disk. Only files created in the writable upper layer (caches, app data, transcripts) actually free quota when deleted. That is exactly why cleaning /usr or system caches has zero effect.
- The base image was there from day one (disk "starts" at ~46% used). The only levers are host-side: rebuild from a leaner image (wipes everything - not realistic), or upgrade the disk plan.
- When reporting cleanup results, give the df numbers AND explain the base image so the residual usage is not mistaken for waste.

## CRITICAL: RAM vs SWAP confusion

A VPS plan may advertise "8GB" that turns out to be swap, not RAM. Reality check (`/proc/meminfo`): **MemTotal 3.8G, SwapTotal 8G** (file-based swap). Facts:

- `swapoff -a` fails with "swapoff: Not superuser" even as root when the container lacks `cap_sys_admin` (verify: `grep CapEff /proc/self/status`, decode with `capsh --decode`). Swap cannot be removed from inside.
- The swapfile is host-allocated and not present in the container filesystem - only the hosting provider / control panel can remove or shrink it.
- Removing swap entirely is risky: with limited RAM and no swap, the OOM killer will kill processes under load. Suggest shrinking to 2G (frees 6G) rather than removal.
- Writing a file inside the container DOES move the df needle (verified: 1G test file moved df 36->37G), so the writable layer is counted normally.

## Trigger

- User says disk is full / "clean the cache"
- Hosting panel warns disk limit reached
- `df -h /` shows 85%+ usage

## Procedure

### 1. Baseline

```bash
df -h /
```

### 2. Scan the known hog dirs (in parallel)

```bash
du -sh /root/.cache/* 2>/dev/null | sort -rh | head -15
du -sh /tmp /var/cache /var/log 2>/dev/null
ls /root/.cache/huggingface/hub/ 2>/dev/null
du -sh /tmp/* 2>/dev/null | sort -rh | head -12
```

Always check `/root/.cache` FIRST - it is the major hidden space hog (4-5G typical: huggingface, pip, uv, composer, puppeteer, camoufox).

### 3. Delete (safe list)

```bash
# Unused whisper models - ONLY if turbo is the one in use (verify in step 2 first)
rm -rf /root/.cache/huggingface/hub/models--Systran--faster-whisper-medium
rm -rf /root/.cache/huggingface/hub/models--Systran--faster-whisper-small
# Package caches
<venv>/bin/pip cache purge
uv cache clean 2>/dev/null || true
rm -rf /root/.cache/composer /root/.cache/electron /root/.cache/node-gyp 2>/dev/null
rm -rf /root/.npm/_npx /root/.npm/_cacache 2>/dev/null   # npx/npm caches re-download on demand
# System caches + logs
apt-get clean
rm -rf /var/lib/apt/lists/* 2>/dev/null
journalctl --vacuum-size=20M
# PM2 logs grow unbounded - truncate (a bot out.log was 301M in one case)
truncate -s 0 /root/.pm2/logs/*.log
# Stale /tmp items: old backup zips, temp videos, build dirs, node_modules
rm -rf /tmp/*.zip /tmp/node_modules /tmp/*.mp4 2>/dev/null
```

### 3b. Prune pre-update state snapshots (if using Hermes)

Some agent frameworks keep pre-update safety copies of state files (can be 1.8G each, created by the update process). They are NOT business data - old ones can be pruned via the framework's own mechanism:

```bash
cd /opt/hermes && .venv/bin/python -c "
import sys; sys.path.insert(0,'.')
from hermes_cli.backup import prune_quick_snapshots
print('pruned:', prune_quick_snapshots(keep=0))"
```

- `keep=N` keeps the N newest; `keep=0` deletes all (fine when the current state is healthy)
- Do NOT `rm -rf` the directory manually - let the framework's backup module manage it

### 3c. Second-tier targets when caches alone aren't enough

After the cache pass, these freed ~2.5G more:

```bash
# 1. Old raw session transcript archives (~700M, 9k+ files from old cron jobs)
#    SAFE to delete ONLY if searchable history lives in the main state DB
#    (verify that first - otherwise you lose history).
rm -rf /opt/data/sessions   # example: raw JSONL cron logs

# 2. Browser profile auto-downloaded components (~250M, regenerate on next browser use,
#    does NOT lose logins - those live in the profile's Default dir):
rm -rf "<profile>/CertificateRevocation" "<profile>/WasmTtsEngine"

# 3. Shrink an app's .git history (e.g. 1.5G full history) - keeps updates working:
cd /opt/myapp && git gc --aggressive --prune=now   # SLOW: 10-20+ min on 1.5G - run in background

# 4. Stale duplicate of an app install in /tmp (real install lives elsewhere):
#    verify first: grep the launcher script for the path it actually uses
rm -rf /tmp/image2-ads-studio   # example: launcher script uses /opt/data/image2-ads-studio
```

### 4. KEEP (never delete)

- Required ML models in `/root/.cache/huggingface/hub/` (e.g. faster-whisper-large-v3-turbo 1.6G) - REQUIRED for transcriptions
- Business data: state DBs, browser profile, production assets, scripts, skills. Only old snapshots, raw transcript archives (with approval), and `/opt/data/cache`-style dirs are disposable.
- Small framework caches that regenerate cheaply

### 5. Verify

```bash
df -h /
```

Report before/after numbers: used GB, free GB, % usage, and what was cleaned.

## Pitfalls

- **Whisper models re-download:** removed models have come BACK after being removed (something re-downloads them). Cleanup is recurring - always re-check `ls /root/.cache/huggingface/hub/` before assuming the disk stays clean.
- **Hosting panel shows stale usage:** the control panel can lag 5-30 min after cleanup. `df -h` is the real number immediately - say so when reporting.
- **Never touch business data blindly:** state DBs, session profiles, and production files are business data. Only explicit cache dirs are disposable.
- **Check for in-use /tmp files:** before `rm -rf /tmp/*`, confirm running processes aren't actively writing there. Deleting known stale items by name is safer than a blanket wipe.
- **Videos in /tmp are temp copies:** downloaded media lands in /tmp and is safe to delete - originals live elsewhere.
