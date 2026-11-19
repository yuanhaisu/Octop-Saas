# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Canonical guide

`AGENTS.md` is the authoritative agent handbook for this repo — module boundaries, hard bans, i18n rules, DB conventions, cross-platform test rules, and change workflow. It is imported below; follow it, and edit it (not this file) when those conventions change.

@AGENTS.md

## Commands

Always use `uv run …`; never bare `pytest`. On Windows, run `make` from Git Bash.

| Task | Command |
| --- | --- |
| Install deps | `make install` (`uv sync`) |
| Enable git hooks (once per clone) | `make install-hooks` |
| Ship bar | `make all` — format-all + lint + typecheck + test |
| All tests (no LLM calls) | `uv run pytest -m "not live"` |
| Unit / integration only | `uv run pytest tests/unit -x -q` / `uv run pytest tests/integration -x -q` |
| One test | `uv run pytest tests/unit/api/test_app.py::test_name -q` |
| Fast suite (skip slow) | `make test-fast` |
| Lint / format / typecheck | `make lint` / `make format` / `make typecheck` |
| i18n key parity | `uv run pytest tests/unit/i18n -q` |
| Frontend tests | `cd dashboard && npm test` (Vitest) |
| Frontend typecheck | `cd dashboard && npx tsc -b` |
| Dev servers | `make dev` (Vite :5173 + `octop run` :8088) |
| Build dashboard into wheel | `make build-frontend` → `src/octop/dashboard/` |

The installed pre-commit hook (`.githooks/pre-commit`) does **not** run `make all`: it runs `make precommit` (format-all, lint, typecheck, and pytest-testmon-scoped tests) plus `npm run build` in `dashboard/`. Emergency bypass: `SKIP_PRECOMMIT=1 git commit …`. CI (Linux + Windows) runs `make install`, `make lint`, `make typecheck`, `make test`.

## Architecture

**One Python process, one wheel.** FastAPI + uvicorn backend, React/Vite SPA, and Click CLI, shipped as `octop` (`octop.cli.main:cli`). No worker process, no external queue; restart rebuilds state from the control-plane DB (SQLite WAL by default, PostgreSQL optional).

```
dashboard/  ──HTTP/WS──►  api/routers/  ──►  infra/  ──►  octop-harness
cli/  ──►  launch.py  ──►  api/ + infra/                    │
                                                            ▼
                                             agent workspace (BackendWorkspace)
IM channels ──► octop-gateway ──► infra/gateway/processor.py ─┤
cron         ──► infra/cron/  ─────────────────────────────────┘
```

- `launch.py` and `infra/server.py` are the composition root: `OctopServer.start()` wires DB → `SharedServices` → `AgentManager` → `Gateway` → `CronManager`. `launch.py` is the only module allowed to import both `infra/server` and `api/app`.
- Dependencies point inward: `infra/` never imports `api/`, `cli/`, or `launch.py`; routers stay thin and delegate to `infra/`; CLI commands do not import `api/`.
- Routers receive `server: OctopServer = Depends(get_server)`; live singletons live on `server.app_runtime.*` (`AppRuntime`) and repos on `server.services` (`SharedServices`).
- All conversation surfaces (dashboard WS, CLI REPL, IM gateway) drive the same `HarnessAgent` turn; cron pushes through the same session mechanism.

**Sibling libraries own their behavior — do not reimplement them under `src/octop/`:**

| Area | Library |
| --- | --- |
| Agent loop, tools, skills, backends | `octop-harness` |
| IM protocol, media upload, group activation | `octop-gateway` |
| Memory capture/recall/storage | `octop-memory` |
| Chrome CDP, profiles, DOM | `octop-browser` |

Change those in their own repos; here only bump the pin in `pyproject.toml`.

## High-risk invariants

- `src/octop/dashboard/` is a build artifact. Edit `dashboard/` and run `make build-frontend`.
- Agent workspace content I/O must go through `HarnessAgent.workspace` (`BackendWorkspace`), never `Path.read_text`/`write_text` on `~/.octop/agents/<id>/`, and never backend-type branching in Octop.
- Schema changes need both `infra/db/migrations/00N_*.sql` and `00N_*.pg.sql`; bump the `assert v == N` in `tests/unit/db/test_db_pool.py`. Unreleased work folds into the current unreleased migration instead of adding a new number. (The number cited in AGENTS.md may lag; the test is the source of truth.)
- Resource tables: integer `id` PK plus public string `{entity}_id` (ULID) unique; child rows reference the string id, never the integer.
- Server-owned user-visible strings live in `src/octop/i18n/{en,zh}.json` with identical key trees; new `ErrorCode`s also need `apiErrors.*` in `dashboard/src/locales/{en,zh}.json`.
- User-facing times use the server timezone (`GET /api/settings/timezone`, `useServerTimezone()`), never bare `toLocaleString()`.
- Frontend calls the backend only through `/api` typed wrappers in `dashboard/src/api/`.

## Branching and release

`feature/*` → `develop`; `release/x.y.z` → `main` (merge commit) → tag `v*` **on `main` only** → Actions publish PyPI/Docker and sync `main → develop`. Hotfix from `main`, then PR into `develop`. Feature PRs target `develop`, never `main`. Detailed flow: `.cursor/skills/publish/SKILL.md` and `CONTRIBUTING.md`.

## More detail

`docs/architecture.md` (human overview), `docs/api.md`, `docs/cli.md`, `docs/configuration.md`, `docs/adr/`, and the "Where to look" table in AGENTS.md §9.
