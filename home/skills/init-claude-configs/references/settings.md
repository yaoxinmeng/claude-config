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
      "Bash(git push *)"
    ],
    "deny": [
      "Read(.env)",
      "Read(.env.*)",
      "Read(**/.env)",
      "Read(**/.env.*)",
      "Read(**/*.pem)",
      "Read(**/*.key)"
    ]
  }
}
```

What each block does and how to fill it:

- `Read(./**)` and `Edit(./**)` cover every file under the project. `Edit(...)` is the rule for all file-writing tools (Write, Edit, NotebookEdit); a `Write(...)` rule is not matched. Deny rules win over allow, so the `.env` entries still block.
- The `Bash(<...> *)` rows are the commands the Checks table in `coding` and the runner in `testing` use - one row per verified prefix (`uv run`, `npm run`, `make`, `cargo`, `go`, `pytest`). Add nothing that step 3 did not verify; a prefix rule allows every command that starts with it.
- Git: branch, checkout, switch, add, and commit run without a prompt; status, diff, and log are read-only and appear in nearly every turn. Push goes to `ask` because it publishes; a rule in `ask` prompts even when a broader allow would match.
- Deny covers `.env` files at any depth and private keys. Add any other secret file the fact sheet found (`secrets/**`, a cloud credentials file), and say so in the report.
- If the repo already has `.claude/settings.local.json`, leave it alone; it is the developer's personal overrides and is gitignored.
