# claude-config

Shared Claude Code configuration: global instructions, skills, and agents. This repo is the source of truth; `~/.claude/` is a copy of it.

## Layout

| Directory | Installs to | Contents |
|---|---|---|
| `home/` | `~/.claude/` | Global `CLAUDE.md`, skills and agents used in every project |
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
3. Restart Claude Code. `/init-project` and `/init-claude-configs` should appear in the slash-command list and the agents under the Agent tool.
4. After editing anything here, re-run step 1. Nothing syncs automatically.

Project-level skills and agents (`coding`, `testing`, `review`, `deps`, `coder`, `test-writer`) are not copied; `/init-claude-configs` generates them for the project's own stack (see below).

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

The generated project skills layer on these: `coding` on `write-code`, `testing` on `write-tests`. The global skills hold what is true in every repo; the project skills hold only what is specific to one.

### Skills

| Skill | Invoke | What it does |
|---|---|---|
| `concise` | Loaded automatically by `CLAUDE.md` | Rules for every reply: lead with the answer, gloss jargon on first use, keep task reports to a paragraph plus bullets, and make every `AskUserQuestion` option concrete and tweakable. |
| `write-code` | Loaded automatically by `CLAUDE.md` before any code is written | Coding standards in six steps: find existing code and the right file location first, DRY with judgement, readable names and small functions, abstractions only with two real uses, validate at the boundary and test every behaviour, then lint, re-read the diff, and ask `simplifier` when the diff outgrows the problem. |
| `write-tests` | Loaded before writing or editing any test; preloaded by `test-writer` | Test standards for any language: find the runner and existing fixtures first, one behaviour per test, assertions that can actually fail, names that state the expected result, mocking only at boundaries, no sleeps or ordering between tests, and a failing test left in place when it exposes a real bug. |
| `write-docs` | Loaded automatically by `CLAUDE.md` before any documentation is written; preloaded by `docs-writer` | House style for prose docs: lead with the answer, second person and present tense, every claim concrete, say why rather than what, no marketing words, own the caveats. Defines one purpose per page type (`docs/spec/`, `docs/how-to/`, `docs/explanation/`, `docs/decisions/`, `README.md`) and the never list: no documenting private functions, no hand-editing generated `docs/reference/`, no TODOs left in a page. |
| `init-project` | `/init-project <one-line description>` (user only) | Requirements interview in five rounds (product, architecture, design, stack, infrastructure), then writes `docs/spec/*.md`, ADRs under `docs/decisions/`, and a `## Project spec` section in the project `CLAUDE.md` so future sessions find the spec, then points to `/init-claude-configs`. Never writes code. |
| `init-claude-configs` | `/init-claude-configs [directory]` (user only) | Generates the project's `.claude/` for its own stack. Reads manifests, lockfiles, tool config, CI, tests, and representative source into a fact sheet; interviews for what the code cannot answer (everything, on a greenfield repo); runs every check command before writing it down; then writes the `coding`, `testing`, `review`, and `deps` skills and the `coder` and `test-writer` agents from the templates in `references/`, a `.claude/settings.json` that allows edits and verified check commands in the project, allows `git` branch/checkout/commit, asks before `git push`, denies reading `.env` files and private keys, denies hand-edits to lockfiles and generated files, and carries a format-on-write hook plus a `docs/reference/` regeneration or staleness hook where the stack supports one, plus a `## Project skills` section in the project `CLAUDE.md`. Refuses to start without `docs/spec/` and points to `/init-project`. Re-run it when the stack changes. |

### Agents

All agents are read-only except `docs-writer`. Claude picks them from their descriptions; you can also ask for one by name.

| Agent | Model | What it does |
|---|---|---|
| `code-reviewer` | opus | Reviews the uncommitted diff against `origin/main`. Reports BLOCKER / SHOULD-FIX / NIT as `file:line - problem - fix`, or `LGTM`. Keeps per-project memory of recurring issues. Preloads the `write-code` and `write-tests` skills, and loads the project's `coding` and `testing` skills when it has them, so the same agent judges idiom by whatever stack it is pointed at. Judges tests in the diff against `write-tests`, plus the diff-only checks: behaviour changed with no test, assertions loosened instead of updated, new branches left uncovered, tests deleted while their behaviour stayed. |
| `security-reviewer` | opus | Audits the diff for authz gaps, injection, leaked secrets, over-broad IAM, risky dependencies. Reports only what the diff introduces, ordered by severity. |
| `simplifier` | sonnet | Finds code the diff added that can be deleted or inlined: single-use wrappers, impossible guards, orphans, tests that cannot fail. Only proposes changes that reduce line count. |
| `docs-writer` | sonnet | The only agent that edits files, and only `README.md` and `docs/`. Keeps `docs/spec/` current, writes how-tos, explanations, and ADRs. Runs every snippet before writing it. Preloads the `write-docs` skill, which holds the house style and page types; the agent itself only adds the delegation rules - establish the subject from the repo rather than the prompt, and report gaps instead of guessing. |
| `docs-researcher` | haiku | Answers "how do I call this API at the version this project has installed". Checks `uv.lock` / `npm ls` first, then Context7, then official docs. Returns version, snippet, gotchas, source. |

## What `/init-claude-configs` generates

Seven files plus `settings.json` under `<project>/.claude/`, each written for the project's language, framework, and toolchain. A rule that would be true in any repo is left to the global skills; these hold only what is specific to this one, each convention with the file that shows it.

| File | Invoke | What it holds |
|---|---|---|
| `skills/coding/SKILL.md` | Loaded before writing code, after `write-code` | Stack table with runner prefix, where each kind of file lives, framework and dependency idioms the code shows, the checks table (format, lint, types, fast and slow tests, build - every command verified before it was written), and how dependencies are added. |
| `skills/testing/SKILL.md` | Loaded before writing tests, after `write-tests` | Runner and config already in force (markers, plugins, coverage threshold), test layout, shared fixtures to reuse, and the framework idioms. |
| `skills/review/SKILL.md` | `/review [base branch]` (user only) | Runs the checks table from `coding`, then dispatches `code-reviewer`, `security-reviewer`, and `simplifier` in parallel and merges their findings into one BLOCKER / SHOULD-FIX / CONSIDER list, ending with whether the branch is mergeable. Reports only. |
| `skills/deps/SKILL.md` | `/deps add <pkg>`, `/deps audit`, `/deps upgrade [pkg]` | Dependency changes through the package manager only, never hand-written versions. `add` checks the need first and reports the resolved version; `audit` reports outdated and vulnerable packages without changing anything; `upgrade` asks `docs-researcher` for breaking changes before touching one package at a time. Refuses versions released less than 7 days ago so a hijacked release has time to be caught: natively where the manager supports it (`uv --exclude-newer`, `npm --before`), otherwise by checking the registry's release date and pinning the newest old-enough version. The template carries verified command sets for `uv`, `poetry`, `pip`, and `npm`; any other manager is looked up and verified in the repo. |
| `agents/coder.md` | Agent tool, opus | Implements a scoped change and runs the checks. Preloads `write-code`, `write-tests`, `coding`, `testing`. Stops and reports rather than guessing on an ambiguous task; never touches `.claude/`, `docs/`, CI, or manifests. |
| `agents/test-writer.md` | Agent tool, opus | Writes tests only, under the test directory `testing` names. Preloads `write-tests` and `testing`. |

On a greenfield repo the checks table is marked unverified until the first coding session has run every row.

`docs-researcher` lives in `home/`, so a project on a machine without the global config does not have it; the generated `deps` and the `aws-cdk` skill say to do the lookup by hand and report that they did, rather than answering from memory. The generated `review` depends on `code-reviewer`, `security-reviewer`, and `simplifier` from the same place.

## What is in `infra/`

Skills for projects that define cloud infrastructure. Copy `infra/skills` into `<project>/.claude/` only when the repo has infrastructure code; they are not global.

| Skill | Invoke | What it does |
|---|---|---|
| `aws-cdk` | Loaded before touching CDK code | Conventions for AWS CDK in TypeScript: stack layout and props over globals, L2 constructs over `Cfn*`, generated physical names, grant methods over wildcard IAM, `RemovalPolicy.RETAIN` on stateful resources, and the `tsc` / `cdk synth` / `cdk diff` / `assertions` loop that has to pass. Never deploys or destroys on its own. |

## Greenfield project

1. Create the repo and `cd` into it.
2. Run `/init-project <one-line description>`. Answer the interview; say "you decide" for anything you do not care about and it is recorded in `docs/spec/assumptions.md`. Stop early if you must; unresolved items go to `docs/spec/open-questions.md`.
3. Confirm the final summary. The skill writes `docs/spec/`, `docs/decisions/`, and a `CLAUDE.md` pointing at them, then asks whether to commit.
4. Run `/init-claude-configs`. It reads `docs/spec/stack.md`, asks about whatever the spec leaves open, and writes the `coding`, `testing`, `review`, and `deps` skills and the `coder` and `test-writer` agents for that stack. Restart the session afterwards - skills and agents are discovered at session start.
5. Add `cp -r infra/skills <project>/.claude/` if the project defines cloud infrastructure.
6. Build from the spec. When a design changes, update the spec file in the same commit; `docs-writer` can do this.

## Brownfield project

1. `cd` into the existing repo. If it already has a `CLAUDE.md`, keep it; `init-project` only adds or replaces the `## Project spec` section.
2. Run `/init-project <one-line description>`. It reads the README, manifests, entry points, and data models first and only asks about what the code does not answer. Where the code and your answer disagree, it reports the gap rather than papering over it. `/init-claude-configs` will not start without the spec it writes.
3. Run `/init-claude-configs`. It builds the stack fact sheet from the lockfile, tool config, CI, tests, and source, asks only about what the code does not answer, verifies every check command, and writes the skills and agents. Where the code and your answer disagree, it asks which stands rather than papering over it.
4. Add the `infra/` skills if the repo already has CDK code.
5. Run `/deps audit` for a first picture of outdated or vulnerable dependencies. Plan majors separately; do security fixes first.
6. Before the first non-trivial change, run `/review` so the reviewer agents' project memory starts with the repo's existing conventions.
