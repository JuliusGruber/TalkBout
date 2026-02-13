# Plan: Web UI for ViennaTalksBout Tag Cloud

## Context

The backend pipeline (Mastodon stream → PostBuffer → TopicExtractor → TopicStore) is working and produces up to 20 trending topics with lifecycle states (`entering`, `growing`, `shrinking`). Hourly JSON snapshots are saved to `data/snapshots/`. There is **no frontend** yet. This plan adds a web UI that serves a live tag cloud with CSS animations and a time slider for historical browsing.

---

## Architecture

```
Main Thread (uvicorn + FastAPI)          Background Thread (daemon)
┌──────────────────────────────┐        ┌──────────────────────────┐
│ GET /          → index.html  │        │ pipeline.start()         │
│ GET /api/topics → live JSON  │◄─reads─│ Mastodon→Buffer→LLM→Store│
│ GET /api/topics?hour=14      │        └──────────────────────────┘
│ GET /api/health              │
│ GET /api/snapshots           │
└──────────────────────────────┘
```

The TopicStore is already thread-safe — the web server reads it directly.

---

## Files to Create

### 1. `viennatalksbout/web.py` — FastAPI app + pipeline thread management

- **`create_app(store, health, snapshot_dir)`** — app factory (testable with mocked deps)
  - Stores refs on `app.state` (no globals)
- **Routes:**
  - `GET /` → `FileResponse(STATIC_DIR / "index.html")`
  - `GET /api/topics` → calls `store.get_current_topics()`, serializes to JSON
  - `GET /api/topics?hour=14` → constructs filename `topics_YYYYMMDD_14.json`, calls `store.load_snapshot(path)`, returns 404 if missing
  - `GET /api/health` → calls `health.get_status()`, returns metrics dict (excludes `last_post_time` since it's monotonic)
  - `GET /api/snapshots` → globs `snapshot_dir` for today's files, returns list of available hour strings
- **`_run_pipeline_in_background(pipeline)`** — starts pipeline in a daemon thread with `install_signal_handlers=False`
- **`main()`** — entry point: `setup_logging()` → `build_pipeline()` → start background thread → `create_app()` → register shutdown handler → `uvicorn.run()`
- Config env vars: `VIENNATALKSBOUT_WEB_HOST` (default `127.0.0.1`), `VIENNATALKSBOUT_WEB_PORT` (default `8000`)

### 2. `viennatalksbout/static/index.html` — Single-file frontend

Self-contained HTML with embedded `<style>` and `<script>`. No build toolchain, no npm.

**Layout:**
- Dark background (`#0f0f1a`), centered flexbox tag cloud
- Header: "ViennaTalksBout" + subtitle
- Main: `#tag-cloud` div — flexbox, wrap, centered, gap between tags
- Footer: time slider + health dot indicator

**CSS animations mapping to TopicState:**
- `.topic.entering` → `@keyframes fadeIn` (opacity 0→1, scale 0.5→1)
- `.topic.growing` → white/bright color, `transition: all 0.8s ease`
- `.topic.shrinking` → dimmed color, reduced opacity
- `.topic.removing` → `@keyframes fadeOut` (opacity 1→0, scale 1→0.5)

**JS logic (~100 lines):**
- `currentTopics` Map for diffing (keyed by topic name)
- `poll()` — fetches `/api/topics` + `/api/health`, calls `renderTopics()`
- `renderTopics(data)` — diffs new data against `currentTopics`:
  - Disappeared topics: add `.removing` class, `setTimeout(remove, 800)`
  - New topics: create `<span>` with lifecycle class
  - Existing topics: update `font-size` and class (CSS transition handles animation)
- `scoreFontSize(score)` — linear interpolation: score 0→`0.9rem`, score 1→`3.5rem`
- Time slider: `input` event handler switches between live polling and historical snapshot fetch
- Polling every 15 seconds, stops when viewing historical data

### 3. `tests/test_web.py` — API endpoint tests

Following existing patterns: class-based organization, `_make_topic()` factory, `MagicMock(spec=TopicStore)`, `TestClient` from starlette.

**Test classes:**
- `TestIndexEndpoint` — returns 200, HTML content-type, contains `tag-cloud` element
- `TestTopicsEndpoint` — returns topics list, correct fields, empty when no topics, historical by hour, 404 for missing snapshot, 400 for invalid hour
- `TestHealthEndpoint` — returns metrics, initial zero state
- `TestSnapshotsEndpoint` — lists available hours, empty when no files
- `TestCreateApp` — factory wires state correctly

---

## Files to Modify

### 4. `viennatalksbout/ingest.py` — Add `install_signal_handlers` param

**Problem:** `start()` calls `signal.signal()` on lines 232-235 — raises `ValueError` from non-main thread.

**Change in `start()` (line 222):**
```python
def start(self, install_signal_handlers: bool = True) -> None:
```
Wrap lines 232-235 with `if install_signal_handlers:`.

**Change in `stop()` (lines 298-301):** Wrap signal restoration with `try/except ValueError` for safety when called from non-main thread.

### 5. `viennatalksbout/__main__.py` — Route to web mode by default

```python
import sys
if "--ingest" in sys.argv:
    from viennatalksbout.ingest import main
else:
    from viennatalksbout.web import main
main()
```

Web mode is the new default. `--ingest` flag preserves the old pipeline-only mode.

### 6. `pyproject.toml` — Add dependencies + package data

Add to `dependencies`:
```
"fastapi>=0.115.0",
"uvicorn[standard]>=0.30.0",
```

Add `httpx>=0.27.0` to `[project.optional-dependencies] dev` (required by `TestClient`).

Add package data so `static/` is included:
```toml
[tool.setuptools.package-data]
viennatalksbout = ["static/*"]
```

### 7. `requirements.txt` — Add fastapi, uvicorn, httpx

---

## Implementation Order

1. Add dependencies to `pyproject.toml` + `requirements.txt`, pip install
2. Create `viennatalksbout/static/index.html`
3. Create `viennatalksbout/web.py` with `create_app()` and all routes (no pipeline thread yet)
4. Create `tests/test_web.py` — all endpoint tests with mocked store
5. Modify `viennatalksbout/ingest.py` — add `install_signal_handlers` parameter
6. Add pipeline thread management to `web.py` (`_run_pipeline_in_background`, `main()`)
7. Update `viennatalksbout/__main__.py` to route to web mode by default
8. Run full test suite, verify 90%+ coverage

---

## Verification

1. **Tests:** `python -m pytest tests/ -v --tb=short --cov=viennatalksbout --cov-report=term-missing`
2. **Manual:** `python -m viennatalksbout` → open `http://localhost:8000` → see tag cloud (needs `.env` configured with Mastodon + Anthropic credentials)
3. **API check:** `curl http://localhost:8000/api/topics` → JSON response
4. **Pipeline-only mode still works:** `python -m viennatalksbout --ingest`
