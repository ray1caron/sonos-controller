---
description: Python-specific rules for backend code
globs: ["src/**", "tests/**", "*.py", "pyproject.toml"]
---

# Backend Python rules

These rules apply when working in Python source files for the backend.

## Code style

- **Formatter:** `black` with default settings (88-char line length)
- **Linter:** `ruff` with config in `pyproject.toml`
- **Type checker:** `mypy --strict` for new code; existing TinySonos code is grandfathered
- **Imports:** Standard library first, then third-party, then local. Sorted alphabetically within each group. `ruff` enforces this.
- **Docstrings:** Google style. Required on public functions and classes; optional on private (`_prefixed`) helpers.

## SoCo usage

- Always go through the `PlaybackController` wrapper for transport commands. It's single-threaded and serialises commands to avoid race conditions on the Sonos coordinator.
- Discovery should use `soco.discover()` with a reasonable timeout (10 seconds). Don't poll faster than every 30 seconds in the background — Sonos's SSDP responses are flaky under tight polling.
- Group operations (join, unjoin) target the **coordinator**, not arbitrary group members. The PlaybackController's group helpers handle this.
- When a SoCo call raises, log the exception with full traceback and let the API endpoint return 500 with a useful error body. Don't catch and swallow.

## FastAPI patterns

- Endpoints live in `src/api/`, organised by resource (rooms, library, queue, etc).
- Use FastAPI's dependency injection for the bearer-token check — one `auth` dependency, attached to every router.
- Pydantic models for request and response bodies. Don't return raw dicts from endpoints.
- Async endpoints. SoCo is sync, so wrap blocking calls in `asyncio.to_thread()` rather than blocking the event loop.

## Database (SQLite library cache)

- The schema is defined in PRD section 11. Migrations live in `src/db/migrations/` and run on startup.
- Use parameterised queries — `cursor.execute("... WHERE id = ?", (id,))`. **NEVER** f-string user input into SQL.
- Indexes are on every column the API filters or sorts by. If a new query is slow, the answer is usually a missing index.
- The cache is rebuildable from Sonos at any time. Treat it as a hot mirror, not the source of truth.

## URI handling

- Track URIs sent to Sonos must be `x-file-cifs://NAS/Music/...`. The original TinySonos `http://host:port/file/...` pattern is removed.
- Build URIs from the indexed file paths in the SQLite cache. Don't construct them ad hoc in the API layer.
- Album art URIs go through the proxy/cache layer (see PRD Q2 resolution). Don't expose Sonos-internal art URLs to the frontend directly.

## Testing

- Smoke tests in `tests/smoke/` exercise real endpoints against a real speaker. Tagged `@pytest.mark.smoke`. Excluded from default `pytest`; run explicitly with `pytest -m smoke`.
- Unit tests in `tests/unit/` are pure logic — no network, no SoCo, no DB. Use fixtures for SoCo mocks.
- Every endpoint has at least one smoke test that hits it with a valid token and verifies the shape of the response.
- Coverage isn't a metric we enforce, but if a function has no test you should ask whether it's worth keeping.

## Error handling

- API endpoints return appropriate HTTP status codes:
  - 400 for invalid client input
  - 401 for missing/invalid auth
  - 404 for unknown resource (e.g. unknown room ID)
  - 409 for state conflicts (e.g. trying to play on an offline speaker)
  - 500 for backend bugs
  - 502 for SoCo/Sonos failures we couldn't recover from
- Error responses have a JSON body: `{"error": "human-readable message", "code": "machine-readable-slug"}`.
- Log the full exception at ERROR level. The client gets a sanitised message; the logs get the trace.
