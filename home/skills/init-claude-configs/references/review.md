# Template: `.claude/skills/review/SKILL.md`

The only project-specific part is the checks section; the checks table itself lives in the `coding` skill so it is written once. Fill `<...>` from the fact sheet. Lines in `> ` are instructions to you, not content; remove them.

```markdown
---
name: review
description: Full pre-merge review of this <language> / <framework> repo - runs the repo's own checks, then dispatches the reviewer, security, and simplifier agents in parallel and merges their findings into one prioritized list.
disable-model-invocation: true
argument-hint: "[base branch, defaults to origin/main]"
---

Review the uncommitted and unpushed work against $ARGUMENTS (default `origin/main`).

## 1. Scope

Run `git diff --stat --merge-base origin/main` (fall back to `git diff --stat HEAD`). If the diff is empty, say so and stop.
If it exceeds ~1500 changed lines, say it should be split and ask before continuing.

## 2. Checks

Run these first. Agents should not spend turns on what a linter catches.

Read `.claude/skills/coding/SKILL.md` and run every row of its Checks table, in order, with the runner prefix it names. <Skip the slow tests row when Docker is unavailable, and say so.>

Never install a tool to satisfy the table. Name every check you skipped and why - a skipped check is a finding, not a blank.

Report failures as a short list. Do not fix anything yet.

## 3. Agents

Dispatch all three in one message so they run in parallel:

- `code-reviewer` - correctness, conventions, missing tests. It loads this repo's `coding` and `testing` skills itself, so it judges idiom by this stack.
- `security-reviewer` - authz, injection, secrets, <tenant scoping / any repo-specific concern from the fact sheet>
- `simplifier` - what can be deleted

Give each the base branch in its prompt. Do not summarize the diff for them; they read it themselves.

`code-reviewer`, `security-reviewer`, and `simplifier` all ship with the global config (`~/.claude/agents/`), not with this repo. If any is unavailable, do that pass yourself against the same criteria, and say which agent you stood in for - the findings still have to be produced.

## 4. Merge

One list, deduplicated, in this order:

1. **BLOCKER** - failing checks, plus CRITICAL/HIGH security findings and reviewer BLOCKERs
2. **SHOULD-FIX** - reviewer SHOULD-FIX, MEDIUM security, deletions over 20 lines
3. **CONSIDER** - nits, small deletions, LOW findings

Each item: `file:line - problem - fix`. Where two agents found the same thing, keep the more specific wording and cite both. Where they disagree, show both positions in one line and say which you would take.

Finish with a single line: how many blockers, and whether this is mergeable.

Then stop. Do not start fixing unless asked.
```
