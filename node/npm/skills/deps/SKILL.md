---
name: deps
description: Add, audit, or upgrade Node dependencies with npm - resolver-chosen versions, changelog check before upgrades, vulnerability scan. Use for any "add/update/upgrade this package" request, or when something may be outdated.
argument-hint: "add <pkg> | audit | upgrade [pkg]"
---

Dependency work for $ARGUMENTS. Pick the mode from the first word.

All changes go through `npm` so `package.json` and `package-lock.json` stay consistent.
**Never write a version number from memory in any mode.** The resolver picks versions; you report what it picked. Never edit `package.json` or `package-lock.json` by hand for dependency changes.

## Release age

Never install a version released less than 7 days ago. Compromised packages are usually caught within days of publishing, so waiting keeps a hijacked release out of this repo.

Pass `--before=<date>` to every `npm install` and `npm update` below, where `<date>` is 7 days ago in `YYYY-MM-DD`. Compute it with `date -d '7 days ago' +%F` (Linux), `date -v-7d +%F` (macOS), or `(Get-Date).AddDays(-7).ToString('yyyy-MM-dd')` (PowerShell). This filters transitive dependencies too. If npm picks an older version than the latest, say so and name the newer version that was skipped (`npm view <pkg> time` lists release dates).

## add

1. Check it isn't already available: `npm ls <pkg> --depth=0`, and grep the code for an existing helper or built-in Node module that does the job. Say so if it's already covered.
2. Weigh it: last release date, maintenance, install size (`npm view <pkg> dist.unpackedSize`), transitive dependencies, whether Node built-ins or an existing dependency already cover the need. A one-function dependency isn't worth it.
3. Install without a version (plus `--before`, see Release age):
   - Runtime: `npm install <pkg>`
   - Dev-only tools (linters, test runners, build tools, type definitions): `npm install --save-dev <pkg>`
   - Peer dependency of a library this repo publishes: `npm install --save-peer <pkg>`
4. Report the resolved version from `npm ls <pkg> --depth=0`, then ask docs-researcher for the current API at that version before writing any code against it.

## audit

Run all of these and report as one table (`package · installed · latest · severity · note`):

- `npm outdated` for direct dependencies behind their latest (the `Wanted` column is what the current range allows, `Latest` is the newest release)
- `npm audit` for known vulnerabilities

Group as: security fixes (do now), majors behind (needs planning), minors/patches (batch).
For each security finding, name the fixed version and whether it's a major jump.
Don't upgrade anything in this mode - report only. In particular, never run `npm audit fix --force`.

## upgrade

One package at a time, or one coherent group. Never all at once.

1. Current and target: `npm ls <pkg> --depth=0`, then `npm view <pkg> version` for the latest.
2. **Before touching anything**, ask docs-researcher for breaking changes between the two versions. Summarize them for me and confirm before proceeding on a major bump.
3. Grep for every usage of the package in the repo. List the call sites that the breaking changes affect.
4. Upgrade (plus `--before`, see Release age):
   - Within the existing range: `npm update <pkg>`
   - Past the range (major bump): `npm install <pkg>@latest` to re-resolve and rewrite the range
   If the package has a matching `@types/<pkg>` dev dependency, upgrade it in the same step.
5. Fix call sites, then run the full check suite (lint, type check, tests).
6. Report: old -> new version, files changed, and anything you couldn't verify.

If the upgrade needs changes in more than ~5 files, stop and show me the plan first.
