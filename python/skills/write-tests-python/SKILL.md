---
name: write-tests-python
description: Pytest specifics on top of the shared test standards - picking the runner from the lockfile, reusing conftest fixtures, and the pytest idioms for errors, parameterised cases, and boundary patching. Load before writing or editing any Python test.
---

The `write-tests` skill sets what to test and how to assert it; load that first. Everything below is what pytest adds on top. If the repo's own conventions (CLAUDE.md, existing tests, `conftest.py`) contradict anything here, follow the repo.

## 1. Pick the runner

- `uv.lock` means `uv run pytest`. `poetry.lock` means `poetry run pytest`. Otherwise `pytest` inside the active virtualenv.
- Read `pyproject.toml`, `pytest.ini`, and `setup.cfg` for `[tool.pytest.ini_options]`, registered markers, and plugins already in force.
- Never add a plugin or dependency to make a test possible. If a test needs one, report that instead of installing it.

## 2. Find the layout and the fixtures

- Tests live either in a `tests/` tree mirroring the package or in `test_*.py` beside the module. Copy whichever the repo already uses; never introduce a second layout.
- Read every `conftest.py` on the path from the repo root to the target, and reuse those fixtures before defining a new one.

## 3. Pytest idioms

- `pytest.raises(SpecificError, match=...)` for errors, never a bare `Exception`.
- `@pytest.mark.parametrize` with `ids=` when several inputs share one expectation. Separate tests when the expectations differ.
- `monkeypatch` for boundary patching, `tmp_path` for files, `caplog` for log assertions. Never patch the module under test.
- Fixtures return values; they don't assert. A fixture that asserts hides which test failed.
- Mark slow or integration tests with the marker the repo already registers, never a new one.

## 4. Finish

Run the tests you wrote, then the whole suite with the runner from step 1. Report the command you used alongside the result.
