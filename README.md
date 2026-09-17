# claude-config

Shared Claude Code configuration: global instructions, skills, and agents. This repo is the source of truth; `~/.claude/` is a copy of it.

## Layout

| Directory | Installs to | Contents |
|---|---|---|
| `home/` | `~/.claude/` | Global `CLAUDE.md`, skills and agents used in every project |
| `python/<tool>/` | `<project>/.claude/` | Skills that only make sense in a Python project, one directory per package manager: `uv`, `pip`, `poetry` |
| `python/skills/` | `<project>/.claude/skills/` | Skills for any Python project, whatever the package manager |
| `python/agents/` | `<project>/.claude/agents/` | Agents for any Python project, whatever the package manager |
| `node/npm/` | `<project>/.claude/` | Skills that only make sense in a Node project using `npm` |
| `infra/` | `<project>/.claude/` | Skills for projects that define cloud infrastructure, whatever the application language |

## Setup

1. Copy `home/` over `~/.claude/`:

   ```powershell
   Copy-Item -Recurse -Force home\* $HOME\.claude\
   ```

   ```sh
   cp -r home/. ~/.claude/
   ```

2. `docs-researcher` runs the Context7 MCP server through `npx`, so Node.js must be on `PATH`.
3. Restart Claude Code. `/init-project` and `/deps` should appear in the slash-command list and the agents under the Agent tool.
4. After editing anything here, re-run step 1. Nothing syncs automatically.

Language-specific skills (`python/`, `node/`) are copied into the project instead. Pick the directory matching the project's package manager and copy its `skills/` into `<project>/.claude/`:

```powershell
Copy-Item -Recurse -Force python\uv\skills <project>\.claude\
```

```sh
cp -r python/uv/skills <project>/.claude/
```

Replace `python/uv` with `python/pip`, `python/poetry`, or `node/npm` as needed. Copy exactly one: every variant defines a `/deps` skill, so the last one copied would win.

Python projects also get the shared skills and agents, regardless of package manager:

```powershell
Copy-Item -Recurse -Force python\skills, python\agents <project>\.claude\
```

```sh
cp -r python/skills python/agents <project>/.claude/
```

Projects that define cloud infrastructure also get the `infra/` skills:

```powershell
Copy-Item -Recurse -Force infra\skills <project>\.claude\
```

```sh
cp -r infra/skills <project>/.claude/
```

## What is in `home/`

### `CLAUDE.md`

Global rules for every session: no em dashes, no agent co-author lines, favour quality over development cost, reproduce bugs end-to-end before fixing, fix any lint or test failure you see, load the `concise` skill before replying, load the `write-code` skill before writing code, load the `write-tests` skill before writing tests, and load the `write-docs` skill before writing documentation.

### Skills

| Skill | Invoke | What it does |
|---|---|---|
| `concise` | Loaded automatically by `CLAUDE.md` | Rules for every reply: lead with the answer, gloss jargon on first use, keep task reports to a paragraph plus bullets, and make every `AskUserQuestion` option concrete and tweakable. |
| `write-code` | Loaded automatically by `CLAUDE.md` before any code is written | Coding standards in six steps: find existing code and the right file location first, DRY with judgement, readable names and small functions, abstractions only with two real uses, validate at the boundary and test every behaviour, then lint, re-read the diff, and ask `simplifier` when the diff outgrows the problem. |
| `write-tests` | Loaded before writing or editing any test; preloaded by `test-writer` | Test standards for any language: find the runner and existing fixtures first, one behaviour per test, assertions that can actually fail, names that state the expected result, mocking only at boundaries, no sleeps or ordering between tests, and a failing test left in place when it exposes a real bug. |
| `write-docs` | Loaded automatically by `CLAUDE.md` before any documentation is written; preloaded by `docs-writer` | House style for prose docs: lead with the answer, second person and present tense, every claim concrete, say why rather than what, no marketing words, own the caveats. Defines one purpose per page type (`docs/spec/`, `docs/how-to/`, `docs/explanation/`, `docs/decisions/`, `README.md`) and the never list: no documenting private functions, no hand-editing generated `docs/reference/`, no TODOs left in a page. |
| `init-project` | `/init-project <one-line description>` (user only) | Requirements interview in five rounds (product, architecture, design, stack, infrastructure), then writes `docs/spec/*.md`, ADRs under `docs/decisions/`, and a `## Project spec` section in the project `CLAUDE.md` so future sessions find the spec. Never writes code. |

### Agents

All agents are read-only except `docs-writer`. Claude picks them from their descriptions; you can also ask for one by name.

| Agent | Model | What it does |
|---|---|---|
| `code-reviewer` | opus | Reviews the uncommitted diff against `origin/main`. Reports BLOCKER / SHOULD-FIX / NIT as `file:line - problem - fix`, or `LGTM`. Keeps per-project memory of recurring issues. Preloads the `write-code` and `write-tests` skills. Judges tests in the diff against `write-tests`, plus the diff-only checks: behaviour changed with no test, assertions loosened instead of updated, new branches left uncovered, tests deleted while their behaviour stayed. |
| `security-reviewer` | opus | Audits the diff for authz gaps, injection, leaked secrets, over-broad IAM, risky dependencies. Reports only what the diff introduces, ordered by severity. |
| `simplifier` | sonnet | Finds code the diff added that can be deleted or inlined: single-use wrappers, impossible guards, orphans, tests that cannot fail. Only proposes changes that reduce line count. |
| `docs-writer` | sonnet | The only agent that edits files, and only `README.md` and `docs/`. Keeps `docs/spec/` current, writes how-tos, explanations, and ADRs. Runs every snippet before writing it. Preloads the `write-docs` skill, which holds the house style and page types; the agent itself only adds the delegation rules - establish the subject from the repo rather than the prompt, and report gaps instead of guessing. |
| `docs-researcher` | haiku | Answers "how do I call this API at the version this project has installed". Checks `uv.lock` / `npm ls` first, then Context7, then official docs. Returns version, snippet, gotchas, source. |

## What is in `python/` and `node/`

### Skills

Each of `python/uv/`, `python/pip/`, `python/poetry/`, and `node/npm/` holds the same `deps` skill written for that package manager. The commands differ; the rules do not.

| Skill | Invoke | What it does |
|---|---|---|
| `deps` | `/deps add <pkg>`, `/deps audit`, `/deps upgrade [pkg]` | Dependency changes through the package manager only, never hand-written versions. `add` checks the need first and reports the resolved version; `audit` reports outdated and vulnerable packages without changing anything; `upgrade` asks `docs-researcher` for breaking changes before touching one package at a time. |

| Variant | Keeps consistent | Notes |
|---|---|---|
| `uv` | `pyproject.toml` + `uv.lock` | `uv add`, `uv tree --outdated`, `uv lock --upgrade-package` |
| `poetry` | `pyproject.toml` + `poetry.lock` | `poetry add`, `poetry show --outdated`, `poetry update` |
| `pip` | `requirements.txt` (+ `requirements-dev.txt`) | No lockfile, so the skill pins the version `pip show` reports after each install. Uses whatever requirements files the repo already has. |
| `npm` | `package.json` + `package-lock.json` | `npm install`, `npm outdated`, `npm update` / `npm install <pkg>@latest`; upgrades a matching `@types/` package in the same step. Never runs `npm audit fix --force`. |

Every variant refuses versions released less than 7 days ago, so a hijacked release has time to be caught before it lands in a project. `uv` (`--exclude-newer`) and `npm` (`--before`) enforce this natively, transitive dependencies included; `pip` and `poetry` check the direct package against PyPI's release dates and pin the newest old-enough version instead.

The Python variants run `pip-audit` for vulnerabilities, installed into the environment as a tool and never added as a dependency. The npm variant uses the built-in `npm audit`.

Every variant asks `docs-researcher` for the API at a resolved version and for breaking changes before an upgrade. That agent lives in `home/`, so a project that only got the skills copied in does not have it; each skill says to do the lookup by hand and report that it did, rather than answering from memory. The `aws-cdk` skill depends on it the same way.

`python/skills/` holds one skill that applies to any Python project, whatever the package manager:

| Skill | Invoke | What it does |
|---|---|---|
| `write-tests-python` | Loaded before writing or editing any Python test; preloaded by `test-writer` | The pytest layer on top of `write-tests`: pick the runner from the lockfile (`uv run pytest`, `poetry run pytest`, or plain `pytest`), read `[tool.pytest.ini_options]` for markers and plugins, never add a plugin to make a test possible, reuse every `conftest.py` on the path, and the idioms - `pytest.raises(..., match=...)` over bare `Exception`, `parametrize` with `ids=`, `monkeypatch` and `tmp_path` at boundaries, fixtures that return rather than assert. |

### Agents

| Agent | Model | What it does |
|---|---|---|
| `test-writer` | opus | Writes pytest unit tests for new or untested code, editing only the test directory. Preloads `write-tests` and `write-tests-python`, which hold the standards; the agent itself only adds the delegation rules - read the code under test and its callers in full, run the suite, and report what could not be tested through the public interface. Requires `python/skills/` to be installed in the project alongside `python/agents/`. |

## What is in `infra/`

Skills for projects that define cloud infrastructure. Copy `infra/skills` into `<project>/.claude/` only when the repo has infrastructure code; they are not global.

| Skill | Invoke | What it does |
|---|---|---|
| `aws-cdk` | Loaded before touching CDK code | Conventions for AWS CDK in TypeScript: stack layout and props over globals, L2 constructs over `Cfn*`, generated physical names, grant methods over wildcard IAM, `RemovalPolicy.RETAIN` on stateful resources, and the `tsc` / `cdk synth` / `cdk diff` / `assertions` loop that has to pass. Never deploys or destroys on its own. |

## Greenfield project

1. Create the repo and `cd` into it.
2. Run `/init-project <one-line description>`. Answer the interview; say "you decide" for anything you do not care about and it is recorded in `docs/spec/assumptions.md`. Stop early if you must; unresolved items go to `docs/spec/open-questions.md`.
3. Confirm the final summary. The skill writes `docs/spec/`, `docs/decisions/`, and a `CLAUDE.md` pointing at them, then asks whether to commit.
4. Copy the language skills for your package manager into the project, for example `cp -r python/uv/skills <project>/.claude/`, plus `python/skills` and `python/agents` for a Python project (see Setup). Add `cp -r infra/skills <project>/.claude/` if the project defines cloud infrastructure.
5. Build from the spec. When a design changes, update the spec file in the same commit; `docs-writer` can do this.

## Brownfield project

1. `cd` into the existing repo. If it already has a `CLAUDE.md`, keep it; `init-project` only adds or replaces the `## Project spec` section.
2. Run `/init-project <one-line description>`. It reads the README, manifests, entry points, and data models first and only asks about what the code does not answer. Where the code and your answer disagree, it reports the gap rather than papering over it.
3. Copy the language skills and agents into the project as in the greenfield steps, choosing the variant that matches what the repo already uses (`uv.lock`, `requirements.txt`, `poetry.lock`, or `package-lock.json`). Add the `infra/` skills if the repo already has CDK code.
4. Run `/deps audit` for a first picture of outdated or vulnerable dependencies. Plan majors separately; do security fixes first.
5. Before the first non-trivial change, ask for the `code-reviewer` and `security-reviewer` agents on the diff so their project memory starts with the repo's existing conventions.
