# Template: `.claude/skills/testing/SKILL.md`

Fill every `<...>` from the fact sheet. Delete any section, row, or bullet with nothing project-specific to say. Lines in `> ` are instructions to you, not content; remove them.

```markdown
---
name: testing
description: <Test framework> specifics for this repo on top of the shared test standards - the runner, the config already in force, where tests live, which fixtures to reuse, and the framework idioms. Load before writing or editing any test in this project.
---

The `write-tests` skill sets what to test and how to assert it; load it first. Everything below is what this repo adds on top. Where the existing tests and this file disagree, the tests are right and this file needs updating - say so.

## 1. Runner

| Component | Command | Config |
|---|---|---|
| <api> | <`uv run pytest`> | <`[tool.pytest.ini_options]` in `pyproject.toml`> |

> Exactly as verified. One row per component.

Already in force, so never re-declare them:

- <Markers: `integration` (needs Docker), `slow`. Registered in `pyproject.toml`; a new marker is a config change, not a test change.>
- <Plugins: `pytest-asyncio` in auto mode, `pytest-cov` with `fail_under = 85`.>
- <Timeout: 30s per test via `pytest-timeout`.>

## 2. Layout and fixtures

- <Tests live in `tests/`, mirroring `src/app/`: `src/app/api/users.py` is tested by `tests/api/test_users.py`.>
- <Shared fixtures: `tests/conftest.py` (`client`, `db_session`, `settings`), `tests/api/conftest.py` (`auth_headers`). Read every `conftest.py` on the path and reuse before defining a new one.>
- <Factories: `tests/factories.py` builds domain objects; never construct a model by hand in a test when a factory exists.>
- <The database fixture rolls back per test; do not commit in a test.>

## 3. Idioms

> One bullet per framework idiom this repo uses. Skip anything `write-tests` already says.

- <`pytest.raises(AppError, match=...)` for errors; the match is the message, not the class.>
- <`@pytest.mark.parametrize(..., ids=[...])` when inputs share one expectation.>
- <`monkeypatch` at boundaries, `tmp_path` for files, `caplog` for logs. `respx` for `httpx` calls; never patch `httpx` directly.>
- <Async tests are plain `async def`; no `@pytest.mark.asyncio` (auto mode).>
- <Integration tests carry `@pytest.mark.integration` and use the `postgres` fixture from `tests/integration/conftest.py`.>

## 4. Finish

Run the tests you wrote, then <the fast suite: `uv run pytest -m "not integration" -q`>. Report the command alongside the result. <Run the slow suite only when you touched a module it covers.>
```
