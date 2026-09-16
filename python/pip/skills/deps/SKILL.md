---
name: deps
description: Add, audit, or upgrade Python dependencies with pip and requirements files - installer-chosen versions, changelog check before upgrades, vulnerability scan. Use for any "add/update/upgrade this package" request, or when something may be outdated.
argument-hint: "add <pkg> | audit | upgrade [pkg]"
---

Dependency work for $ARGUMENTS. Pick the mode from the first word.

All installs go through `pip` inside the project's virtualenv (activate it, or use `python -m pip`). pip has no lockfile, so the requirements files are the record: every runtime dependency lives in `requirements.txt`, dev tools in `requirements-dev.txt` (or whatever the repo already uses - check before creating a new file).
**Never write a version number from memory in any mode.** pip picks the version; you copy what `pip show` reports into the requirements file. Never edit a version pin by hand for any other reason.

## add

1. Check it isn't already available: `pip list | grep -i <pkg>`, grep the requirements files, and grep the code for an existing helper or stdlib module that does the job. Say so if it's already covered.
2. Weigh it: last release date, maintenance, install size, transitive dependencies, whether the stdlib or an existing dependency already covers the need. A one-function dependency isn't worth it.
3. Install without a version: `pip install <pkg>`.
4. Read the resolved version with `pip show <pkg>` and add `<pkg>==<that version>` to the right requirements file (runtime or dev), keeping the file's existing ordering. If the file pins with `>=` instead of `==`, follow the file's convention.
5. Report the resolved version, then ask docs-researcher for the current API at that version before writing any code against it.

## audit

Run all of these and report as one table (`package · installed · latest · severity · note`):

- `pip list --outdated` for packages behind latest, then filter it to the ones named in the requirements files (direct dependencies)
- `pip install pip-audit && pip-audit` for known vulnerabilities in the installed environment. pip-audit is a tool, not a dependency: don't add it to any requirements file.

Group as: security fixes (do now), majors behind (needs planning), minors/patches (batch).
For each security finding, name the fixed version and whether it's a major jump.
Don't upgrade anything in this mode - report only.

## upgrade

One package at a time, or one coherent group. Never all at once.

1. Current and target: `pip show <pkg>`, then the latest on PyPI (`pip index versions <pkg>`).
2. **Before touching anything**, ask docs-researcher for breaking changes between the two versions. Summarize them for me and confirm before proceeding on a major bump.
3. Grep for every usage of the package in the repo. List the call sites that the breaking changes affect.
4. Upgrade:
   - Within the existing constraint (the requirements line has an upper bound you want to keep): `pip install --upgrade "<pkg><upper bound>"`
   - Past the constraint (major bump): `pip install --upgrade <pkg>`
   Then copy the version from `pip show <pkg>` into the requirements line.
5. Reinstall from the files to prove they're consistent: `pip install -r requirements.txt -r requirements-dev.txt`.
6. Fix call sites, then run the full check suite (lint, type check, tests).
7. Report: old -> new version, files changed, and anything you couldn't verify.

If the upgrade needs changes in more than ~5 files, stop and show me the plan first.
