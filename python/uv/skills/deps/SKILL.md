---
name: deps
description: Add, audit, or upgrade Python dependencies with uv - resolver-chosen versions, changelog check before upgrades, vulnerability scan. Use for any "add/update/upgrade this package" request, or when something may be outdated.
argument-hint: "add <pkg> | audit | upgrade [pkg]"
---

Dependency work for $ARGUMENTS. Pick the mode from the first word.

All changes go through `uv` so `pyproject.toml` and `uv.lock` stay consistent.
**Never write a version number from memory in any mode.** The resolver picks versions; you report what it picked. Never edit `pyproject.toml` or `uv.lock` by hand for dependency changes.

## Looking things up

The steps below ask the `docs-researcher` agent for the API at a resolved version and for breaking changes before an upgrade. That agent ships with the global config (`~/.claude/agents/`), not with this skill. If it is not available here, do the lookup yourself from the package's official docs and release notes, and say in your report that you did - never skip the lookup and never answer from memory.

## Release age

Never install a version released less than 7 days ago. Compromised packages are usually caught within days of publishing, so waiting keeps a hijacked release out of this repo.

Pass `--exclude-newer <date>` to every `uv add` and `uv lock` below, where `<date>` is 7 days ago in `YYYY-MM-DD`. Compute it with `date -d '7 days ago' +%F` (Linux), `date -v-7d +%F` (macOS), or `(Get-Date).AddDays(-7).ToString('yyyy-MM-dd')` (PowerShell). This filters transitive dependencies too. If the resolver picks an older version than the latest, say so and name the newer version that was skipped.

## add

1. Check it isn't already available: `uv pip list | grep -i <pkg>`, and grep the code for an existing helper or stdlib module that does the job. Say so if it's already covered.
2. Weigh it: last release date, maintenance, install size, transitive dependencies, whether the stdlib or an existing dependency already covers the need. A one-function dependency isn't worth it.
3. Install without a version (plus `--exclude-newer`, see Release age):
   - Runtime: `uv add <pkg>`
   - Dev-only tools (linters, test runners): `uv add --dev <pkg>`
   - Optional feature: `uv add --optional <extra> <pkg>`
4. Report the resolved version from `uv.lock`, then ask docs-researcher for the current API at that version before writing any code against it.

## audit

Run all of these and report as one table (`package · installed · latest · severity · note`):

- `uv tree --outdated --depth 1` for direct dependencies behind latest
- `uv run --with pip-audit pip-audit` for known vulnerabilities

Group as: security fixes (do now), majors behind (needs planning), minors/patches (batch).
For each security finding, name the fixed version and whether it's a major jump.
Don't upgrade anything in this mode - report only.

## upgrade

One package at a time, or one coherent group. Never all at once.

1. Current and target: `uv pip show <pkg>`, then the latest on PyPI.
2. **Before touching anything**, ask docs-researcher for breaking changes between the two versions. Summarize them for me and confirm before proceeding on a major bump.
3. Grep for every usage of the package in the repo. List the call sites that the breaking changes affect.
4. Upgrade (plus `--exclude-newer`, see Release age):
   - Within the existing constraint: `uv lock --upgrade-package <pkg>` then `uv sync`
   - Past the constraint (major bump): `uv add <pkg>` to re-resolve and rewrite the constraint
5. Fix call sites, then run the full check suite (lint, type check, tests).
6. Report: old -> new version, files changed, and anything you couldn't verify.

If the upgrade needs changes in more than ~5 files, stop and show me the plan first.
