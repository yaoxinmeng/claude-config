# Template: `.claude/skills/coding/SKILL.md`

Fill every `<...>` from the fact sheet. Delete any section, row, or bullet with nothing project-specific to say. Lines in `> ` are instructions to you, not content; remove them.

```markdown
---
name: coding
description: <Language> / <framework> specifics for this repo on top of the shared coding standards - runner prefix, layout, framework idioms, and the checks that must pass. Load before writing or editing any code in this project.
---

The `write-code` skill sets the standards for any code; load it first. Everything below is what this repo adds on top. Where the code and this file disagree, the code is right and this file needs updating - say so.

## 1. Stack

| Component | Language | Framework | Package manager | Runner prefix |
|---|---|---|---|---|
| <api> | <Python 3.12> | <FastAPI 0.115> | <uv> | <`uv run`> |

> One row per component. Versions come from the lockfile, never from memory.
> Every command in this file and in `testing` takes the runner prefix.

## 2. Layout

| Kind of file | Lives in | Example |
|---|---|---|
| <HTTP handlers> | <`src/app/api/<resource>.py`> | <`src/app/api/users.py`> |
| <domain models> | <`src/app/models/`> | |
| <config> | <`src/app/settings.py`> | |
| <migrations> | <`alembic/versions/`> | |
| <tests> | <`tests/` mirroring `src/app/`> | |

> Only the kinds this repo actually has. A new kind of file goes where the nearest existing kind lives; never a new top-level directory.

## 3. Idioms

> One bullet per convention the code shows or the user chose, each with its evidence. Cover what matters for this stack: boundary validation, error types, logging, async, ORM sessions, HTTP client, config loading, framework wiring. Skip anything `write-code` already says.

- <Request and response bodies are Pydantic models under `src/app/schemas/`; handlers never read `request.json()` directly (see `src/app/api/users.py`).>
- <Errors raised to the client subclass `AppError` in `src/app/errors.py`; the handler in `src/app/main.py` maps them to status codes. Never raise `HTTPException` from a service.>
- <Logging is `structlog` with `log = structlog.get_logger()` at module level; no f-strings in log calls.>
- <All I/O is async; a sync call in a handler is a bug.>

## 4. Checks

Run every row before reporting work done, in this order. Never install a tool to satisfy this table; if a row fails to start, report it.

| Check | Command |
|---|---|
| Format | <`uv run ruff format .`> |
| Lint | <`uv run ruff check .`> |
| Types | <`uv run pyright`> |
| Fast tests | <`uv run pytest -m "not integration" -q`> |
| Slow tests | <`uv run pytest -m integration -q` - needs Docker> |
| Build | <`npm run build`> |

> Exactly as verified. If the repo defines one task that runs several rows (`make check`, `npm run verify`), list that task first and say what it covers.
> Greenfield: add the line `Unverified - remove this line after the first session runs every row` above the table.

## 5. Dependencies

- Every dependency change goes through `/deps`; never edit <`pyproject.toml` or `uv.lock`> by hand, and never write a version number from memory.
- <Any exclusions the user named, with the reason: "No `requests`; use the `httpx` client already in `src/app/http.py`.">
```
