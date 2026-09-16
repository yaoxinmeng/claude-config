---
name: deps
description: Add, audit, or upgrade Python dependencies with Poetry - resolver-chosen versions, changelog check before upgrades, vulnerability scan. Use for any "add/update/upgrade this package" request, or when something may be outdated.
argument-hint: "add <pkg> | audit | upgrade [pkg]"
---

Dependency work for $ARGUMENTS. Pick the mode from the first word.

All changes go through `poetry` so `pyproject.toml` and `poetry.lock` stay consistent.
**Never write a version number from memory in any mode.** The resolver picks versions; you report what it picked. Never edit `pyproject.toml` or `poetry.lock` by hand for dependency changes.

## add

1. Check it isn't already available: `poetry show | grep -i <pkg>`, and grep the code for an existing helper or stdlib module that does the job. Say so if it's already covered.
2. Weigh it: last release date, maintenance, install size, transitive dependencies, whether the stdlib or an existing dependency already covers the need. A one-function dependency isn't worth it.
3. Install without a version:
   - Runtime: `poetry add <pkg>`
   - Dev-only tools (linters, test runners): `poetry add --group dev <pkg>`
   - Optional feature: `poetry add --optional <extra> <pkg>`
4. Report the resolved version from `poetry show <pkg>`, then ask docs-researcher for the current API at that version before writing any code against it.

## audit

Run all of these and report as one table (`package · installed · latest · severity · note`):

- `poetry show --outdated --top-level` for direct dependencies behind latest
- `poetry run pip install pip-audit && poetry run pip-audit` for known vulnerabilities, then `poetry sync` to drop pip-audit from the environment again. pip-audit is a tool, not a dependency: don't `poetry add` it.

Group as: security fixes (do now), majors behind (needs planning), minors/patches (batch).
For each security finding, name the fixed version and whether it's a major jump.
Don't upgrade anything in this mode - report only.

## upgrade

One package at a time, or one coherent group. Never all at once.

1. Current and target: `poetry show <pkg>` (prints installed and latest).
2. **Before touching anything**, ask docs-researcher for breaking changes between the two versions. Summarize them for me and confirm before proceeding on a major bump.
3. Grep for every usage of the package in the repo. List the call sites that the breaking changes affect.
4. Upgrade:
   - Within the existing constraint: `poetry update <pkg>`
   - Past the constraint (major bump): `poetry add <pkg>@latest` to re-resolve and rewrite the constraint
5. Fix call sites, then run the full check suite (lint, type check, tests).
6. Report: old -> new version, files changed, and anything you couldn't verify.

If the upgrade needs changes in more than ~5 files, stop and show me the plan first.
