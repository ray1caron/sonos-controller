# Sonos Local Library Controller — Backend

This repo is a fork of [jasonacox/TinySonos](https://github.com/jasonacox/TinySonos) (MIT licensed) that becomes the backend for a self-hosted Sonos controller. The frontend lives in a separate repo (`sonos-controller-pwa`). The full specification is in `docs/PRD.md`. The build journey is in `docs/BuildGuide.md`.

This CLAUDE.md is the **operating manual** for working in this repo. The PRD is the spec; this file is how Claude Code should behave while implementing it.

## What this project is

- A Python backend that controls Sonos speakers via SoCo, exposing a REST + SSE API for an iPhone PWA frontend.
- Runs in Docker on a dedicated Raspberry Pi 4 (production) or on the developer's Ubuntu workstation (dev/test).
- The Pi is on the same VLAN as the speakers, the QNAP NAS, and the iPhone. UPnP discovery does not cross VLANs in this setup.
- Audio data **never** flows through the Pi. Speakers stream directly from the NAS via `x-file-cifs://` URIs.

## Architectural principles (in priority order)

1. **Speed is the feature.** Volume changes feel instantaneous. Library searches return in <500ms. Anything that compromises responsiveness is a bug, not a tradeoff.
2. **NAS-first.** The local music library is the headline experience. Streaming services are out of scope for v1.
3. **Minimum taps.** Common actions take one tap; nothing common takes more than three.
4. **Reliability through simplicity.** The Pi runs only this service. The container restarts unless-stopped. The OS root is read-only. SD-card corruption is not a failure mode we accept.

## Build strategy: this is a fork, not a from-scratch build

We inherit the SoCo wrapper, discovery, transport, volume, SSE stream, multi-room grouping, queue primitives, Docker host networking, and M3U parsing from TinySonos. We add bearer-token auth, the SQLite library cache, `x-file-cifs://` URI rewriting, the watchdog, and a new iPhone-first PWA frontend. We remove the legacy HTTP file server, the Plex export tools, and the original web UI.

Section 4.2 of `docs/PRD.md` is the authoritative reuse-vs-change boundary. **Before adding any new module, check whether something equivalent already exists in the upstream code.** Don't reinvent.

For general-purpose fixes that aren't project-specific (SoCo wrapper improvements, Docker fixes, performance work), draft an upstream PR to jasonacox/TinySonos as a courtesy. Project-specific changes (the new frontend, the SQLite cache, bearer-token auth, x-file-cifs rewriting) stay in the fork.

## Tech stack

- **Language:** Python 3.12+
- **Web framework:** FastAPI (or whatever TinySonos uses — check before changing)
- **Sonos control:** SoCo (pinned in `pyproject.toml` — do not auto-bump)
- **Library cache:** SQLite (single file on the writeable `/data` volume)
- **Container:** Docker with `network_mode: host`, multi-arch (linux/amd64 dev, linux/arm64 prod)
- **Auth:** Bearer token in the `Authorization` header on every `/api/*` endpoint

## Conventions

### API shape

- All endpoints are prefixed `/api/`. The canonical shape is in PRD section 10.
- Use RESTful paths: `/api/rooms/{id}/play`, not `/play?room=X`.
- During the migration from TinySonos's original paths, keep legacy aliases working but don't add new endpoints under the old shape.
- JSON in, JSON out. Empty success responses are HTTP 204.

### Code

- Run `ruff` and `black` on every change. Never commit unformatted code.
- Type hints on all new functions. Existing untyped TinySonos code can stay untyped during Phase 1; add types as you touch it in later phases.
- Prefer SoCo's high-level methods (`play_uri`, `add_to_queue`, `join`) over raw SOAP. If SoCo doesn't expose what you need, ask whether the goal is in scope before reaching for `soap_client`.

### Logging

- Structured JSON to stdout, consumed by `docker logs`.
- Levels: `INFO` for normal operation, `WARNING` for recoverable issues, `ERROR` for failures, `DEBUG` only behind a flag.
- Never log the bearer token, even at DEBUG. Never log NAS credentials. Redact path segments that might contain user identifiers.

### Tests

- `pytest` with `tests/smoke/` for integration tests against a real speaker, `tests/unit/` for pure logic.
- Smoke tests are slow and require a real speaker on the LAN. Tag them `@pytest.mark.smoke` and exclude from the default run.
- After every meaningful change, run the smoke suite. A regression here is a stop-everything bug.

### Commits and branches

- One atomic commit per logical change. GSD's `/gsd-execute-phase` produces one commit per task — that's the right granularity.
- Conventional commit prefixes: `feat:`, `fix:`, `refactor:`, `docs:`, `test:`, `chore:`. Plus `from-upstream:` when cherry-picking from TinySonos.
- Branch names follow GSD's `phase-N-task-M` pattern. PRs from `/gsd-ship` go to `main`.

## Security and secrets

The agent runs with `--dangerously-skip-permissions` for workflow speed. Defence-in-depth lives in the deny-list in `.claude/settings.json`. Specifically:

- **NEVER** read `.env`, `.env.*`, `**/secrets/*`, `**/*credential*`, `**/*.pem`, `**/*.key`, or `/data/token`. The deny-list enforces this; this rule reinforces the intent.
- **NEVER** commit secrets, even briefly. The bearer token is generated at first run on the Pi and lives in `/data/token`, which is in `.gitignore`.
- **NEVER** log a request's `Authorization` header or its decoded contents.
- **NEVER** add a third-party telemetry, analytics, or crash-reporting library. The system is LAN-only and stays that way.

## Pi access

The Pi is `sonos-controller.local` (mDNS) or its pinned IP via UDM Pro DHCP reservation (typically `192.168.1.50`). Access is via SSH using the dedicated key `~/.ssh/sonos_controller_key`. The `~/.ssh/config` alias `sonos-controller` lets you write `ssh sonos-controller`.

Standard deploy from the workstation:
```
docker buildx build --platform linux/arm64 -t sonos-controller:latest --load .
docker save sonos-controller:latest | ssh sonos-controller "docker load"
ssh sonos-controller "cd ~/sonos-controller && docker compose up -d"
```

For OS package updates (read-only root requires a temporary remount):
```
ssh sonos-controller "sudo overlayroot-chroot"
# inside the chroot:
apt update && apt upgrade -y
exit
ssh sonos-controller "sudo reboot"
```

Do not edit files on the Pi directly. All source-of-truth changes happen on the workstation, get committed, get built into a new image, get deployed. The Pi has no editor more capable than `vi` for a reason.

## GSD workflow

This project uses GSD v1 for spec-driven development. The phase structure mirrors PRD section 12. Each phase = one GSD cycle:

1. `/gsd-discuss-phase N` — capture preferences before planning
2. `/gsd-plan-phase N` — generate atomic task plans
3. `/gsd-execute-phase N` — run plans in waves, atomic commit per task
4. `/gsd-verify-work N` — user-acceptance pass
5. `/gsd-ship N` — clean PR branch

The `.planning/` directory holds the durable planning state (PROJECT.md, REQUIREMENTS.md, ROADMAP.md, STATE.md, plus per-phase plans). It is committed to git.

For ad-hoc work outside the phase structure, use `/gsd-quick`. Don't try to fit one-off bugfixes into a full phase ceremony.

## Things to NEVER do

- **NEVER** stream audio through the Pi. URIs sent to speakers must be `x-file-cifs://NAS/Music/...`, never `http://localhost:...`.
- **NEVER** add streaming-service support (Spotify, Apple Music, Tidal, etc.). Out of scope for v1 per PRD section 2.3.
- **NEVER** add functionality that requires the Pi to reach the public internet. The system is LAN-only.
- **NEVER** persist state to the Pi's root filesystem. Only `/data` is writeable.
- **NEVER** reach across VLAN boundaries. If a request needs a thing on another VLAN, the answer is "put both on the same VLAN," not "configure multicast routing."
- **NEVER** suppress a SoCo exception silently. If SoCo says discovery failed, surface it.
- **NEVER** commit changes to `.planning/STATE.md` outside of GSD's normal flow. The agent owns that file.
- **NEVER** modify the LICENSE file or remove the upstream-attribution paragraph from the README. The MIT terms inherited from TinySonos are non-negotiable.

## References

- Full specification: `docs/PRD.md` (section 4.2 = fork strategy, section 10 = API spec, section 11 = data model, section 12 = build plan)
- Build journey: `docs/BuildGuide.md`
- Upstream project: https://github.com/jasonacox/TinySonos
- Path-scoped rules: `.claude/rules/`

## Path-scoped rules (loaded automatically when working in matching paths)

- `.claude/rules/backend-python.md` — Python-specific conventions for `src/**`, `tests/**`
- `.claude/rules/deployment.md` — deployment, Docker, and Pi-specific rules for `Dockerfile`, `docker-compose*.yml`, `deploy/**`

<!-- GSD:project-start source:PROJECT.md -->
## Project

Project not yet initialized. Run /gsd-new-project to set up.
<!-- GSD:project-end -->

<!-- GSD:stack-start source:STACK.md -->
## Technology Stack

Technology stack not yet documented. Will populate after codebase mapping or first phase.
<!-- GSD:stack-end -->

<!-- GSD:conventions-start source:CONVENTIONS.md -->
## Conventions

Conventions not yet established. Will populate as patterns emerge during development.
<!-- GSD:conventions-end -->

<!-- GSD:architecture-start source:ARCHITECTURE.md -->
## Architecture

Architecture not yet mapped. Follow existing patterns found in the codebase.
<!-- GSD:architecture-end -->

<!-- GSD:skills-start source:skills/ -->
## Project Skills

No project skills found. Add skills to any of: `.claude/skills/`, `.agents/skills/`, `.cursor/skills/`, `.github/skills/`, or `.codex/skills/` with a `SKILL.md` index file.
<!-- GSD:skills-end -->

<!-- GSD:workflow-start source:GSD defaults -->
## GSD Workflow Enforcement

Before using Edit, Write, or other file-changing tools, start work through a GSD command so planning artifacts and execution context stay in sync.

Use these entry points:
- `/gsd-quick` for small fixes, doc updates, and ad-hoc tasks
- `/gsd-debug` for investigation and bug fixing
- `/gsd-execute-phase` for planned phase work

Do not make direct repo edits outside a GSD workflow unless the user explicitly asks to bypass it.
<!-- GSD:workflow-end -->

<!-- GSD:profile-start -->
## Developer Profile

> Profile not yet configured. Run `/gsd-profile-user` to generate your developer profile.
> This section is managed by `generate-claude-profile` -- do not edit manually.
<!-- GSD:profile-end -->
