# Roadmap: Sonos Local Library Controller

## Overview

This roadmap mirrors the seven-phase build plan in PRD §12 (v0.4) verbatim. The journey moves from a working environment (Phase 0), through a fork that runs locally with all upstream functionality intact (Phase 1), through a hardened backend that meets the PRD's must-have API and reliability contracts (Phase 2), through an installable PWA skeleton that shows real data (Phase 3), through every must-have user interaction landing on screen (Phase 4), through visual and operational polish (Phase 5), and finally onto the dedicated Raspberry Pi as the household's primary Sonos controller (Phase 6). The frontend codebase lives in a separate repo (`sonos-controller-pwa`) but its phases are tracked here so requirement coverage is auditable from one document. Audio data never flows through the Pi at any phase; speakers stream directly from the QNAP NAS via `x-file-cifs://` URIs.

## Phases

**Phase Numbering:**
- Integer phases (0, 1, 2, …): Planned milestone work, mirroring PRD §12 phases
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

Decimal phases appear between their surrounding integers in numeric order.

- [ ] **Phase 0: Set up the development environment** - Workstation toolchain, Pi base provisioning, GSD primed
- [ ] **Phase 1: Fork and learn** - TinySonos fork runs locally with all upstream functionality intact and a documented map of the codebase
- [ ] **Phase 2: Harden the backend** - Backend meets must-have requirements TinySonos doesn't yet meet, with API normalised to PRD §10
- [ ] **Phase 3: Frontend skeleton** - Minimal but installable iPhone PWA showing real data on four primary screens
- [ ] **Phase 4: Core interactions** - Every must-have interaction works end-to-end on the PWA
- [ ] **Phase 5: Polish** - App is reliable, pleasant, and ready to leave the development workstation
- [ ] **Phase 6: Deployment to the Pi** - Application lives on dedicated hardware as the household's primary Sonos controller

## Phase Details

### Phase 0: Set up the development environment
**Goal**: A workstation that can plan, build, test, and deploy the project, plus a Raspberry Pi with a baseline OS and network identity ready to receive a container.
**Depends on**: Nothing (first phase)
**Requirements**: (none — environment phase)
**Success Criteria** (what must be TRUE):
  1. Workstation has Claude Code, GSD, gh CLI, Docker buildx with multi-arch support, and Python 3.12+ all installed and verified working
  2. GitHub fork (`ray1caron/sonos-controller`) is created, cloned to the workstation, and the SSH key for GitHub authenticates `git push` without prompts
  3. Raspberry Pi 4 hardware is acquired, flashed with Raspberry Pi OS Lite 64-bit, has its IP pinned via UDM Pro DHCP reservation, and resolves as `sonos-controller.local` via Avahi mDNS
  4. A dedicated SSH key (`~/.ssh/sonos_controller_key`, separate from the GitHub key) authorises only the workstation to administer the Pi, with `~/.ssh/config` alias `sonos-controller` working for `ssh sonos-controller`
**Plans**: TBD

### Phase 1: Fork and learn
**Goal**: A working fork of TinySonos running locally on the workstation, with all upstream functionality intact against real Sonos speakers, plus a documented understanding of how the codebase fits together.
**Depends on**: Phase 0
**Requirements**: F-DM-01, F-DM-02, F-DM-03, F-PC-01, F-PC-02, F-PC-04, F-PC-07, F-VL-02, F-VL-03, F-QM-04, F-QM-06, F-QM-07, F-GR-03, F-GR-05, F-OP-05, F-PL-04, NFR-COMPAT-02, NFR-LIC-01, NFR-LIC-02, NFR-LIC-03
**Success Criteria** (what must be TRUE):
  1. The forked TinySonos backend runs in Docker on the workstation, auto-discovers every real Sonos speaker on the LAN, and survives speaker join/leave/regroup events without restart
  2. The upstream-equivalent web UI (still served from the original `web/` for now) executes play, pause, skip, shuffle, repeat, mute, group volume, group dissolve, and queue clear against real speakers — verified manually
  3. A `tests/smoke/` suite tagged `@pytest.mark.smoke` exercises every endpoint above with `pytest -m smoke` and passes against the running container
  4. `/gsd-map-codebase` has been run, and the developer can answer in under one minute where any given upstream feature lives in the source tree
  5. `LICENSE` is preserved unmodified in the source tree, the README credits TinySonos with a link to `jasonacox/TinySonos`, and the fork remains MIT-licensed
**Plans**: TBD

### Phase 2: Harden the backend
**Goal**: The backend meets the must-have requirements TinySonos doesn't yet meet — bearer-token auth, the SQLite library cache, NAS-direct playback URIs, the canonical `/api/` shape, structured health and logging, the watchdog, and a multi-arch container image.
**Depends on**: Phase 1
**Requirements**: F-LB-01, F-LB-08, F-LB-09, F-SR-01, F-SR-02, F-SR-03, F-SR-04, F-OP-02, F-OP-03, F-OP-04, NFR-REL-04, NFR-REL-07, NFR-COMPAT-03, NFR-SEC-02, NFR-SEC-03, NFR-PERF-02, NFR-PERF-03
**Success Criteria** (what must be TRUE):
  1. Every `/api/*` request without a valid bearer token returns 401, and a token generated at first run lives at `/data/token` and is never logged at any level
  2. Track URIs sent to Sonos coordinators are `x-file-cifs://NAS/Music/...` and audio bypasses the Pi entirely; the legacy port-54000 HTTP file-server code path is gone
  3. `/api/library/*` endpoints serve artists, albums, tracks, genres, and folders from a SQLite cache on `/data`, return any browse view in under 500 ms for a 50,000-track library, and self-heal if the cache file is deleted between runs
  4. `/api/search` returns artists, albums, and tracks aggregated in one response in under 500 ms for the same library, with case-insensitive substring matching
  5. `/api/healthz` returns structured JSON with discovery state, library cache age, and SSE subscriber count, and the watchdog self-exits the FastAPI process if discovery returns zero rooms for more than 60 s after previously having rooms
  6. `docker buildx` produces a single multi-arch image (`linux/amd64` + `linux/arm64`) and the backend makes no outbound calls beyond the LAN
**Plans**: TBD
**UI hint**: no

### Phase 3: Frontend skeleton
**Goal**: A minimal but installable PWA in the separate `sonos-controller-pwa` repo, with the four primary screens laid out as static structures fed by real backend data, plus PWA installability on iPhone.
**Depends on**: Phase 2
**Requirements**: F-DM-04, F-LB-02, F-LB-07, F-NP-01, NFR-PERF-04, NFR-COMPAT-01
**Success Criteria** (what must be TRUE):
  1. The PWA renders four primary screens — Now Playing, Library Browse, Rooms, Search/Queue — each populated by real data from the backend's `/api/*` endpoints over an authenticated client
  2. The Rooms screen shows each speaker's current playback state live via SSE, and the Library Browse screen displays album art thumbnails plus recently-added and recently-played sections
  3. The PWA manifest, service worker, and shell caching let an iPhone running iOS Safari 16+ install the app to the home screen, and a cold start to interactive (with cached shell) completes in under 2 s
  4. The Now Playing screen shows large album art, track metadata, a static progress bar, and transport-control affordances (interactivity lands in Phase 4)
**Plans**: TBD
**UI hint**: yes

### Phase 4: Core interactions
**Goal**: Every must-have user interaction works end-to-end — volume sliders feel instantaneous, the alphabet jump-bar makes long lists usable, touch-to-play and the long-press action sheet land, the queue supports drag and swipe, rooms can be grouped and ungrouped, search results are aggregated, and the Now Playing view is fully wired.
**Depends on**: Phase 3
**Requirements**: F-PC-03, F-VL-01, F-VL-04, F-VL-05, F-LB-03, F-LB-04, F-LB-05, F-QM-01, F-QM-02, F-QM-03, F-GR-01, F-GR-02, F-GR-06, F-NP-02, F-NP-03, F-NP-04, F-NP-05, F-PL-01, NFR-PERF-01
**Success Criteria** (what must be TRUE):
  1. Per-speaker volume sliders update optimistically in the UI, are throttled to ≤20 requests/sec/slider during drag, and produce audible volume changes on the speaker within 100 ms of touch
  2. Long alphabetised lists show a right-edge alphabet jump-bar; tapping an album cover plays it immediately (Touch-to-Play); long-pressing any track or album reveals a full action sheet (play now / play next / add to end / replace queue / add to playlist)
  3. The queue view is reorderable via drag, supports swipe-to-delete, and keeps the now-playing row always visible at the top
  4. The Rooms screen lets the user add a room to a group via tap or drag, the coordinator is clearly identified within each group, and the seek bar on Now Playing scrubs the current track
  5. The Now Playing view exposes volume for the active room/group, the queue, and a room/group switcher without leaving the screen, with album art occupying the dominant visual area
  6. Sonos playlists and Sonos favourites are browsable via `/api/playlists` and rendered in the PWA
**Plans**: TBD
**UI hint**: yes

### Phase 5: Polish
**Goal**: The app is reliable under real-world failure modes, pleasant to use, accessible, themed, and stripped of TinySonos legacy paths and the original `web/` UI.
**Depends on**: Phase 4
**Requirements**: F-PC-05, F-PC-06, F-VL-06, F-LB-06, F-SR-05, F-QM-05, F-GR-04, F-PL-02, F-PL-03, F-OP-01, F-OP-06, NFR-REL-01, NFR-REL-03, NFR-A11Y-01, NFR-A11Y-02, NFR-A11Y-03, NFR-A11Y-04
**Success Criteria** (what must be TRUE):
  1. A settings screen exposes default tap action (play now vs add to end), light/dark/auto theme, and a manual library refresh trigger; theme switching works without restart
  2. Every screen has explicit loading, empty, and error states, including graceful handling when a speaker disappears (room shown as offline, app does not crash)
  3. The container's healthcheck is verified end-to-end: forced kill produces an `unless-stopped` recovery cycle, and the container survives a long soak run
  4. All Should-priority interactions land — crossfade, replay-previous, numeric volume popup during drag, party mode, save-queue-as-playlist, playlist create/rename/delete, add-to-playlist, clear-search and recent-searches
  5. Accessibility passes: tap targets ≥44×44, dynamic type honoured, WCAG AA contrast met, and critical state changes (now playing, room switch) are announced via `aria-live`
  6. TinySonos legacy URL aliases and the original `web/` UI are removed from the deployed image
**Plans**: TBD
**UI hint**: yes

### Phase 6: Deployment to the Pi
**Goal**: The application lives on its dedicated Raspberry Pi 4, the household's iPhone is the primary controller, and the system survives a 7-day cutover trial without intervention.
**Depends on**: Phase 5
**Requirements**: NFR-REL-02, NFR-REL-05, NFR-REL-06, NFR-PERF-05, NFR-PERF-06, NFR-SEC-01, NFR-SEC-04, NFR-SEC-05
**Success Criteria** (what must be TRUE):
  1. The Pi runs Raspberry Pi OS Lite with the root filesystem mounted read-only via overlayroot; only `/data` is writeable, and the container fails loudly if `/data` is unwritable
  2. The deployed backend's resident memory stays below 250 MB and CPU below 5% of one Pi 4 core at idle (below 30% during indexing) over a sustained run
  3. The backend listens only on the Pi's LAN IP; mean time to recovery from any controller failure is under 60 s automatically; and Pi power-cycle to working controller completes in under 60 s
  4. The Pi is hardened (SSH key-only, fail2ban running, automatic security updates configured) and only the workstation's dedicated SSH key authorises administration
  5. A 7-day cutover trial completes with the iPhone PWA as the household's primary Sonos controller, with no manual intervention beyond expected daily use
**Plans**: TBD

## Progress

**Execution Order:**
Phases execute in numeric order: 0 → 1 → 2 → 3 → 4 → 5 → 6. Phase 0 work proceeds in parallel with Phase 1 once the workstation toolchain is verified, since Phase 1 is workstation-only and does not depend on the Pi being live yet.

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 0. Set up the development environment | 0/TBD | In progress (workstation done; Pi-side bootstrap remaining) | - |
| 1. Fork and learn | 0/TBD | Not started | - |
| 2. Harden the backend | 0/TBD | Not started | - |
| 3. Frontend skeleton | 0/TBD | Not started | - |
| 4. Core interactions | 0/TBD | Not started | - |
| 5. Polish | 0/TBD | Not started | - |
| 6. Deployment to the Pi | 0/TBD | Not started | - |

---
*Roadmap created: 2026-05-07 (synthesized from PRD v0.4 §12; mirrors PRD phase structure verbatim)*
