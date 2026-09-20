# Template: `.claude/agents/code-reviewer.md`

A project-level agent with the same name as a user-level one takes precedence, so this file replaces the global `code-reviewer` inside this repo. The only difference is that it preloads `coding` and `testing`, so the reviewer judges idiom by this stack instead of guessing it from the diff.

Do not write this agent from scratch; the global one is the source of truth for the review process and criteria, and a second copy would drift.

1. Read `~/.claude/agents/code-reviewer.md`. If it is missing, stop and tell the user to install the global config (`home/` from the `claude-config` repo) first - `review` also depends on `security-reviewer` and `simplifier` from the same place.
2. Copy it to `.claude/agents/code-reviewer.md` verbatim, then make exactly these changes:
   - In the frontmatter `skills:` list, add `coding` and `testing` after the existing entries.
   - In `description`, prefix with the stack so the agent listing reads as this repo's: `Reviews the current uncommitted diff in this <language> / <framework> repo for correctness, ...`.
   - After the line `You review diffs. You never edit files.`, add this paragraph:

     > The `coding` and `testing` skills describe this repo's stack, layout, idioms, and checks. A diff that contradicts them is a SHOULD-FIX; a diff that shows the skill is out of date is a NIT that names the file to update.
3. Change nothing else. When the global agent changes, re-run `/init-claude-configs` to refresh this copy.
