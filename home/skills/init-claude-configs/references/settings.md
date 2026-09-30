# Template: `.claude/settings.json`

Project permissions, committed so every session and every teammate starts with the same rules. Read the existing file first and merge into its arrays; never replace a rule the team already added. Validate the result with `jq . .claude/settings.json` (or any JSON parser): a malformed file silently disables every setting in it.

```json
{
  "permissions": {
    "allow": [
      "Read(./**)",
      "Edit(./**)",
      "Bash(<runner prefix> *)",
      "Bash(<task runner> *)",
      "Bash(git status)",
      "Bash(git diff *)",
      "Bash(git log *)",
      "Bash(git branch *)",
      "Bash(git checkout *)",
      "Bash(git switch *)",
      "Bash(git add *)",
      "Bash(git commit *)"
    ],
    "ask": [
      "Bash(git push *)",
      "Edit(<migrations dir>/**)"
    ],
    "deny": [
      "Read(.env)",
      "Read(.env.*)",
      "Read(**/.env)",
      "Read(**/.env.*)",
      "Read(**/*.pem)",
      "Read(**/*.key)",
      "Edit(**/<lockfile>)",
      "Bash(<installer> install *)",
      "Bash(<installer> add *)",
      "Edit(CHANGELOG.md)",
      "Edit(docs/reference/**)",
      "Edit(.claude/skills/write-code/**)",
      "Edit(.claude/skills/write-tests/**)",
      "Edit(.claude/agents/code-reviewer.md)",
      "Edit(.claude/agents/security-reviewer.md)",
      "Edit(.claude/agents/simplifier.md)",
      "Edit(.claude/agents/docs-researcher.md)"
    ]
  }
}
```

What each block does and how to fill it:

- `Read(./**)` and `Edit(./**)` cover every file under the project. `Edit(...)` is the rule for all file-writing tools (Write, Edit, NotebookEdit); a `Write(...)` rule is not matched. Deny rules win over allow, so the `.env` entries still block.
- The `Bash(<...> *)` rows are the commands the Checks table in `coding` and the runner in `testing` use - one row per verified prefix (`uv run`, `npm run`, `make`, `cargo`, `go`, `pytest`). Add nothing that step 3 did not verify; a prefix rule allows every command that starts with it.
- Git: branch, checkout, switch, add, and commit run without a prompt; status, diff, and log are read-only and appear in nearly every turn. Push goes to `ask` because it publishes; a rule in `ask` prompts even when a broader allow would match.
- Deny covers `.env` files at any depth and private keys. Add any other secret file the fact sheet found (`secrets/**`, a cloud credentials file), and say so in the report.
- The remaining deny rows make rules the project already states actually binding, so nothing depends on a session remembering them:
  - `<lockfile>` and `<installer>` come from the fact sheet (`uv.lock` + `uv`, `package-lock.json` + `npm`, `Cargo.lock` + `cargo`). They route every dependency change through `/deps`, which resolves versions and checks release age and advisories. List each installer subcommand that writes the manifest - `install`, `add`, `remove`, `uninstall` - not a bare `Bash(<installer> *)` prefix, which would also block reads and running scripts.
  - `CHANGELOG.md` and `docs/reference/**` are generated. Deny them only where that is true here: a hand-maintained changelog is a normal file and must stay editable. Check before adding the row.
  - The `write-code`, `write-tests`, and reviewer-agent rows are the files copied from the global config. They are maintained in the source repo, so an edit here is drift by definition. Write one row per file actually copied and drop the rest - a deny for a file that is not there is a claim about this repo that is false. The rows do not block a refresh: `/init-claude-configs` replaces a copy with `cp`, and the deny covers the file-writing tools.
  - `Edit(<migrations dir>/**)` goes in `ask`, not `deny`: a landed migration is immutable, but writing a new one is ordinary work, and `ask` gets the pause without blocking it. Drop the row when the project has no migrations.
- A deny is a permission rule, not a hook. Anything the permission system can express belongs here - it is enforced before the tool runs, needs no shell, and cannot be defeated by a failing script.
- If the repo already has `.claude/settings.local.json`, leave it alone; it is the developer's personal overrides and is gitignored.

## The format-on-write hook

Add this whenever step 3 verified a formatter. It runs on the file that was just written, so a turn never ends with a diff full of formatting churn and the Checks table's format step has nothing left to do:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "jq -r '.tool_response.filePath // .tool_input.file_path' | { read -r f; case \"$f\" in <source globs>) <verified format command> \"$f\" ;; esac; } >/dev/null 2>&1 || true",
            "timeout": 30
          }
        ]
      }
    ]
  }
}
```

How to fill it:

- `<verified format command>` is the formatter from step 3 with its runner prefix and its write flag - `uv run ruff format`, `npx prettier --write`, `gofmt -w`, `cargo fmt --`. A check-only invocation formats nothing; make sure the flag that rewrites the file is there.
- `<source globs>` is a `case` pattern of the extensions the formatter handles, `|`-separated: `*.py`, `*.ts|*.tsx|*.js|*.jsx`, `*.go`. Without it the formatter is handed Markdown and JSON it does not know, and an editing turn pays for a failing subprocess every time. Skip the `case` only when the formatter has its own ignore-unknown flag (`prettier --ignore-unknown`).
- Several formatters, one per language, means one `PostToolUse` entry each, each with its own `case`. Do not chain them in one command.
- `|| true` keeps a formatter failure from surfacing as a tool error. A file the formatter rejects is a syntax error the session is about to see anyway.
- Pipe-test the command before writing it, with a real file from this repo: `echo '{"tool_input":{"file_path":"<a real source file>"}}' | <command>`. Check the file was actually reformatted, not just that the command exited 0.
- The formatter prefix needs its `Bash(...)` row in `permissions.allow`.

## The no-op `cd` hook

Add this to every project. Sessions habitually open a command with `cd <project dir> &&` even though the shell already starts there. The `cd` adds nothing, but it turns the command into a compound one, which no `Bash(<prefix> *)` allow row matches, so the user gets a prompt for a command the rules already allow. This hook refuses a leading `cd` into the directory the shell is already in, and tells the session to drop it:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash|PowerShell",
        "hooks": [
          {
            "type": "command",
            "command": "jq -r '.cwd, .tool_input.command' | tr -d '\\r' | { read -r cwd; read -r cmd; d=$(printf '%s' \"$cmd\" | sed -nE \"s/^[[:space:]]*(cd|Set-Location|pushd)[[:space:]]+(\\\"([^\\\"]*)\\\"|'([^']*)'|([^[:space:];&|]+))[[:space:]]*(&&|;|\\$).*/\\3\\4\\5/p\"); here=$(cd -- \"$cwd\" 2>/dev/null && pwd -P); [ -n \"$d\" ] && [ -n \"$here\" ] && [ \"$(cd -- \"$here\" && cd -- \"$d\" 2>/dev/null && pwd -P)\" = \"$here\" ] && { echo \"The shell is already in $cwd. Drop the leading cd and run the rest of the command as is.\" >&2; exit 2; }; exit 0; }",
            "timeout": 10
          }
        ]
      }
    ]
  }
}
```

Why it is shaped this way:

- It compares against the hook input's `cwd`, not `$CLAUDE_PROJECT_DIR`. The shell's directory persists between calls, so after a session has moved into a subdirectory, a `cd` back to the root does real work and must pass. Only a `cd` that lands where the shell already is gets refused.
- Both sides are resolved with `cd` and `pwd -P`, so `C:\repo`, `C:/repo`, `/c/repo`, `.`, and a symlinked path all compare equal. Matching the path as text would miss most of the spellings a session actually uses.
- `tr -d '\r'` matters on Windows: jq there ends lines with CRLF, the stray `\r` makes both `cd` calls fail, and two empty strings compare equal - every `cd` would be refused. The `-n "$here"` test is the second guard against the same failure.
- `exit 2` from `PreToolUse` blocks the call and hands stderr to the session, which reruns the command without the `cd`. A permission rule cannot express "a `cd` to the current directory", which is why this one is a hook.
- `Set-Location` and `pushd` cover the PowerShell tool on Windows; drop `PowerShell` from the matcher on a project nobody runs on Windows, and keep the rest.
- Pipe-test it before writing it, with the project root as `cwd`: `jq -nc --arg cwd "$PWD" --arg c "cd \"$PWD\" && git status" '{cwd:$cwd,tool_input:{command:$c}}' | <command>` must exit 2, and the same with `cd <a subdirectory> && ls` and with a plain `git status` must exit 0. Build the input with `jq -n` rather than by hand - a Windows path in hand-written JSON is an invalid escape, jq fails, and the hook passes everything silently.

## The copied-files drift hook

Add this whenever step 4 copied any file from the global config. It compares each copy against its source once per session start and names the ones that have drifted:

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup",
        "hooks": [
          {
            "type": "command",
            "command": "for f in <copied paths>; do p=\"$CLAUDE_PROJECT_DIR/.claude/$f\"; s=\"$HOME/.claude/$f\"; [ -f \"$p\" ] && [ -f \"$s\" ] || continue; grep -v '^> Copied from the global config' \"$p\" | diff -q - \"$s\" >/dev/null || echo \"Stale copy: .claude/$f differs from ~/.claude/$f. Re-run /init-claude-configs to refresh it; fix the source, not the copy.\"; done; exit 0",
            "timeout": 10
          }
        ]
      }
    ]
  }
}
```

How to fill it and why it is shaped this way:

- `<copied paths>` is the space-separated list of the files step 4 actually copied, each relative to `.claude/` - `skills/write-code/SKILL.md agents/code-reviewer.md`. The same relative path holds under `~/.claude/`, which is what lets one loop cover both. Drop the hook when nothing was copied.
- `SessionStart` stdout is added to the session's context, so a drifted copy is something the session knows about before it loads anything. Silence is the no-drift case: the loop prints nothing and costs one `diff` per file.
- The `grep -v` strips the provenance line the copies carry, which is the one intended difference. It strips one line, so the copy must carry exactly one and no blank line with it, as step 4 says. Get that wrong and the hook reports every file as stale on a clean project - test the silent case before you trust the noisy one.
- Both `-f` tests matter. On a collaborator's machine `$HOME/.claude/` has no source to compare against, and the hook stays quiet rather than warning about something they cannot fix - it is a check for whoever maintains the source repo.
- `matcher: "startup"` only. On `clear` and `compact` the same warning would be re-added to context mid-task, where nobody is going to act on it.
- `exit 2` from a `SessionStart` hook blocks the session from starting, so the command ends in `exit 0`: drift is worth saying, never worth refusing to work over. The `|| continue` and the trailing `exit 0` also keep a missing file from ending the loop early.
- Hooks do not go through the permission system, so this needs no `permissions.allow` row.
- Test it before writing it, with `CLAUDE_PROJECT_DIR` set to the project root: once as-is, which must print nothing, and once after appending a line to one copy, which must name that one file. Undo the edit afterwards.

## The reference-docs hook

Only when step 5 of the skill says the project qualifies. Step 5 also decides which of the two shapes below applies; a repo with both a cheap and an expensive generator gets both, as two entries in the same `Stop` array. Merge alongside `permissions` - a second `hooks` key silently replaces the first.

### Cheap generator: regenerate

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "git diff --quiet HEAD -- <source dir> || <verified generate command> >/dev/null 2>&1 || true",
            "async": true,
            "timeout": 120,
            "statusMessage": "Regenerating docs/reference"
          }
        ]
      }
    ]
  }
}
```

How to fill and why it is shaped this way:

- `Stop` fires once when the turn ends, so the reference is regenerated per turn rather than per edit - a `PostToolUse` hook on `Write|Edit` would rebuild the docs several times inside one change and slow every edit down.
- The `git diff --quiet HEAD -- <source dir>` guard is what keeps it cheap: on a turn that touched no source file the hook exits immediately. `<source dir>` is the package root from the fact sheet (`src/`, `lib/`, `pkg/`), several paths if the repo has several components.
- `<verified generate command>` is the exact command from step 3, runner prefix included (`uv run pdoc -o docs/reference src/<pkg>`, `npm run docs`, `make docs`). Prefer the repo's own task target when it has one.
- `async: true` returns the turn immediately and generates in the background; `|| true` keeps a generator failure from ending the turn with an error. A broken generator shows up when the command is run by hand or in CI - it does not belong in the Stop path.
- The generate command's prefix also needs a `Bash(...)` row in `permissions.allow`, same as any other verified command.
- Add `docs/reference/` to the repo's `.gitignore` only if the team does not publish the generated docs from the repo; ask rather than assume, and leave an existing choice alone.
- Validate with `jq -e '.hooks.Stop[].hooks[].command' .claude/settings.json` after writing. The hook only loads for sessions started after the file exists - tell the user to run `/hooks` once or restart.

### Expensive generator: warn that the artifact is stale

For an artifact that only a container, a database, a migration run, or a booted server can produce. The hook never runs that generator - it compares mtimes and tells the session to run it:

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "[ -f <artifact> ] && [ -z \"$(find <source dir> -newer <artifact> -name '<source glob>' -print -quit)\" ] || echo '{\"systemMessage\":\"<artifact> is behind <source dir>. Run `<verified generate command>` to regenerate it.\"}'",
            "timeout": 10
          }
        ]
      }
    ]
  }
}
```

How to fill it:

- `<artifact>` is the generated file itself (`docs/reference/openapi.json`, `docs/reference/schema.sql`), not the directory - a directory mtime does not move when a file inside it is rewritten.
- `<source dir>` and `<source glob>` are what the artifact is derived from, which is often narrower than the whole package: migrations for a schema dump (`migrations`, `*.sql`), the route and model modules for an OpenAPI schema. Too wide a source is a hook that cries stale on every unrelated edit and gets ignored.
- The `[ -f <artifact> ]` test makes a missing artifact warn too, which is the case that matters on a fresh clone.
- `<verified generate command>` is the full on-demand command from step 3, `docker compose up` and all. Spell it out - the warning is only useful if the reader can act on it without going looking.
- No `async` here: the hook's `systemMessage` is what surfaces the warning, and a `find` over one source tree is already fast. Keep `timeout` short so a huge tree cannot stall the turn.
- This hook runs no generator, so it needs no new `permissions.allow` row.
