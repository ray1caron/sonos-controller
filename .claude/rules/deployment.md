---
description: Deployment, Docker, and Pi operations rules
globs: ["Dockerfile", "docker-compose*.yml", "deploy/**", ".github/workflows/**"]
---

# Deployment and Pi rules

These rules apply when working on Docker, deployment, or Pi-side configuration.

## Container

- **Base image:** Python 3.12-slim. Pinned by digest in production builds.
- **Multi-arch:** Build with `docker buildx` for both `linux/amd64` (workstation dev) and `linux/arm64` (Pi production). Single image manifest.
- **Network mode:** `host`. Required for UPnP/SSDP discovery — bridge mode silently breaks discovery.
- **Restart policy:** `unless-stopped`. Always.
- **Healthcheck:** `curl -f http://localhost:8080/api/healthz` every 30 seconds, 5-second timeout, 3 retries before unhealthy. Container marked unhealthy → Docker restarts it under `unless-stopped`.

## Volumes

- Only `/data` is writeable from the container's perspective. Mount the host's `/data/sonos-controller` to `/data` in the container.
- `/data` contains: `token` (the bearer token), `library.db` (SQLite cache), `logs/` (rotated log files), `config.toml` (user-tunable settings).
- The container should fail loudly if `/data` is read-only or unwritable.
- The container must NOT write outside `/data`. If a logging library wants to write to `/var/log` or similar, redirect it to `/data/logs/`.

## Environment variables

| Variable | Purpose | Default |
|---|---|---|
| `AUTH_TOKEN_FILE` | Path to the bearer token file | `/data/token` |
| `LOG_LEVEL` | Logging level | `info` |
| `LIBRARY_REFRESH_HOURS` | Background reindex interval | `6` |
| `WATCHDOG_STUCK_TIMEOUT_SECONDS` | Discovery watchdog threshold | `60` |

Add new env vars only when a value genuinely needs to be runtime-configurable. Don't pollute the env namespace with every internal constant.

## Pi OS

- **Distro:** Raspberry Pi OS Lite, 64-bit
- **Root filesystem:** Read-only via `overlayroot`. Only `/data` is a tmpfs/persistent writeable mount.
- **Updates:** OS package updates require `sudo overlayroot-chroot` to temporarily remount. Plan to do this monthly, not weekly.
- **SSH:** Public-key only, fail2ban running, port 22 (no need to obscure on a LAN-only host).
- **mDNS:** `sonos-controller.local` resolves via Avahi. The pinned IP is the fallback if mDNS breaks.

## Deploy script pattern

A typical deploy from the workstation:
```bash
# 1. Build
docker buildx build --platform linux/arm64 -t sonos-controller:latest --load .

# 2. Transfer
docker save sonos-controller:latest | ssh sonos-controller "docker load"

# 3. Bring up
ssh sonos-controller "cd ~/sonos-controller && docker compose up -d"

# 4. Verify
ssh sonos-controller "docker compose logs --tail=50 sonos-controller"
TOKEN=$(ssh sonos-controller "sudo cat /data/sonos-controller/token")
curl -H "Authorization: Bearer $TOKEN" http://sonos-controller.local:8080/api/healthz | jq
```

The four steps live in `deploy/ship.sh`. **Don't** create alternative deploy paths. One script, one flow.

## Pre-flight before deploying

- `pytest tests/unit/` passes
- `pytest tests/smoke/` passes against the workstation container
- `docker buildx build` produces both architectures without errors
- The new image was tested on the workstation for at least one full smoke-suite run

If any of these fail, abort the deploy. The Pi is running production for a household; an untested deploy is worse than no deploy.

## Recovery runbook

- **Pi unresponsive but switch port shows link:** Check `ssh sonos-controller`. If timeout, the OS is hung. Power-cycle the switch port (or rely on PoE Auto Recovery if configured).
- **Pi cycles power repeatedly:** SD card may be failing. Swap to the spare card prepared in Phase 0; restore `/data` from the QNAP backup.
- **Container restart loop:** `ssh sonos-controller "docker compose logs --tail=200 sonos-controller"` to see why. Usually a SoCo discovery exception or a bad config in `/data/config.toml`.
- **All speakers offline in the API but visible in the official Sonos app:** Network change. The watchdog should catch this within 60 seconds and force a container restart. If the container is stuck, restart it manually.

## Things to NEVER do in deployment

- **NEVER** push images to a public registry. The image contains nothing secret, but registry hygiene is a habit worth keeping.
- **NEVER** ssh in to the Pi to make a one-off code change. If something needs fixing, fix it on the workstation, build, redeploy. The Pi is immutable from the developer's perspective.
- **NEVER** disable the healthcheck or change `restart: unless-stopped` to `restart: no`. If you're tempted to, the underlying problem is what needs fixing.
- **NEVER** add a `cron` task on the Pi that writes outside `/data`. The read-only root will silently fail, leaving you confused later.
