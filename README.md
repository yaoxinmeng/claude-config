# claude-config

Shared Claude Code configuration: global instructions, skills, and agents. This repo is the source of truth; `~/.claude/` is a copy of it.

## Layout

| Directory | Installs to | Contents |
|---|---|---|
| `home/` | `~/.claude/` | Global `CLAUDE.md`, skills and agents used in every project |
| `python/<tool>/` | `<project>/.claude/` | Skills that only make sense in a Python project, one directory per package manager: `uv`, `pip`, `poetry` |

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

Language-specific skills (`python/`) are copied into the project instead. Pick the directory matching the project's package manager and copy its `skills/` into `<project>/.claude/`:

```powershell
Copy-Item -Recurse -Force python\uv\skills <project>\.claude\
```

```sh
cp -r python/uv/skills <project>/.claude/
```

Replace `uv` with `pip` or `poetry` as needed. Copy exactly one: all three define a `/deps` skill, so the last one copied would win.

## What is in `home/`

### `CLAUDE.md`

Global rules for every session: no em dashes, no agent co-author lines, favour quality over development cost, reproduce bugs end-to-end before fixing, fix any lint or test failure you see, and load the `concise` skill before replying.

### Skills

| Skill | Invoke | What it does |
|---|---|---|
| `concise` | Loaded automatically by `CLAUDE.md` | Rules for every reply: lead with the answer, gloss jargon on first use, keep task reports to a paragraph plus bullets, and make every `AskUserQuestion` option concrete and tweakable. |
| `init-project` | `/init-project <one-line description>` (user only) | Requirements interview in five rounds (product, architecture, design, stack, infrastructure), then writes `docs/spec/*.md`, ADRs under `docs/decisions/`, and a `## Project spec` section in the project `CLAUDE.md` so future sessions find the spec. Never writes code. |

### Agents

All agents are read-only except `docs-writer`. Claude picks them from their descriptions; you can also ask for one by name.

| Agent | Model | What it does |
|---|---|---|
| `code-reviewer` | opus | Reviews the uncommitted diff against `origin/main`. Reports BLOCKER / SHOULD-FIX / NIT as `file:line - problem - fix`, or `LGTM`. Keeps per-project memory of recurring issues. Preloads `python-backend`, `nextjs-frontend`, `aws-cdk`, and `testing` skills if the project defines them. |
| `security-reviewer` | opus | Audits the diff for authz gaps, injection, leaked secrets, over-broad IAM, risky dependencies. Reports only what the diff introduces, ordered by severity. |
| `simplifier` | sonnet | Finds code the diff added that can be deleted or inlined: single-use wrappers, impossible guards, orphans, tests that cannot fail. Only proposes changes that reduce line count. |
| `docs-writer` | sonnet | The only agent that edits files, and only `README.md` and `docs/`. Keeps `docs/spec/` current, writes how-tos, explanations, and ADRs. Runs every snippet before writing it. |
| `docs-researcher` | haiku | Answers "how do I call this API at the version this project has installed". Checks `uv.lock` / `npm ls` first, then Context7, then official docs. Returns version, snippet, gotchas, source. |

## What is in `python/`

Each of `python/uv/`, `python/pip/`, and `python/poetry/` holds the same `deps` skill written for that package manager. The commands differ; the rules do not.

| Skill | Invoke | What it does |
|---|---|---|
| `deps` | `/deps add <pkg>`, `/deps audit`, `/deps upgrade [pkg]` | Dependency changes through the package manager only, never hand-written versions. `add` checks the need first and reports the resolved version; `audit` reports outdated and vulnerable packages without changing anything; `upgrade` asks `docs-researcher` for breaking changes before touching one package at a time. |

| Variant | Keeps consistent | Notes |
|---|---|---|
| `uv` | `pyproject.toml` + `uv.lock` | `uv add`, `uv tree --outdated`, `uv lock --upgrade-package` |
| `poetry` | `pyproject.toml` + `poetry.lock` | `poetry add`, `poetry show --outdated`, `poetry update` |
| `pip` | `requirements.txt` (+ `requirements-dev.txt`) | No lockfile, so the skill pins the version `pip show` reports after each install. Uses whatever requirements files the repo already has. |

All three run `pip-audit` for vulnerabilities. It is installed into the environment as a tool, never added as a dependency.

## Greenfield project

1. Create the repo and `cd` into it.
2. Run `/init-project <one-line description>`. Answer the interview; say "you decide" for anything you do not care about and it is recorded in `docs/spec/assumptions.md`. Stop early if you must; unresolved items go to `docs/spec/open-questions.md`.
3. Confirm the final summary. The skill writes `docs/spec/`, `docs/decisions/`, and a `CLAUDE.md` pointing at them, then asks whether to commit.
4. Copy the language skills for your package manager into the project, for example `cp -r python/uv/skills <project>/.claude/` (see Setup).
5. Optionally add project skills named `python-backend`, `nextjs-frontend`, `aws-cdk`, or `testing` under `<project>/.claude/skills/`; `code-reviewer` treats violations of those as SHOULD-FIX.
6. Build from the spec. When a design changes, update the spec file in the same commit; `docs-writer` can do this.

## Brownfield project

1. `cd` into the existing repo. If it already has a `CLAUDE.md`, keep it; `init-project` only adds or replaces the `## Project spec` section.
2. Run `/init-project <one-line description>`. It reads the README, manifests, entry points, and data models first and only asks about what the code does not answer. Where the code and your answer disagree, it reports the gap rather than papering over it.
3. Copy the language skills into the project as in the greenfield steps, choosing `uv`, `pip`, or `poetry` to match what the repo already uses (`uv.lock`, `requirements.txt`, or `poetry.lock`).
4. Run `/deps audit` for a first picture of outdated or vulnerable dependencies. Plan majors separately; do security fixes first.
5. Before the first non-trivial change, ask for the `code-reviewer` and `security-reviewer` agents on the diff so their project memory starts with the repo's existing conventions.
