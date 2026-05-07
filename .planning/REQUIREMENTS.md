# Requirements: Sonos Local Library Controller

**Defined:** 2026-05-07 (synthesized from PRD v0.4 §6 and §7)
**Core Value:** Volume changes feel instantaneous (<100 ms), library searches return in <500 ms, common tasks take ≤2 taps, container survives 90 days continuous, MTR <60 s.

> Requirement IDs preserve the PRD's `F-<area>-NN` numbering so every line traces directly to a row in PRD §6 or §7. Priority is the PRD priority colour: **Must** = v1 committed; **Should** = v1 stretch (kept in v1 list); **Nice** = v2; **Out** = Out of Scope.
>
> The **Source** annotation indicates work locus: *TinySonos* = already implemented upstream, *TinySonos+* = upstream baseline plus meaningful additional work, *New* = built from scratch in the fork.

## v1 Requirements

### Speaker discovery and management

- [ ] **F-DM-01** (Must): Auto-discover all Sonos speakers on the LAN at backend start. *Source: TinySonos*
- [ ] **F-DM-02** (Must): Maintain an up-to-date list of speakers, group topology, and coordinator assignments. *Source: TinySonos*
- [ ] **F-DM-03** (Must): Re-discover speakers when topology changes (speaker added, removed, regrouped). *Source: TinySonos*
- [ ] **F-DM-04** (Must): Display each speaker's current playback state on the Rooms screen. *Source: TinySonos+ (frontend new)*

### Playback control

- [ ] **F-PC-01** (Must): Play, pause, and stop the currently selected room. *Source: TinySonos*
- [ ] **F-PC-02** (Must): Skip to next track and to previous track. *Source: TinySonos*
- [ ] **F-PC-03** (Must): Seek within the current track via a draggable progress bar. *Source: TinySonos+ (frontend new)*
- [ ] **F-PC-04** (Must): Toggle shuffle and repeat (off / one / all). *Source: TinySonos*
- [ ] **F-PC-05** (Should): Toggle crossfade for the current group. *Source: New*
- [ ] **F-PC-06** (Should): Replay the previously played track in the queue. *Source: New*
- [ ] **F-PC-07** (Must): Route all transport commands to the group coordinator, raising a clear error if attempted on a follower. *Source: TinySonos*

### Volume

- [ ] **F-VL-01** (Must): Per-speaker volume sliders visible on the Rooms screen with no mode change required. *Source: TinySonos+ (frontend new)*
- [ ] **F-VL-02** (Must): Group volume slider that scales all member volumes proportionally. *Source: TinySonos*
- [ ] **F-VL-03** (Must): Mute toggle per speaker and per group. *Source: TinySonos*
- [ ] **F-VL-04** (Must): Volume changes appear in the UI optimistically before backend confirmation. *Source: New (frontend pattern)*
- [ ] **F-VL-05** (Must): Volume changes during a drag are throttled to ≤20 requests per second per slider. *Source: New (frontend pattern)*
- [ ] **F-VL-06** (Should): Numeric volume value popup appears while dragging. *Source: New (frontend pattern)*

### Music library browsing

- [ ] **F-LB-01** (Must): Browse the library by Artist, Album, Album-Artist, Genre, Track, and Folder. *Source: TinySonos+ (cache new)*
- [ ] **F-LB-02** (Must): Display album art thumbnails in list views and large album art on detail views. *Source: TinySonos+*
- [ ] **F-LB-03** (Must): Alphabet jump-bar on the right edge of long alphabetised lists. *Source: New (signature feature)*
- [ ] **F-LB-04** (Must): Tapping an album cover plays it immediately, replacing the queue ("Touch to Play" default). *Source: New (frontend pattern)*
- [ ] **F-LB-05** (Must): Long-press on any list item reveals a full action sheet (play now, play next, add to end, replace queue, add to playlist). *Source: New (frontend pattern)*
- [ ] **F-LB-06** (Should): User-configurable default tap action (play now vs add to end). *Source: New*
- [ ] **F-LB-07** (Should): Recently added and recently played sections on the Library home screen. *Source: TinySonos+ (recent already partial)*
- [ ] **F-LB-08** (Must): Manual library re-index trigger with progress indicator. *Source: New*
- [ ] **F-LB-09** (Should): Background re-index every 6 hours (configurable). *Source: New*

### Search

- [ ] **F-SR-01** (Must): Single search field that queries artists, albums, and tracks simultaneously. *Source: New*
- [ ] **F-SR-02** (Must): Results presented as one screen, grouped into Artists / Albums / Tracks sections, with counts visible. *Source: New*
- [ ] **F-SR-03** (Must): Results return within 500 ms for libraries up to 50,000 tracks. *Source: New (cache-driven)*
- [ ] **F-SR-04** (Must): Search is case-insensitive and matches substrings within metadata fields. *Source: New*
- [ ] **F-SR-05** (Should): Clear search button and recent searches list. *Source: New*

### Queue management

- [ ] **F-QM-01** (Must): View current queue with the now-playing track always visible at the top. *Source: New (frontend, with TinySonos backend)*
- [ ] **F-QM-02** (Must): Reorder tracks via drag-and-drop with a clear drag handle. *Source: New*
- [ ] **F-QM-03** (Must): Remove individual tracks via swipe-to-delete. *Source: New (frontend pattern)*
- [ ] **F-QM-04** (Must): Clear the entire queue with a confirmation prompt. *Source: TinySonos*
- [ ] **F-QM-05** (Should): Save the current queue as a Sonos playlist. *Source: New*
- [ ] **F-QM-06** (Must): "Add to queue", "Play next", and "Replace queue" available from any track or album action sheet. *Source: TinySonos+*
- [ ] **F-QM-07** (Must): Queue automatically follows playback progress. *Source: TinySonos*

### Multi-room and grouping

- [ ] **F-GR-01** (Must): View all groups and standalone rooms on a single Rooms screen. *Source: TinySonos+ (UI new)*
- [ ] **F-GR-02** (Must): Add a room to an existing group via tap or drag. *Source: TinySonos+*
- [ ] **F-GR-03** (Must): Remove a room from its group with a single tap. *Source: TinySonos*
- [ ] **F-GR-04** (Should): "Party Mode" button groups all rooms under a single coordinator. *Source: TinySonos*
- [ ] **F-GR-05** (Must): Group composition changes propagate to all clients within 1 second. *Source: TinySonos*
- [ ] **F-GR-06** (Should): UI clearly identifies the coordinator within each group. *Source: New*

### Playlists and favourites

- [ ] **F-PL-01** (Must): Browse Sonos playlists and Sonos favourites. *Source: New (TinySonos M3U-only today)*
- [ ] **F-PL-02** (Should): Create, rename, and delete Sonos playlists. *Source: New*
- [ ] **F-PL-03** (Should): Add tracks, albums, and artists to an existing playlist. *Source: New*
- [ ] **F-PL-04** (Should): Browse imported playlists from the music library (M3U, etc.). *Source: TinySonos*

### Now Playing

- [ ] **F-NP-01** (Must): Full-screen Now Playing view with large album art, track metadata, progress bar, and transport controls. *Source: New*
- [ ] **F-NP-02** (Must): Volume slider for the active room or group accessible without leaving the Now Playing view. *Source: New*
- [ ] **F-NP-03** (Must): Quick access to the queue from the Now Playing view. *Source: New*
- [ ] **F-NP-04** (Must): Quick access to the room/group switcher from the Now Playing view. *Source: New*
- [ ] **F-NP-05** (Must): Album art occupies a large portion of the screen with no peripheral clutter. *Source: New*

### Settings and operations

- [ ] **F-OP-01** (Must): Settings screen exposes default tap action, theme (light/dark/auto), and library refresh trigger. *Source: New*
- [ ] **F-OP-02** (Must): Backend exposes a `/api/healthz` endpoint returning JSON with discovery state and library cache freshness. *Source: New*
- [ ] **F-OP-03** (Must): Bearer token authentication on all backend endpoints, configurable in a single config file. *Source: New*
- [ ] **F-OP-04** (Must): Logs are written to a file on the writeable volume with rotation, accessible via SSH on the Pi. *Source: New*
- [ ] **F-OP-05** (Must): Backend recovers automatically if Sonos network changes (DHCP renewal, speaker reboot). *Source: TinySonos*
- [ ] **F-OP-06** (Must): Container restarts automatically on failure with no manual intervention. *Source: TinySonos+*

### Non-functional requirements (PRD §7)

- [ ] **NFR-PERF-01** (Must): Volume slider drag — touch-to-audible latency under 100 ms over a quiet LAN. *Source: PRD §7.1*
- [ ] **NFR-PERF-02** (Must): Library list initial render under 500 ms for any browse view. *Source: PRD §7.1*
- [ ] **NFR-PERF-03** (Must): Search response under 500 ms for libraries up to 50,000 tracks. *Source: PRD §7.1*
- [ ] **NFR-PERF-04** (Must): App cold start to interactive under 2 s when service worker has cached the shell. *Source: PRD §7.1*
- [ ] **NFR-PERF-05** (Must): Backend resident memory under 250 MB on the Pi. *Source: PRD §7.1*
- [ ] **NFR-PERF-06** (Must): Backend CPU under 5% of one Pi 4 core at idle, under 30% during indexing. *Source: PRD §7.1*
- [ ] **NFR-REL-01** (Must): Container uptime — 90 days continuous operation expected. Restart policy `unless-stopped`. *Source: PRD §7.2*
- [ ] **NFR-REL-02** (Must): Mean time to recovery from any controller failure under 60 s, automatically. *Source: PRD §7.2*
- [ ] **NFR-REL-03** (Must): Speaker disappearance handled gracefully — affected room shown as offline, app does not crash. *Source: PRD §7.2*
- [ ] **NFR-REL-04** (Must): Library cache rebuild is incremental where possible and self-healing if cache is deleted. *Source: PRD §7.2*
- [ ] **NFR-REL-05** (Must): Pi root filesystem mounted read-only; only `/data` writeable. *Source: PRD §7.2*
- [ ] **NFR-REL-06** (Must): Pi boots back to a working controller within 60 s of power restoration. *Source: PRD §7.2*
- [ ] **NFR-REL-07** (Must): Watchdog kills the FastAPI process if discovery returns zero rooms for >60 s when it had rooms previously. *Source: PRD §7.2*
- [ ] **NFR-SEC-01** (Must): Backend listens only on the LAN interface; bound to the Pi's LAN IP. *Source: PRD §7.3*
- [ ] **NFR-SEC-02** (Must): All `/api/*` endpoints require a bearer token. Token generated at first install, stored at `/data/token`. *Source: PRD §7.3*
- [ ] **NFR-SEC-03** (Must): No user data sent off the LAN. No telemetry, no analytics, no cloud. *Source: PRD §7.3*
- [ ] **NFR-SEC-04** (Must): Pi hardened — SSH key-only, fail2ban, automatic security updates. *Source: PRD §7.3*
- [ ] **NFR-SEC-05** (Must): Dedicated SSH key authorises only the workstation to administer the Pi. *Source: PRD §7.3*
- [ ] **NFR-COMPAT-01** (Must): Frontend supports iOS Safari 16 and later; tested on iPhone 12 and newer. *Source: PRD §7.4*
- [ ] **NFR-COMPAT-02** (Must): Backend supports Sonos S1 and S2 firmware via SoCo's UPnP control. *Source: PRD §7.4*
- [ ] **NFR-COMPAT-03** (Must): Backend Docker image targets linux/arm64 (Pi 4) and linux/amd64 (workstation), single multi-arch image. *Source: PRD §7.4*
- [ ] **NFR-A11Y-01** (Must): All interactive elements have a tap target of at least 44×44 points. *Source: PRD §7.5*
- [ ] **NFR-A11Y-02** (Must): Text uses iOS dynamic type sizing where supported. *Source: PRD §7.5*
- [ ] **NFR-A11Y-03** (Must): Colour contrast ratios meet WCAG AA at minimum. *Source: PRD §7.5*
- [ ] **NFR-A11Y-04** (Must): Critical state changes (now playing, room switch) announced via `aria-live`. *Source: PRD §7.5*
- [ ] **NFR-LIC-01** (Must): Fork inherits MIT license from TinySonos and remains MIT licensed. *Source: PRD §7.6*
- [ ] **NFR-LIC-02** (Must): Fork's README prominently credits TinySonos and links to the upstream repo. *Source: PRD §7.6*
- [ ] **NFR-LIC-03** (Must): Original `LICENSE` file from TinySonos preserved in source tree. *Source: PRD §7.6*

## v2 Requirements

Deferred to a future milestone. Tracked but not in current roadmap.

### Speaker management

- **F-DM-05** (Nice): Show speaker model and firmware version in a details view. *Source: New*

### Volume

- **F-VL-07** (Nice): Bass, treble, and loudness EQ controls available in a per-room settings sheet. *Source: New*

### Other v2 candidates

- Alarms and sleep timers (PRD §2.3 — explicitly deferred to v2)
- Sonos playlist creation/edit if not landed in v1 (F-PL-02, F-PL-03 if descoped)
- WireGuard remote-access UX (PRD §7.3 — same auth applies; UX considerations deferred)

## Out of Scope

Explicit v1 exclusions per PRD §2.3 and §4.2.4. Re-adding requires explicit discussion.

| Feature | Reason |
|---------|--------|
| Streaming services (Spotify, Apple Music, Tidal, Amazon Music, etc.) | NAS-first product principle; explicit non-goal §2.3 |
| Internet radio / TuneIn | LAN-only system; non-goal §2.3 |
| Apple Watch support | Impossible from a PWA |
| Lock-screen and Control Center transport controls | PWA platform limitation |
| CarPlay support | PWA platform limitation |
| Speaker setup, firmware updates, stereo-pair configuration | Handled by official Sonos app |
| Voice control / Siri integration | Out of scope §2.3 |
| Public internet access | LAN-only by design; remote access via existing UDM Pro WireGuard if needed |
| TinySonos's HTTP file server (port 54000) | Replaced by `x-file-cifs://` URIs (§4.2.4) |
| TinySonos's Plex export tools (`tools/`) | Outside this product's scope (§4.2.4) |
| TinySonos's original web UI (`web/`) | Replaced by new PWA (§4.2.4) |
| CI/CD via GitHub Actions | Deferred per PRD §5.7; manual smoke + 7-day cutover trial substitute for v1 |
| Public Docker registry pushes | Single-developer deploy via `docker save \| ssh ... docker load` |
| Multi-user / per-account auth | Single shared bearer token; PRD §13.2 Q5 marked re-visit if needed |

## Traceability

Each v1 requirement maps to exactly one phase from PRD §12 / ROADMAP.md. Phase 0 (environment bootstrap) carries no functional or non-functional requirements — it is purely a workstation/Pi setup phase. Phase 1 (Fork & learn) verifies inherited TinySonos requirements work end-to-end against real speakers, plus carries the license/attribution requirements that govern the source tree itself.

| Requirement | Priority | Phase | Status |
|-------------|----------|-------|--------|
| F-DM-01 | Must | Phase 1 | Pending |
| F-DM-02 | Must | Phase 1 | Pending |
| F-DM-03 | Must | Phase 1 | Pending |
| F-DM-04 | Must | Phase 3 | Pending |
| F-PC-01 | Must | Phase 1 | Pending |
| F-PC-02 | Must | Phase 1 | Pending |
| F-PC-03 | Must | Phase 4 | Pending |
| F-PC-04 | Must | Phase 1 | Pending |
| F-PC-05 | Should | Phase 5 | Pending |
| F-PC-06 | Should | Phase 5 | Pending |
| F-PC-07 | Must | Phase 1 | Pending |
| F-VL-01 | Must | Phase 4 | Pending |
| F-VL-02 | Must | Phase 1 | Pending |
| F-VL-03 | Must | Phase 1 | Pending |
| F-VL-04 | Must | Phase 4 | Pending |
| F-VL-05 | Must | Phase 4 | Pending |
| F-VL-06 | Should | Phase 5 | Pending |
| F-LB-01 | Must | Phase 2 | Pending |
| F-LB-02 | Must | Phase 3 | Pending |
| F-LB-03 | Must | Phase 4 | Pending |
| F-LB-04 | Must | Phase 4 | Pending |
| F-LB-05 | Must | Phase 4 | Pending |
| F-LB-06 | Should | Phase 5 | Pending |
| F-LB-07 | Should | Phase 3 | Pending |
| F-LB-08 | Must | Phase 2 | Pending |
| F-LB-09 | Should | Phase 2 | Pending |
| F-SR-01 | Must | Phase 2 | Pending |
| F-SR-02 | Must | Phase 2 | Pending |
| F-SR-03 | Must | Phase 2 | Pending |
| F-SR-04 | Must | Phase 2 | Pending |
| F-SR-05 | Should | Phase 5 | Pending |
| F-QM-01 | Must | Phase 4 | Pending |
| F-QM-02 | Must | Phase 4 | Pending |
| F-QM-03 | Must | Phase 4 | Pending |
| F-QM-04 | Must | Phase 1 | Pending |
| F-QM-05 | Should | Phase 5 | Pending |
| F-QM-06 | Must | Phase 1 | Pending |
| F-QM-07 | Must | Phase 1 | Pending |
| F-GR-01 | Must | Phase 4 | Pending |
| F-GR-02 | Must | Phase 4 | Pending |
| F-GR-03 | Must | Phase 1 | Pending |
| F-GR-04 | Should | Phase 5 | Pending |
| F-GR-05 | Must | Phase 1 | Pending |
| F-GR-06 | Should | Phase 4 | Pending |
| F-PL-01 | Must | Phase 4 | Pending |
| F-PL-02 | Should | Phase 5 | Pending |
| F-PL-03 | Should | Phase 5 | Pending |
| F-PL-04 | Should | Phase 1 | Pending |
| F-NP-01 | Must | Phase 3 | Pending |
| F-NP-02 | Must | Phase 4 | Pending |
| F-NP-03 | Must | Phase 4 | Pending |
| F-NP-04 | Must | Phase 4 | Pending |
| F-NP-05 | Must | Phase 4 | Pending |
| F-OP-01 | Must | Phase 5 | Pending |
| F-OP-02 | Must | Phase 2 | Pending |
| F-OP-03 | Must | Phase 2 | Pending |
| F-OP-04 | Must | Phase 2 | Pending |
| F-OP-05 | Must | Phase 1 | Pending |
| F-OP-06 | Must | Phase 5 | Pending |
| NFR-PERF-01 | Must | Phase 4 | Pending |
| NFR-PERF-02 | Must | Phase 2 | Pending |
| NFR-PERF-03 | Must | Phase 2 | Pending |
| NFR-PERF-04 | Must | Phase 3 | Pending |
| NFR-PERF-05 | Must | Phase 6 | Pending |
| NFR-PERF-06 | Must | Phase 6 | Pending |
| NFR-REL-01 | Must | Phase 5 | Pending |
| NFR-REL-02 | Must | Phase 6 | Pending |
| NFR-REL-03 | Must | Phase 5 | Pending |
| NFR-REL-04 | Must | Phase 2 | Pending |
| NFR-REL-05 | Must | Phase 6 | Pending |
| NFR-REL-06 | Must | Phase 6 | Pending |
| NFR-REL-07 | Must | Phase 2 | Pending |
| NFR-SEC-01 | Must | Phase 6 | Pending |
| NFR-SEC-02 | Must | Phase 2 | Pending |
| NFR-SEC-03 | Must | Phase 2 | Pending |
| NFR-SEC-04 | Must | Phase 6 | Pending |
| NFR-SEC-05 | Must | Phase 6 | Pending |
| NFR-COMPAT-01 | Must | Phase 3 | Pending |
| NFR-COMPAT-02 | Must | Phase 1 | Pending |
| NFR-COMPAT-03 | Must | Phase 2 | Pending |
| NFR-A11Y-01 | Must | Phase 5 | Pending |
| NFR-A11Y-02 | Must | Phase 5 | Pending |
| NFR-A11Y-03 | Must | Phase 5 | Pending |
| NFR-A11Y-04 | Must | Phase 5 | Pending |
| NFR-LIC-01 | Must | Phase 1 | Pending |
| NFR-LIC-02 | Must | Phase 1 | Pending |
| NFR-LIC-03 | Must | Phase 1 | Pending |

**Coverage:**
- v1 Must requirements: 74 (46 functional + 28 NFRs)
- v1 Should requirements: 13 (kept in v1 list, may slip to v2 if time-pressured)
- v2 Nice requirements: 2 explicit (F-DM-05, F-VL-07) plus alarms/sleep, WireGuard UX, Sonos playlist edit if descoped
- Total v1 requirements: 87
- Mapped to phases: 87 / 87 (100%)
- Unmapped: 0

**By phase:**

| Phase | Requirements | Must | Should |
|-------|--------------|------|--------|
| Phase 0 | 0 | 0 | 0 |
| Phase 1 | 20 | 19 | 1 |
| Phase 2 | 17 | 16 | 1 |
| Phase 3 | 6 | 5 | 1 |
| Phase 4 | 19 | 18 | 1 |
| Phase 5 | 17 | 8 | 9 |
| Phase 6 | 8 | 8 | 0 |
| **Total** | **87** | **74** | **13** |

---
*Requirements defined: 2026-05-07*
*Last updated: 2026-05-07 — traceability table populated by gsd-roadmapper agent (PRD §12 phase structure)*
