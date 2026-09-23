---
name: init-claude-configs
description: Generate a project's `.claude/` config - `coding`, `testing`, `review`, and `deps` skills, `coder` and `test-writer` agents, and `settings.json` permissions and hooks - written for this project's language, framework, dependencies, and toolchain, plus copies of the global skills and agents that config depends on so the project also works on a machine without the global config. Reads the codebase first, interviews the user for what the code cannot answer, and verifies every command before writing it down. Use when a project has no `.claude/skills/`, when starting a greenfield project, or when the stack changed and the config has drifted.
disable-model-invocation: true
argument-hint: "[directory to scope to, defaults to the repo root]"
---

Generate `.claude/` for the project at $ARGUMENTS (default: the repo root).

Goal: leave the project with skills and agents that a session can load and immediately write, test, and review code the way this repo does it - right runner prefix, right layout, right idioms, right checks. The global skills (`write-code`, `write-tests`) already hold the standards that apply everywhere; the files you generate hold only what is specific to this repo. A line that would be true in any project does not belong in them.

Never write application code in this skill.

## 1. Read before asking

First check for `docs/spec/README.md`. If it is missing, stop and tell the user to run `/init-project` first: it records the stack, layout, and testing strategy that this skill would otherwise have to interview for, and on a brownfield repo it reads the code before asking. Continue without it only if the user says so, and note in the report that the spec is missing.

Build a stack fact sheet from the repo so you only ask about what the code cannot answer. Look at:

- Manifests and lockfiles: `pyproject.toml`, `package.json`, `go.mod`, `Cargo.toml`, `*.csproj`, `Gemfile`, `pom.xml`, `build.gradle*`, and their locks. The lockfile decides the package manager and the runner prefix (`uv run`, `poetry run`, `npm run`, `pnpm`, `cargo`, `go`).
- Doc generators already in the dev dependencies or task runner (`pdoc`, `sphinx` with `autodoc`, `mkdocstrings`, `typedoc`, `cargo doc`, `godoc`/`gomarkdoc`, `dotnet docfx`), and whether `docs/reference/` already exists and what wrote it.
- Tool config: lint, format, and type-check sections or files (`ruff`, `eslint`, `biome`, `prettier`, `mypy`/`pyright`, `tsconfig`, `golangci`, `clippy`), `.editorconfig`, `pre-commit` config.
- Task runners: `Makefile`, `justfile`, `package.json` scripts, `[tool.poe]`, `nox`/`tox`, `Taskfile`. A task the repo defines is the definition of "the checks pass here"; prefer it over calling tools one by one.
- CI workflows: which checks run on a pull request, in what order, with what flags.
- Tests: framework, config (markers, plugins, timeouts, coverage threshold), layout (mirrored tree or beside the module), shared fixtures (`conftest.py`, `setup.ts`, test base classes), and the three most recent test files for idiom.
- Source: the entry points, the directory layout (where models, handlers, services, config, migrations live), and three representative modules for idiom - error types, logging, validation, async usage, how the framework is wired.
- Frameworks and key dependencies from imports, not from the manifest alone: a listed dependency nobody imports is not a convention.
- `docs/spec/stack.md` and `docs/spec/design.md`, and the existing `CLAUDE.md`.
- Existing `.claude/skills/` and `.claude/agents/`. If `coding`, `testing`, `review`, or `deps` already exist, read them; you will update in place, not overwrite blind.

Record the fact sheet in the scratchpad directory (not the repo) with one row per fact and its source (`file:line` or "interview"). Summarize it to the user in under fifteen lines and name every fact still missing.

## 2. Interview for the gaps

A greenfield repo answers nothing, so every area below is a question. A brownfield repo usually answers most of them; ask only about the rest, and about anything the code answers two different ways.

1. **Language and framework** - language and version per component; framework per component; anything the user wants excluded and why.
2. **Toolchain** - package manager, lint, format, type checker, test framework, task runner. For greenfield, recommend the mainstream choice for the framework and say why in one line.
3. **Layout** - where each kind of file goes, `src/` layout or flat, feature folders or layer folders, where tests go.
4. **Idioms** - validation at the boundary (which library), error handling (which base type, what gets logged), logging library and format, async or sync, ORM and session handling, HTTP client, config loading. Ask only about the ones that matter for this stack.
5. **Checks** - which commands must pass before work is reported done, in which order, and any that are slow enough to run separately (integration tests, e2e, build).

Rules:

- Use AskUserQuestion, following the `concise` skill: one dimension per question, concrete options, recommended option first with a one-line reason, an explicit default so the user can skip.
- At most four questions per call. Batch what is independent.
- When the user says "you decide", decide, say the choice and reason in one line, and move on. Do not ask again.
- When an answer contradicts what the code does, say so and ask which stands. If the code stands, the answer is not a convention; if the answer stands, note in the generated skill that the code has not caught up yet.
- When an answer is vague ("standard setup", "the usual"), ask for the tool name or the command.
- End with a summary of the fact sheet under twenty lines and ask the user to confirm or correct it before writing anything.

## 3. Verify every command

A command written into a skill is a promise that it works here. Before writing any check command:

- Run it with the runner prefix from the fact sheet. Record the exact command that succeeded, or that failed for a real reason (a lint error in the repo is a real reason; "command not found" is not - that tool is not configured and does not go in the table).
- Prefer the repo's own task over the raw tool: `make check`, `npm run lint`, `uv run poe test`. Name the raw tool in a note if a reader will need it.
- The reference-doc generator from step 5 is a check command like any other: run it, confirm it writes into `docs/reference/`, and record the exact command and roughly how long it took - step 5 needs the timing. If it writes somewhere else, point its output at `docs/reference/` in its own config file, not with a shell move. If it needs a container, a database, or a running server that is not already up, do not start one: mark the command unverified, record what it needs, and tell the user.
- Never install a tool to make a command work. Report the gap instead; the user decides.
- Greenfield: there is nothing to run yet. Write the commands the chosen tools document for a default install and mark the table `Unverified - remove this line after the first session runs every row`. The first coding session removes the marker.

## 4. Write the files

Write these six files from the templates under `references/`. Each template states what goes in each section and the rules for filling it; read the template before writing the file.

| File | Template | Purpose |
|---|---|---|
| `.claude/skills/coding/SKILL.md` | `references/coding.md` | Stack, layout, idioms, checks - loaded before any code is written |
| `.claude/skills/testing/SKILL.md` | `references/testing.md` | Runner, config, layout, fixtures, framework idioms - loaded before any test is written |
| `.claude/skills/review/SKILL.md` | `references/review.md` | `/review`: runs the checks, dispatches the reviewer agents, merges findings |
| `.claude/skills/deps/SKILL.md` | `references/deps.md` | `/deps add`, `/deps audit`, `/deps upgrade`: dependency changes through the package manager, with release-age and vulnerability checks |
| `.claude/agents/coder.md` | `references/coder.md` | Implements a scoped change; preloads `write-code`, `write-tests`, `coding`, `testing` |
| `.claude/agents/test-writer.md` | `references/test-writer.md` | Writes tests only; preloads `write-tests` and `testing` |

Rules that apply to every file:

- No placeholder left. A section with nothing project-specific to say is deleted, not filled with generalities.
- No global rule restated. If a sentence would be true in any repo, it belongs in `write-code` or `write-tests`, and is already there.
- Every command appears exactly as it was verified in step 3, with the runner prefix.
- Every convention names its evidence when the source is the code (`see src/api/errors.py`), so a future session can check whether it still holds.
- The frontmatter `description` says when to load the skill, in one or two sentences, and names the language and framework so the skill triggers on them.
- A repo with several components (a monorepo, a backend plus a frontend) gets one subsection per component inside each skill, not one skill per component. The agents are shared.
- When updating an existing file, keep any section the user wrote that the fact sheet does not contradict, and say what you changed.
- The generated files name `write-code`, `write-tests`, `code-reviewer`, `security-reviewer`, `simplifier`, and `docs-researcher`. Those live in the global config, so the next subsection copies them into the project rather than leaving the references to resolve against a home directory the next collaborator may not have.

### Vendor the global dependencies

A project whose `.claude/` points at `~/.claude/` only works on a machine that has this user's global config. Copy the six files the generated config depends on into the project so it stands on its own, and so every rule a session loads is visible in the repo:

| Copy from | Copy to | Needed by |
|---|---|---|
| `~/.claude/skills/write-code/SKILL.md` | `.claude/skills/write-code/SKILL.md` | `coder`, and `CLAUDE.md` before any code is written |
| `~/.claude/skills/write-tests/SKILL.md` | `.claude/skills/write-tests/SKILL.md` | `coder`, `test-writer` |
| `~/.claude/agents/code-reviewer.md` | `.claude/agents/code-reviewer.md` | `/review` |
| `~/.claude/agents/security-reviewer.md` | `.claude/agents/security-reviewer.md` | `/review` |
| `~/.claude/agents/simplifier.md` | `.claude/agents/simplifier.md` | `/review` |
| `~/.claude/agents/docs-researcher.md` | `.claude/agents/docs-researcher.md` | `/deps upgrade` |

Rules:

- Copy verbatim, with `cp`. Step 5 denies the file-writing tools on these paths, so `cp` is also how a later run refreshes one. These are not the place for project specifics - that is what `coding` and `testing` are for. A project copy that has drifted from its source is a second set of standards to keep in sync.
- Add a provenance line to each copy and nothing else: exactly one line, immediately after the closing `---` of the frontmatter, with no blank line around it. The drift hook in step 5 strips that one line before comparing, so an extra blank line makes every copy report as stale. Insert it in the same shell step as the copy (`sed`/`awk`), for the same reason the copy uses `cp`. The form: "> Copied from the global config (`<source path>`) on `<date>`. Do not edit here - edit the source and re-run `/init-claude-configs`."
- A source file that is missing on this machine is not copied and not invented. Say so in the report, name what breaks without it, and leave the reference in place.
- Nothing loads twice when both exist, but which one wins differs by kind, and the two rules point opposite ways: for a skill, `~/.claude/` wins over the project, so on a machine with the global config the copies of `write-code` and `write-tests` never load; for an agent, the project wins over `~/.claude/`, so the copied `code-reviewer`, `security-reviewer`, `simplifier`, and `docs-researcher` are the ones that run, there and everywhere else. A stale copied agent therefore silently replaces the source on the author's own machine. That is what the drift hook and the diff below are for.
- On a re-run, diff each copy against its current source before touching it. When they differ, say which files changed and whether the change came from the source or from a hand-edit in the project, and ask before overwriting a hand-edit. Never overwrite silently.

## 5. Set permissions and the reference-docs hook

Write `.claude/settings.json` from `references/settings.md`, merging into whatever the file already holds. The rules it sets: read and edit anything under the project, run the verified check commands and the runner prefix without a prompt, run `git` branch, checkout, add, and commit without a prompt, ask before `git push`, and never read `.env` files or private keys. Every `Bash(...)` allow row must come from a command verified in step 3; a prefix rule allows everything that starts with it, so an unverified one is a guess about what is safe.

The deny and ask rows make the project's own rules binding rather than remembered: dependency changes go through `/deps` and not a direct installer call, generated files are not hand-edited, a landed migration prompts before it is touched. Each of those rows is a claim about this repo - confirm it from the fact sheet before writing it, and drop the row when the claim is false. Prefer a permission rule over a hook wherever the permission system can express the rule: it is enforced before the tool runs and no script can fail open.

Add the format-on-write hook from `references/settings.md` whenever step 3 verified a formatter. It is the one hook worth having in nearly every project - it costs milliseconds on the file just written and keeps formatting churn out of the diff. Pipe-test it before writing it, as the template describes; a formatter that silently no-ops is worse than none, because the Checks table stops catching what it was supposed to catch.

`docs/reference/` is generated API reference - the global `write-docs` skill forbids hand-writing it, so something has to write it. Two preconditions apply to either hook below, and failing one means no hook at all:

- The project has a public API surface worth a reference: a library, an SDK, an HTTP API with a schema, a CLI, or a database schema other code is written against. A leaf application nobody imports does not get one.
- A generator is already a dependency or a task-runner target here. Never add one to make the hook possible; offer it as a follow-up instead, and say so in the report.

Then classify the generator by what it costs to run, because that decides which hook it gets. Run it once yourself to find out rather than guessing from its name - `openapi.json` is cheap when the app exposes a schema-dump entry point and expensive when the only way to get it is to boot the server.

**Cheap** - static analysis of the source, offline, no service to start, finishes in a couple of seconds: `pdoc`, `typedoc`, `mkdocstrings`, `cargo doc`, an OpenAPI dump from an in-process app object, a CLI `--help` render. These get the regenerating `Stop` hook from `references/settings.md`.

**Expensive** - needs a container, a database, a running server, a migration run, or the network: `pg_dump` of a schema built by migrations, an OpenAPI or GraphQL schema scraped from a booted server, a client generated from a live endpoint. These do not go in a generating hook - a turn that edits one line should not start Postgres, and a hook that leaves containers behind on a failed turn is worse than a stale file. They get the staleness-warning `Stop` hook instead: it compares mtimes, costs nothing, and tells the session the artifact is behind its source. Record the real command in the `coding` skill's Checks table as an on-demand step, and name it in the warning text so the reader knows what to run.

When the same repo has both kinds, each generator gets its own entry under `Stop` - one regenerating, one warning. Say in the report which generator landed in which tier and why.

Validate the JSON after writing. A malformed settings file disables every setting in it without an error.

## 6. Point CLAUDE.md at the skills

Skills only trigger reliably when the project says to load them. Create or update the project's `CLAUDE.md` (repo root) with a `## Project skills` section, leaving every other section untouched, that says in under seven lines:

- Load the `coding` skill before writing or editing any code in this repo, after `write-code`.
- Load the `testing` skill before writing or editing any test, after `write-tests`.
- `write-code`, `write-tests`, and the agents under `.claude/agents/` other than `coder` and `test-writer` are copies of the shared config, carried here so the repo works on any machine: never edit them here. Name where they came from, from the provenance line in the copies. Omit this line when nothing was copied.
- `/review` runs the repo's checks and the reviewer agents; run it before asking for a merge.
- `/deps` is the only way dependencies change; never hand-edit the manifest or lockfile.
- `docs/reference/` is generated, never hand-edited: name the hook in `.claude/settings.json` that regenerates it, and for an expensive generator name the command a session has to run when the hook warns the artifact is stale. Omit this line when no hook was written.

Point, don't copy: nothing from the skills is restated here.

## 7. Finish

Report in under fourteen lines: files written or updated, the files copied from the global config (and any that were missing, with what breaks without them, and any copy whose overwrite the user declined), the commands verified, the commands marked unverified, the permission rules and hooks added to `.claude/settings.json` (and any deny row you dropped because the claim behind it was false), the facts the user decided, and whether `CLAUDE.md` was created or updated. Say in one line that the new skills and agents are only discovered when a session starts, so this session cannot load them - the user needs a restart before `coding`, `testing`, `/review`, or `/deps` resolve. Then ask one question: whether to commit `.claude/` and `CLAUDE.md` now. Do not commit without a yes.
