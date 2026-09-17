---
name: review
description: Full pre-merge review of a Python project - runs the repo's own checks, then dispatches the reviewer, security, and simplifier agents in parallel and merges their findings into one prioritized list.
disable-model-invocation: true
argument-hint: "[base branch, defaults to origin/main]"
---

Review the uncommitted and unpushed work against $ARGUMENTS (default `origin/main`).

## 1. Scope

Run `git diff --stat --merge-base origin/main` (fall back to `git diff --stat HEAD`). If the diff is empty, say so and stop.
If it exceeds ~1500 changed lines, say it should be split and ask before continuing.

## 2. Checks

Run these first. Agents should not spend turns on what a linter catches.

Pick the runner from the lockfile as `write-tests-python` describes: `uv run`, `poetry run`, or nothing inside the active virtualenv. Every command below takes that prefix.

Run only the tools this repo already configures, and find them before running anything: read `pyproject.toml`, `setup.cfg`, `tox.ini`, `pre-commit-config.yaml`, and the CI workflow. Prefer a task the repo already defines (a `pre-commit` hook set, a `Makefile` target, a `[tool.poe]` or `nox` session) over calling each tool yourself - it is the definition of "the checks pass here".

Otherwise run whichever of these the repo configures:

| Check | Typical command |
|---|---|
| Lint | `ruff check .`, or the configured `flake8` / `pylint` |
| Format | `ruff format --check .`, or `black --check .` |
| Types | `pyright`, or `mypy` at the paths its config names |
| Fast tests | `pytest -m "not integration" -q`, adjusted to the repo's registered markers |
| Slow tests | `pytest -m integration -q` - skip if Docker or the fixture backend is unavailable, and say so |
| Dead code | `vulture src/ --min-confidence 80` |

Never install a tool to satisfy this list. Name every check you skipped and why - a skipped check is a finding, not a blank.

Report failures as a short list. Do not fix anything yet.

## 3. Agents

Dispatch all three in one message so they run in parallel:

- `code-reviewer` - correctness, conventions, missing tests
- `security-reviewer` - authz, injection, secrets, tenant scoping
- `simplifier` - what can be deleted

Give each the base branch in its prompt. Do not summarize the diff for them; they read it themselves.

These agents ship with the global config (`~/.claude/agents/`), not with this skill. If any is unavailable, do that pass yourself against the same criteria, and say which agent you stood in for - the findings still have to be produced.

## 4. Merge

One list, deduplicated, in this order:

1. **BLOCKER** - failing checks, plus CRITICAL/HIGH security findings and reviewer BLOCKERs
2. **SHOULD-FIX** - reviewer SHOULD-FIX, MEDIUM security, deletions over 20 lines
3. **CONSIDER** - nits, small deletions, LOW findings

Each item: `file:line - problem - fix`. Where two agents found the same thing, keep the more specific wording and cite both. Where they disagree, show both positions in one line and say which you would take.

Finish with a single line: how many blockers, and whether this is mergeable.

Then stop. Do not start fixing unless asked.
