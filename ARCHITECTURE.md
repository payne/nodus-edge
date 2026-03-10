<!-- RELEASE: nodusrf/nodus-edge ARCHITECTURE.md -->
# NodusNet Edge — Architecture Overview

How the NodusNet edge node update and deployment system works under the hood.

## Design Principles

1. **Pull-based, never push** — Your edge node reaches out to check for updates. We never initiate inbound connections to your machine. No ports are opened, no tunnels are required.
2. **Canary before swap** — New images are validated in throwaway containers before touching your running stack. If the canary fails, nothing changes.
3. **Automatic rollback** — If a post-update health check fails, the system reverts to your previous working images automatically.
4. **Zero-downtime pulls** — New images download in the background while your existing containers keep running. The swap only happens after the download completes and the canary passes.

## System Overview

```
┌─────────────────────────────────────────────────────────┐
│  Your Machine                                           │
│                                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │  nodus-edge  │  │   whisper    │  │   updater    │  │
│  │  (scanner +  │  │  (speech-to- │  │  (checks for │  │
│  │  dashboard)  │  │   text, CPU) │  │  updates)    │  │
│  └──────┬───────┘  └──────────────┘  └──────┬───────┘  │
│         │                                    │          │
│         │ outbound only                      │ outbound │
└─────────┼────────────────────────────────────┼──────────┘
          │                                    │
          ▼                                    ▼
   ┌─────────────┐                    ┌──────────────┐
   │  NodusNet   │                    │   Container  │
   │  Central    │                    │   Registry   │
   │  (segments, │                    │   (images)   │
   │  heartbeats)│                    └──────────────┘
   └─────────────┘
```

## OTA Update Pipeline

The updater runs every 5 minutes via a systemd timer (or cron fallback). Here's what happens on each cycle:

### Step 1: Manifest Check

The updater fetches a version manifest from the NodusNet API — a small JSON file declaring the current image tags, checksums, and component versions.

```json
{
  "images": {
    "nodus-edge":  { "tag": "latest", "digest": "sha256:abc..." },
    "whisper-cpu": { "tag": "latest", "digest": "sha256:def..." }
  },
  "updater":  { "version": "1", "sha256": "..." },
  "compose":  { "version": "3", "sha256": "..." }
}
```

The updater compares manifest tags against its local state file (`.updater-state.json`). If everything matches, it exits — no work to do. A typical check takes <1 second and uses negligible bandwidth.

### Step 2: Self-Update

If the manifest declares a newer updater version, the updater downloads its own replacement, verifies the SHA-256 checksum, replaces itself, and re-executes. This ensures the update mechanism itself can be patched without requiring user intervention.

### Step 3: Compose File Update

If the manifest declares a newer docker-compose version, the updater downloads it with SHA-256 verification. The current compose file is backed up for rollback.

### Step 4: Image Pull (Background)

New images are pulled with `docker compose pull`. During this entire download — which can take minutes on slow connections — your existing containers remain running and serving. No interruption occurs until the canary passes.

### Step 5: Canary Pre-Flight

This is the safety gate. Before touching your production containers, the updater spins up **throwaway canary containers** from the new images on alternate ports:

```
Production (untouched)          Canary (temporary)
─────────────────────          ────────────────────
nodus-edge    :8082     →      canary-recept  :18082
whisper       :8000     →      canary-whisper :18000
```

The canary containers run without hardware access (no USB passthrough) and with `DRY_RUN=true`. The updater waits up to 90 seconds for both canaries to respond to health checks.

**If either canary fails** — crashes, times out, or never becomes healthy — the update is aborted. Your production containers are untouched. The failure is reported back to NodusNet for investigation.

**If both canaries pass** — the new images are verified bootable and healthy. Canary containers are torn down, and the update proceeds to the swap.

### Step 6: Production Swap

With canaries passing, the updater runs `docker compose up -d` to replace the production containers with the verified images. This is a standard Docker Compose rolling restart — typically 2-5 seconds of downtime.

### Step 7: Post-Swap Health Check

After the swap, the updater monitors the new production containers for up to 120 seconds, waiting for the health check to report healthy.

### Step 8: Rollback or Commit

**If healthy** — the updater writes the new tags to its state file and reports success. The backup compose file is cleaned up.

**If unhealthy** — the updater rolls back:
1. Restores the backed-up compose file (if it was updated)
2. Re-deploys the previous image tags
3. Reports the failure

The rollback restores the exact previous state. Your node recovers automatically.

## What Runs on Your Machine

The edge node consists of three containers:

| Container | Purpose | Resource Usage |
|-----------|---------|----------------|
| `nodus-edge` | RTL-SDR scanning, audio capture, transcription dispatch, local dashboard | ~200 MB RAM, low CPU (spikes during audio processing) |
| `whisper` | Speech-to-text (runs locally on CPU) | ~500 MB RAM (model-dependent), CPU-intensive during transcription |
| `updater` | Checks for updates every 5 min | Negligible (runs for <1 second, then sleeps) |

All containers run from a single `docker-compose.yml` in `~/nodusnet/`. Your configuration (`.env`) and captured data (`data/`) are stored on the host filesystem via Docker volume mounts — they persist across updates.

### Network Behavior

- **Outbound only** — all connections are initiated by your node
- **No listening ports exposed** to the internet (the dashboard binds to `localhost:8073`)
- **No inbound SSH, no tunnels, no remote access** in the default configuration
- Traffic: audio segments to NodusNet central, health heartbeats, and periodic manifest checks

### Data Stored Locally

```
~/nodusnet/
├── .env                    # Your configuration (frequencies, callsign, API key)
├── docker-compose.yml      # Container definitions
├── repeaters.json          # Local repeater database (generated at setup)
├── data/                   # Captured audio + transcription output
│   ├── fm_capture/         # Raw audio segments (auto-cleaned)
│   └── output/             # Processed JSON segments
├── .updater-state.json     # Current image versions
└── .updater.log            # Update history (last 500 lines, auto-trimmed)
```

## Security Considerations

- **No root required** — Docker handles device access via `--privileged` for the USB dongle only
- **API key authentication** — all communication with NodusNet central uses a device-specific bearer token
- **SHA-256 verification** — updater scripts and compose files are checksum-verified before being applied
- **No arbitrary code execution** — the updater only pulls Docker images and restarts containers. It does not download or execute scripts beyond its own self-update (which is checksum-verified)
- **Canary isolation** — canary containers run without hardware access and are destroyed after validation
- **Audit trail** — update history is logged locally in `.updater.log` and reported to NodusNet central

## Opting Out

The OTA updater is entirely optional. To disable it:

```bash
# Disable the systemd timer
systemctl --user disable --now nodusnet-updater.timer

# Or remove the cron entry
crontab -e  # delete the nodusnet-updater line
```

You can still update manually at any time:

```bash
cd ~/nodusnet
docker compose pull
docker compose up -d
```

Or trigger a single update check:

```bash
~/nodusnet/nodusnet-updater.sh          # normal check
~/nodusnet/nodusnet-updater.sh --force  # pull even if tags match
```
