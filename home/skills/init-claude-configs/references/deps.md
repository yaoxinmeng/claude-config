# Template: `.claude/skills/deps/SKILL.md`

The rules are the same for every package manager; only the commands differ. Fill every `<...>` from the manager table at the end of this file, or, for a manager not in the table, from its documentation - and verify each command in the repo before writing it, as `SKILL.md` step 3 says. A repo with several package managers gets one `## <component>` block per manager inside each mode, not one skill per manager. Lines in `> ` are instructions to you, not content; remove them.

```markdown
---
name: deps
description: Add, audit, or upgrade <language> dependencies with <manager> - resolver-chosen versions, changelog check before upgrades, vulnerability scan. Use for any "add/update/upgrade this package" request, or when something may be outdated.
argument-hint: "add <pkg> | audit | upgrade [pkg]"
---

Dependency work for $ARGUMENTS. Pick the mode from the first word.

All changes go through <`uv`> so <`pyproject.toml` and `uv.lock`> stay consistent.
**Never write a version number from memory in any mode.** The resolver picks versions; you report what it picked. Never edit <`pyproject.toml` or `uv.lock`> by hand for dependency changes.

> pip has no lockfile: say instead that the requirements files are the record, that you copy what `pip show` reports into them, and never edit a pin by hand for any other reason.

## Looking things up

The steps below ask the `docs-researcher` agent for the API at a resolved version and for breaking changes before an upgrade. It lives in this repo under `.claude/agents/`, copied there from the global config so this skill works on any machine. If it is missing, do the lookup yourself from the package's official docs and release notes, and say in your report that you did - never skip the lookup and never answer from memory.

## Release age

Never install a version released less than 7 days ago. Compromised packages are usually caught within days of publishing, so waiting keeps a hijacked release out of this repo.

> Managers with a native date filter (uv, npm): the paragraph below. Others: the manual check that follows it.

Pass <`--exclude-newer <date>`> to every <`uv add` and `uv lock`> below, where `<date>` is 7 days ago in `YYYY-MM-DD`. Compute it with `date -d '7 days ago' +%F` (Linux), `date -v-7d +%F` (macOS), or `(Get-Date).AddDays(-7).ToString('yyyy-MM-dd')` (PowerShell). This filters transitive dependencies too. If the resolver picks an older version than the latest, say so and name the newer version that was skipped.

<Manager> has no date filter, so check by hand after every install or upgrade below:

1. <`poetry show <pkg>`> for the version the resolver picked.
2. <`curl -s https://pypi.org/pypi/<pkg>/json` and read `releases["<version>"][0].upload_time`.>
3. If that is less than 7 days ago, find the newest release that is at least 7 days old and pin it: <`poetry add "<pkg>@<that version>"` with the same group flags>. Say so in the report and name the version that was skipped, so the pin can be loosened on the next upgrade.

This only covers the package you asked for. Transitive dependencies are not checked; mention that in the report when the package pulls in new ones.

## add

1. Check it isn't already available: <`uv pip list | grep -i <pkg>`>, and grep the code for an existing helper or <standard-library> module that does the job. Say so if it's already covered.
2. Weigh it: last release date, maintenance, install size, transitive dependencies, whether the standard library or an existing dependency already covers the need. A one-function dependency isn't worth it.
3. Install without a version (plus the release-age flag or check above):
   - Runtime: <`uv add <pkg>`>
   - Dev-only tools (linters, test runners, build tools, type definitions): <`uv add --dev <pkg>`>
   - <Optional feature: `uv add --optional <extra> <pkg>`>
4. Report the resolved version from <`uv.lock`>, then ask docs-researcher for the current API at that version before writing any code against it.

## audit

Run all of these and report as one table (`package · installed · latest · severity · note`):

- <`uv tree --outdated --depth 1`> for direct dependencies behind latest
- <`uv run --with pip-audit pip-audit`> for known vulnerabilities. <pip-audit is a tool, not a dependency: never add it to the manifest.>

Group as: security fixes (do now), majors behind (needs planning), minors/patches (batch).
For each security finding, name the fixed version and whether it's a major jump.
Don't upgrade anything in this mode - report only. <In particular, never run `npm audit fix --force`.>

## upgrade

One package at a time, or one coherent group. Never all at once.

1. Current and target: <`uv pip show <pkg>`, then the latest on PyPI>.
2. **Before touching anything**, ask docs-researcher for breaking changes between the two versions. Summarize them for me and confirm before proceeding on a major bump.
3. Grep for every usage of the package in the repo. List the call sites that the breaking changes affect.
4. Upgrade (plus the release-age flag or check above):
   - Within the existing constraint: <`uv lock --upgrade-package <pkg>` then `uv sync`>
   - Past the constraint (major bump): <`uv add <pkg>` to re-resolve and rewrite the constraint>
   <If the package has a matching `@types/<pkg>` dev dependency, upgrade it in the same step.>
5. Fix call sites, then run every row of the Checks table in the `coding` skill.
6. Report: old -> new version, files changed, and anything you couldn't verify.

If the upgrade needs changes in more than ~5 files, stop and show me the plan first.
```

## Manager table

Verified command sets for the managers this config has shipped before. For any other manager (`pnpm`, `yarn`, `cargo`, `go`, `bundler`, `maven`, `gradle`, `dotnet`, ...) look up the equivalents in its documentation, run each one in the repo, and add a row to the generated skill's report saying the set is new.

| | uv | poetry | pip | npm |
|---|---|---|---|---|
| Keeps consistent | `pyproject.toml` + `uv.lock` | `pyproject.toml` + `poetry.lock` | `requirements.txt` (+ `requirements-dev.txt`, or whatever the repo has) | `package.json` + `package-lock.json` |
| Release-age | native: `--exclude-newer <date>` on `uv add` / `uv lock` | manual, via PyPI JSON; pin with `poetry add "<pkg>@<version>"` | manual, via PyPI JSON; install `pip install "<pkg>==<version>"` | native: `--before=<date>` on `npm install` / `npm update`; `npm view <pkg> time` lists release dates |
| Already installed? | `uv pip list \| grep -i <pkg>` | `poetry show \| grep -i <pkg>` | `pip list \| grep -i <pkg>` plus grep the requirements files | `npm ls <pkg> --depth=0` |
| Add runtime | `uv add <pkg>` | `poetry add <pkg>` | `pip install <pkg>`, then `<pkg>==<pip show version>` into `requirements.txt` (follow the file's `==` / `>=` convention) | `npm install <pkg>` |
| Add dev | `uv add --dev <pkg>` | `poetry add --group dev <pkg>` | same, into `requirements-dev.txt` | `npm install --save-dev <pkg>` |
| Add optional / peer | `uv add --optional <extra> <pkg>` | `poetry add --optional <extra> <pkg>` | - | `npm install --save-peer <pkg>` (libraries only) |
| Resolved version | `uv.lock` | `poetry show <pkg>` | `pip show <pkg>` | `npm ls <pkg> --depth=0` |
| Outdated | `uv tree --outdated --depth 1` | `poetry show --outdated --top-level` | `pip list --outdated`, filtered to names in the requirements files | `npm outdated` (`Wanted` = allowed by range, `Latest` = newest) |
| Vulnerabilities | `uv run --with pip-audit pip-audit` | `poetry run pip install pip-audit && poetry run pip-audit`, then `poetry sync` to drop it again | `pip install pip-audit && pip-audit`; never add it to a requirements file | `npm audit`; never `npm audit fix --force` |
| Latest version | PyPI | `poetry show <pkg>` prints installed and latest | `pip index versions <pkg>` | `npm view <pkg> version` |
| Upgrade within constraint | `uv lock --upgrade-package <pkg>` then `uv sync` | `poetry update <pkg>` | `pip install --upgrade "<pkg><upper bound>"` | `npm update <pkg>` |
| Upgrade past constraint | `uv add <pkg>` | `poetry add <pkg>@latest` | `pip install --upgrade <pkg>` | `npm install <pkg>@latest`; upgrade a matching `@types/<pkg>` in the same step |
| After upgrade | - | - | copy the version into the requirements line, then `pip install -r requirements.txt -r requirements-dev.txt` to prove the files are consistent | - |
| Install size | - | - | - | `npm view <pkg> dist.unpackedSize` |
