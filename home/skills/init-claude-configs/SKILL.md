---
name: init-claude-configs
description: Generate a project's `.claude/` config - `coding`, `testing`, `review`, and `deps` skills, `coder`, `test-writer`, and `code-reviewer` agents, and `settings.json` permissions - written for this project's language, framework, dependencies, and toolchain. Reads the codebase first, interviews the user for what the code cannot answer, and verifies every command before writing it down. Use when a project has no `.claude/skills/`, when starting a greenfield project, or when the stack changed and the config has drifted.
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
- Never install a tool to make a command work. Report the gap instead; the user decides.
- Greenfield: there is nothing to run yet. Write the commands the chosen tools document for a default install and mark the table `Unverified - remove this line after the first session runs every row`. The first coding session removes the marker.

## 4. Write the files

Write these seven files from the templates under `references/`. Each template states what goes in each section and the rules for filling it; read the template before writing the file.

| File | Template | Purpose |
|---|---|---|
| `.claude/skills/coding/SKILL.md` | `references/coding.md` | Stack, layout, idioms, checks - loaded before any code is written |
| `.claude/skills/testing/SKILL.md` | `references/testing.md` | Runner, config, layout, fixtures, framework idioms - loaded before any test is written |
| `.claude/skills/review/SKILL.md` | `references/review.md` | `/review`: runs the checks, dispatches the reviewer agents, merges findings |
| `.claude/skills/deps/SKILL.md` | `references/deps.md` | `/deps add`, `/deps audit`, `/deps upgrade`: dependency changes through the package manager, with release-age and vulnerability checks |
| `.claude/agents/coder.md` | `references/coder.md` | Implements a scoped change; preloads `write-code`, `write-tests`, `coding`, `testing` |
| `.claude/agents/test-writer.md` | `references/test-writer.md` | Writes tests only; preloads `write-tests` and `testing` |
| `.claude/agents/code-reviewer.md` | `references/code-reviewer.md` | The global reviewer, preloading `coding` and `testing` so it judges idiom by this stack |

Rules that apply to every file:

- No placeholder left. A section with nothing project-specific to say is deleted, not filled with generalities.
- No global rule restated. If a sentence would be true in any repo, it belongs in `write-code` or `write-tests`, and is already there.
- Every command appears exactly as it was verified in step 3, with the runner prefix.
- Every convention names its evidence when the source is the code (`see src/api/errors.py`), so a future session can check whether it still holds.
- The frontmatter `description` says when to load the skill, in one or two sentences, and names the language and framework so the skill triggers on them.
- A repo with several components (a monorepo, a backend plus a frontend) gets one subsection per component inside each skill, not one skill per component. The agents are shared.
- When updating an existing file, keep any section the user wrote that the fact sheet does not contradict, and say what you changed.

## 5. Set permissions

Write `.claude/settings.json` from `references/settings.md`, merging into whatever the file already holds. The rules it sets: read and edit anything under the project, run the verified check commands and the runner prefix without a prompt, run `git` branch, checkout, add, and commit without a prompt, ask before `git push`, and never read `.env` files or private keys. Every `Bash(...)` allow row must come from a command verified in step 3; a prefix rule allows everything that starts with it, so an unverified one is a guess about what is safe.

Validate the JSON after writing. A malformed settings file disables every setting in it without an error.

## 6. Point CLAUDE.md at the skills

Skills only trigger reliably when the project says to load them. Create or update the project's `CLAUDE.md` (repo root) with a `## Project skills` section, leaving every other section untouched, that says in under six lines:

- Load the `coding` skill before writing or editing any code in this repo, after the global `write-code`.
- Load the `testing` skill before writing or editing any test, after the global `write-tests`.
- `/review` runs the repo's checks and the reviewer agents; run it before asking for a merge.
- `/deps` is the only way dependencies change; never hand-edit the manifest or lockfile.

Point, don't copy: nothing from the skills is restated here.

## 7. Finish

Report in under twelve lines: files written or updated, the commands verified, the commands marked unverified, the permission rules added to `.claude/settings.json`, the facts the user decided, and whether `CLAUDE.md` was created or updated. Then ask one question: whether to commit `.claude/` and `CLAUDE.md` now. Do not commit without a yes.
