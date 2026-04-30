# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What is Maigret

Maigret is an OSINT tool that collects information about a person by username, searching across thousands of social networks and websites. It supports multiple identifier types (not just usernames), recursive cross-platform search, and can generate reports in many formats (HTML, PDF, CSV, JSON, XMind, Markdown, graph).

## Commands

**Install dependencies:**
```sh
poetry install          # production deps only
poetry install --with dev  # include dev/test deps
```

**Run:**
```sh
poetry run maigret <username>
python3 -m maigret <username>
```

**Run tests:**
```sh
make test                        # full suite with coverage report
pytest tests/test_checking.py    # single test file
pytest tests/test_checking.py::test_checking_by_status_code  # single test
pytest --lf -vv                  # rerun only last failures
```

**Lint and format:**
```sh
make lint    # flake8 (syntax/undefined names + warnings) + mypy
make format  # black (skip-string-normalization)
```

**Pre-commit hook** (runs automatically on `git commit`):  
Regenerates `sites.md` and `db_meta.json` from the database JSON, then stages them. Activate with:
```sh
git config core.hooksPath .githooks
```

**Web interface:**
```sh
poetry run maigret --web          # starts Flask on port 5000
poetry run maigret --web 8080     # custom port
```

**DB self-check:**
```sh
poetry run maigret --self-check --auto-disable --diagnose
```

## Architecture

### Core search pipeline

1. **`maigret/maigret.py`** — CLI entry point and orchestration. `main()` parses args, loads `Settings`, loads `MaigretDatabase`, then calls `maigret()` (the search function) in a loop for each username. Handles recursive search by extracting new IDs from results and feeding them back into the queue.

2. **`maigret/checking.py`** — The actual checking engine. The top-level `maigret()` coroutine creates checker objects, sets up an `AsyncioQueueGeneratorExecutor`, and dispatches `check_site_for_username()` for each site in parallel. Result classification logic is in `process_site_result()` — it handles three check types:
   - `status_code`: HTTP 2xx = found
   - `message`: presence/absence string matching in response body
   - `response_url`: redirect behavior + presence strings

3. **`maigret/executors.py`** — Async concurrency. `AsyncioQueueGeneratorExecutor` (the one in active use) is an async generator that yields results as they complete. The other executor classes are deprecated.

4. **`maigret/sites.py`** — Data model. `MaigretSite` represents one site entry with all its check configuration. `MaigretDatabase` loads/saves `data.json` and provides filtering methods like `ranked_sites_dict()` (filter by top N Alexa rank, tags, ID type, disabled status).

5. **`maigret/result.py`** — `MaigretCheckResult` and `MaigretCheckStatus` enum (CLAIMED/AVAILABLE/UNKNOWN/ILLEGAL).

### Checker abstraction

`checking.py` defines multiple `CheckerBase` subclasses selected per-site:
- `SimpleAiohttpChecker` — default async HTTP client
- `ProxiedAiohttpChecker` — SOCKS proxy support
- `CurlCffiChecker` — browser TLS fingerprint impersonation (used when site has `"tls_fingerprint"` in its `protection` field, requires `curl-cffi`)
- `AiodnsDomainResolver` — DNS-only check for domain sites
- `CheckerMock` — no-op placeholder when a protocol checker isn't needed

### Sites database (`maigret/resources/data.json`)

The canonical database is a JSON file with three top-level keys: `sites`, `engines`, and `tags`. Each site entry uses camelCase keys (converted to snake_case internally by `MaigretSite.__init__`). Key fields per site:
- `checkType`: `"message"`, `"status_code"`, or `"response_url"`
- `presenseStrs` / `absenceStrs`: substrings that indicate profile exists/doesn't exist
- `usernameClaimed` / `usernameUnclaimed`: test accounts used by `--self-check`
- `tags`: categories, also used for filtering (`--tags`, `--exclude-tags`)
- `disabled`: skipped unless `--use-disabled-sites`
- `protection`: list of protection mechanisms, e.g. `["tls_fingerprint"]`
- `engine`: reference to a shared engine configuration in the `engines` dict

The database auto-updates on startup (unless `--no-autoupdate`). The cached copy lives in `~/.maigret/data.json`; the bundled fallback is `maigret/resources/data.json`. Update logic is in `maigret/db_updater.py`.

### Settings

`Settings` loads from three paths in order (later paths override earlier): bundled `maigret/resources/settings.json` → `~/.maigret/settings.json` → `./settings.json`. All settings can be overridden from CLI args.

### Public API (`maigret/__init__.py`)

For library use, the package exports:
- `maigret.search` — the core async `maigret()` coroutine from `checking.py`
- `maigret.cli` — `main()` from `maigret.py`
- `MaigretSite`, `MaigretEngine`, `MaigretDatabase` from `sites.py`
- `Notifier` (`QueryNotifyPrint`) from `notify.py`

### Web interface (`maigret/web/`)

Flask app (`app.py`) with Jinja2 templates. Search runs in a background thread via `threading.Thread`. Results stored in an in-memory dict keyed by timestamp. Reports written to `/tmp/maigret_reports/`. Host defaults to `127.0.0.1`; override with `FLASK_HOST` env var.

### Report generation (`maigret/report.py`)

All report formats (HTML, PDF, CSV, JSON/NDJSON, XMind, Markdown, graph) are generated from a `report_context` dict built by `generate_report_context()`. HTML/PDF use Jinja2 templates in `maigret/resources/`. The graph report uses `pyvis`/`networkx`.

### Supported identifier types

Defined in `checking.py::SUPPORTED_IDS`: `username`, `yandex_public_id`, `gaia_id`, `vk_id`, `ok_id`, `wikimapia_uid`, `steam_id`, `uidme_uguid`, `yelp_userid`.

## Testing conventions

- `pytest.ini` sets `asyncio_mode=auto` — all async tests run without explicit `@pytest.mark.asyncio`.
- Tests that make real network calls are marked `@pytest.mark.slow`.
- Test fixtures are in `tests/conftest.py`. Key fixtures: `default_db` (loads the real `data.json`), `test_db` (loads `tests/db.json`), `local_test_db` (loads `tests/local.json` — for local HTTP server tests using `pytest-httpserver`).
- `tests/local.json` defines fake sites pointing to `localhost:8989` for deterministic checker tests.
- Reports created during tests are auto-cleaned by the `reports_autoclean` autouse fixture.

## Adding a new site

**Interactive (beginner):** `maigret --submit <profile_url>`

**Manual:** Edit `maigret/resources/data.json` directly. Required fields: `url`, `urlMain`, `checkType`, `usernameClaimed`, `usernameUnclaimed`, and at least one of `presenseStrs`/`absenceStrs` (for `message` type). After editing, run `--self-check` to validate. The pre-commit hook will regenerate `sites.md` and `db_meta.json` automatically.
