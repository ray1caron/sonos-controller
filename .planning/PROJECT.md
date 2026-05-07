# Sonos Local Library Controller — Backend

## What This Is

A self-hosted iPhone PWA + Python backend for controlling Sonos speakers, with a primary focus on browsing and playing audio files (.mp3 and similar) hosted on a QNAP NAS. The backend is a fork of [jasonacox/TinySonos](https://github.com/jasonacox/TinySonos) (MIT) that runs in Docker on a dedicated Raspberry Pi 4, exposing a REST + SSE API to a separate iPhone-first PWA frontend (`sonos-controller-pwa` repo). The system is LAN-only by design and exists because the official Sonos app, post-2024, treats local-NAS libraries as second-class.

## Core Value

**Speed is the feature.** A volume change is audible on the speaker within 100 ms of touch. Library search returns in under 500 ms for libraries up to 50,000 tracks. Cold-launch to playing music is under 10 seconds. Common tasks — play album, group rooms, change volume, skip track — take no more than two taps. If everything else is correct but the controller feels slow, the project has failed.

## Requirements

### Validated

<!-- Inherited from upstream TinySonos. These already work end-to-end. -->

- ✓ Auto-discover Sonos speakers via SoCo (UPnP/SSDP) — TinySonos
- ✓ Transport control: play, pause, next, previous, seek, shuffle, repeat — TinySonos
- ✓ Per-speaker and per-group volume + mute — TinySonos
- ✓ Multi-room grouping (join/unjoin, party mode, coordinator selection) — TinySonos
- ✓ Server-Sent Events stream for state push (`/api/events`) — TinySonos
- ✓ Queue management primitives (add, clear, view) — TinySonos
- ✓ M3U playlist parsing — TinySonos
- ✓ Docker packaging with `network_mode: host` and `restart: unless-stopped` — TinySonos
- ✓ Single-threaded SoCo command queue (PlaybackController) — TinySonos
- ✓ Dynamic re-discovery on network topology changes — TinySonos

### Active

<!-- v1 scope. Hypotheses until each phase ships and verifies. -->

**Audio path & library:**
- [ ] Switch playback URIs from TinySonos's HTTP file server to `x-file-cifs://NAS/Music/...` so audio bypasses the Pi entirely (PRD §4.1.6, §4.2.2)
- [ ] SQLite library cache (artists, albums, tracks, genres, folders) on `/data` volume, per PRD §11 schema (PRD §4.1.5, §4.2.2)
- [ ] Indexer that walks Sonos's music-library API via SoCo and populates SQLite; manual + scheduled (6h default) refresh (F-LB-08, F-LB-09)
- [ ] `/api/library/*` endpoints (artists, albums, tracks, genres, folders) backed by SQLite (PRD §10.3)
- [ ] `/api/search` aggregated search (artists / albums / tracks in one response, <500 ms for 50k tracks) (F-SR-01..04)

**Auth & API hardening:**
- [ ] Bearer-token auth middleware on all `/api/*` endpoints; token generated at first run, stored in `/data/token` (PRD §4.2.2, §6.10 F-OP-03)
- [ ] Normalise endpoint shapes to canonical `/api/rooms`, `/api/groups`, `/api/library`, `/api/playlists` (PRD §10) with TinySonos legacy paths kept as aliases for one phase
- [ ] Structured `/api/healthz` returning discovery state, library cache age, SSE subscriber count (F-OP-02)
- [ ] Watchdog: FastAPI process self-exits if discovery returns zero rooms for >60s when it had rooms previously (PRD §7.2)
- [ ] JSON-on-stdout structured logging with `/data/logs/` rotation (F-OP-04)

**iPhone-first PWA frontend** (separate repo `sonos-controller-pwa`):
- [ ] Four primary screens: Now Playing, Library Browse, Rooms, Search/Queue (PRD §9 wireframes)
- [ ] Alphabet jump-bar on long alphabetised lists (F-LB-03 — signature feature, missing from every existing controller)
- [ ] Touch-to-Play default: tap = play immediately, long-press = action sheet (F-LB-04, F-LB-05, F-LB-06)
- [ ] Optimistic-UI volume sliders with 20 req/s throttling and SSE confirmation (F-VL-01, F-VL-04, F-VL-05)
- [ ] Queue with drag-to-reorder + swipe-to-delete + always-visible now-playing row (F-QM-01..04)
- [ ] PWA manifest, service worker (cache shell), iPhone home-screen install (PRD §4.1.1)

**Deployment & operations:**
- [ ] Multi-arch Docker image (linux/amd64 + linux/arm64) via buildx (PRD §4.2.3)
- [ ] Read-only root filesystem on Pi (overlayroot), only `/data` writeable (PRD §4.1.4, §7.2)
- [ ] Pi 4 production target reachable as `sonos-controller.local` + DHCP-pinned IP (PRD §5.5.2)
- [ ] Container survives forced kill and recovers within 60 s (PRD §7.2)

### Out of Scope

<!-- Explicit v1 exclusions. Re-adding requires explicit discussion. -->

- **Streaming services** (Spotify, Apple Music, Tidal, Amazon, internet radio, TuneIn) — explicit v1 non-goal per PRD §2.3; NAS-first is the headline experience
- **Apple Watch / lock-screen / Control Center / CarPlay / Siri** — PWA platform limitation; accepted trade-off (PRD §2.3, §13.3 R2)
- **Alarms and sleep timers** — deferred to v2 (PRD §2.3)
- **Speaker setup, firmware updates, stereo-pair configuration** — handled by official Sonos app (PRD §2.3)
- **Public internet access** — system is LAN-only; remote access layered on existing UDM Pro WireGuard if ever needed (PRD §2.3, §7.3)
- **Multi-user accounts** — single shared bearer token; PRD §13.2 Q5 marked re-visit if needed (PRD §13.2)
- **TinySonos's HTTP file server (port 54000)** — removed; replaced by `x-file-cifs://` (PRD §4.2.4)
- **TinySonos's Plex export tools (`tools/`)** — outside this product's scope; not in deployed image (PRD §4.2.4)
- **TinySonos's original web UI (`web/`)** — replaced by the new PWA (PRD §4.2.4)
- **CI/CD via GitHub Actions** — PRD §5.7 explicitly defers; manual smoke suite + 7-day cutover trial substitute for v1
- **Public Docker registry pushes** — single-developer deploy via `docker save | ssh ... docker load` (PRD §5.5.3, deployment.md rule)

## Context

**The household setup the project assumes:**
- Sonos speakers on a single VLAN (UDM Pro)
- A QNAP NAS holding the music library on the same VLAN, share `Music`, accessible via SMB/CIFS
- An iPhone (12 or newer) on the same VLAN
- A dedicated Raspberry Pi 4 (4 GB) running Raspberry Pi OS Lite 64-bit with read-only root, reachable as `sonos-controller.local` (mDNS) and at a DHCP-pinned IP
- An Ubuntu workstation (Ryzen 9 3950X, 32GB+) where development happens, on the same VLAN

**Why now:** The Sonos app's May 2024 redesign degraded the local-library experience meaningfully. SonoPhone and other third-party alternatives proved the demand for a faster, NAS-first controller. This project takes the same shape but goes further: LAN-only, ad-free, dependency-light, and self-hosted on dedicated hardware.

**Why fork TinySonos rather than build from scratch:** TinySonos already implements ~60–80 hours of backend work the PRD would have specified independently — Python + SoCo + FastAPI-style web service, REST + SSE, host-networked Docker, single-threaded command queue, multi-room grouping. Forking captures all of it under MIT terms. The time saved redirects to the iPhone PWA frontend, the SQLite library cache, the alphabet jump-bar, and the optimistic-UI volume model — the parts that actually differentiate this product. Section 4.2 of `PRD/Sonos_Controller_PRD_v0.4.docx` is the authoritative reuse-vs-change boundary.

**Audio data never flows through the Pi.** When the user picks a track, the backend tells the Sonos coordinator (via SoCo) to play `x-file-cifs://NAS/Music/...`, and the speaker opens its own SMB connection to the NAS. The Pi never proxies audio. This is the single most important architectural decision after the choice to fork.

## Constraints

- **Tech stack — Python 3.12+ + FastAPI + SoCo** — inherited from TinySonos and re-affirmed by the PRD (PRD §4.1.3). SoCo pinned in `pyproject.toml`; do not auto-bump.
- **Tech stack — SQLite for the library cache** — single file on `/data`. Schema in PRD §11; queries parameterised; FTS5 reserved as a fallback if substring search becomes slow at scale (PRD §13.3 R3).
- **Tech stack — Docker with `network_mode: host`** — required for UPnP/SSDP discovery; bridge mode silently breaks discovery. Multi-arch (amd64 dev, arm64 prod) from a single buildx pipeline.
- **Compatibility — iOS Safari 16+** — PWA target; tested on iPhone 12 and newer (PRD §7.4).
- **Performance — sub-100 ms volume, sub-500 ms search/list** — PRD §7.1; if a feature compromises this, the feature loses.
- **Reliability — 90-day uptime, <60s automatic recovery** — PRD §7.2; restart policy `unless-stopped`, healthcheck wired, watchdog catches stuck-discovery.
- **Security — LAN-only, no telemetry, no internet egress required** — PRD §7.3, §2.3. Bearer-token auth on all `/api/*`. Token never logged. NAS credentials never logged.
- **Storage — only `/data` is writeable on the Pi** — read-only root via overlayroot. Container must fail loudly if `/data` is unwritable. SD-card corruption is not an accepted failure mode.
- **License — MIT, inherited from TinySonos** — preserve `LICENSE`, retain upstream attribution in README, send general-purpose fixes upstream as PRs (PRD §4.2.5, §7.6).
- **Network — single VLAN for Pi/speakers/NAS/iPhone** — multicast across VLANs is the most common cause of subtle Sonos failures and is deliberately avoided (PRD §4.1.8).
- **Dev workflow — Claude Code + GSD v1, per-project install** — PRD §5; Claude Code launched with `--dangerously-skip-permissions`; defence-in-depth lives in `.claude/settings.json` deny-list and SSH key boundary.

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| **PWA, not native iOS** (PRD §4.1.1) | Native requires Mac+Xcode+Apple Developer; PWA delivers ~all of native UX from web tech that aligns with existing skill set. Trade-off: no lock-screen / Watch / CarPlay — accepted. | ✓ Locked v0.1 |
| **Dedicated Raspberry Pi 4 host** (PRD §4.1.4, §13.1 Q1) | Co-tenant hosting on QNAP or workstation introduces unrelated failure modes (firmware updates, container contention, sleep cycles). Single-purpose hardware is the cheapest reliability win. | ✓ Locked v0.2 |
| **Fork jasonacox/TinySonos** (PRD §4.2, §13.1 Q7) | TinySonos's architecture (Python+SoCo+FastAPI+REST+SSE+host-net Docker) matches what the PRD specified independently in v0.1 — strong signal. MIT-licensed; ~60–80 hours of work captured for free. Time saved redirects to the PWA. | ✓ Locked v0.3 |
| **Claude Code + GSD v1** (PRD §5, §13.1 Q9) | GSD's discuss→plan→execute→verify→ship cycle maps onto the PRD's phased build verbatim. Per-project install keeps workflow with the codebase. | ✓ Locked v0.4 |
| **`x-file-cifs://` URIs, not Pi-proxied HTTP** (PRD §4.1.6, §4.2.2) | Speakers stream directly from NAS over SMB. Removes Pi from audio path entirely → faster, more reliable, no transcoding. Replaces TinySonos's port-54000 file server. | — Pending Phase 2 implementation |
| **SQLite library cache on `/data`** (PRD §4.1.5, §4.2.2) | Querying the Sonos index through SoCo for every browse is slow at scale. Local SQLite mirror makes browse/search sub-500 ms. Cache is rebuildable; not source of truth. | — Pending Phase 2 implementation |
| **Bearer-token auth, single shared token** (PRD §13.2 Q5) | Multi-user accounts add complexity for no clear v1 benefit; revisit if multiple household members start using heavily. Token at `/data/token`, never logged. | — Pending Phase 2 implementation |
| **Same VLAN for Pi+NAS+speakers+iPhone** (PRD §4.1.8) | Multicast across VLANs is the dominant cause of DIY Sonos failures. Simplicity beats configurability. | ✓ Operating constraint |
| **Balanced model profile** (PRD §5.3.3) | Opus for planning where reasoning matters; Sonnet for execution and verification. Tunable via `/gsd-set-profile` later. | ✓ Locked at GSD init |
| **YOLO GSD mode** | CLAUDE.md already commits to launching Claude Code with `--dangerously-skip-permissions`; YOLO is the matching GSD setting. Defence-in-depth via `.claude/settings.json` deny-list. | ✓ Locked at GSD init |
| **PRD-shaped 7-phase roadmap** (PRD §12) | Phases 0–6 in the PRD are the build plan. GSD phases mirror them 1:1 — Phase 0 (env), Phase 1 (fork+learn), Phase 2 (harden backend), Phase 3 (PWA skeleton), Phase 4 (core interactions), Phase 5 (polish), Phase 6 (Pi cutover). | ✓ Locked at GSD init |

## Open Questions

Carried forward from PRD §13.2 — these are decisions deferred to the right phase, not blockers for Phase 0:

- **Q2 — Album art serving:** proxy+cache vs URI rewrite. Examine TinySonos's existing art-resolution logic before deciding. (Phase 2.)
- **Q3 — Library cache: authoritative vs hint?** Authoritative is faster; hint is more correct after manual library changes. (Phase 2.)
- **Q4 — Frontend framework:** vanilla JS vs Vue 3 vs React. PRD recommends vanilla or Vue; React likely overkill. (Phase 3.)
- **Q6 — Pi backup strategy:** nightly tarball of `/data` to QNAP via rsync is the leading candidate. (Phase 6.)
- **Q8 — Upstream sync cadence:** current default is "cherry-pick fixes when relevant; don't auto-merge." Revisit if upstream becomes more active. (Ongoing.)

## References

- **Specification:** `PRD/Sonos_Controller_PRD_v0.4.docx` (definitive)
  - §4.2 = reuse-vs-change boundary
  - §6 = functional requirements with priority
  - §7 = NFRs (performance, reliability, security, compatibility, accessibility, license)
  - §10 = canonical API spec
  - §11 = SQLite library-cache schema
  - §12 = 7-phase build plan (matches GSD roadmap)
- **Operating manual:** `CLAUDE.md` (root) — how Claude Code should behave inside this repo
- **Path-scoped rules:** `.claude/rules/backend-python.md`, `.claude/rules/deployment.md`
- **Upstream:** https://github.com/jasonacox/TinySonos (MIT)
- **Upstream-archived CLAUDE.md:** `docs/CLAUDE_UPSTREAM.md` (TinySonos's own dev notes, preserved for reference)
- **Frontend repo (separate):** to be created at `sonos-controller-pwa` per PRD §5.6

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gsd-transition`):
1. Requirements invalidated? → Move to Out of Scope with reason.
2. Requirements validated? → Move to Validated with phase reference.
3. New requirements emerged? → Add to Active.
4. Decisions to log? → Add to Key Decisions.
5. "What This Is" still accurate? → Update if drifted.

**After each milestone** (via `/gsd-complete-milestone`):
1. Full review of all sections.
2. Core Value check — still the right priority?
3. Audit Out of Scope — reasons still valid?
4. Update Context with current state.

---
*Last updated: 2026-05-07 after initialization (synthesized from PRD v0.4)*
