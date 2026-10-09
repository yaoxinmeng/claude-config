---
name: feature
description: Build a feature end to end - plan with the planner agent and an approval checkpoint, then per step write failing tests from the acceptance criteria, implement, run the checks, review with code-reviewer and security-reviewer in a bounded fix loop, and commit; finish with a whole-branch review and a pull request.
disable-model-invocation: true
argument-hint: "<feature request>, or nothing to resume the feature on this branch"
---

Build this feature: $ARGUMENTS

You orchestrate. The agents write the plan, the tests, the code, and the findings; you relay between them and the user, run the checks, write the work files, and make the commits. Never write application code or tests yourself.

## Context

This session runs for the whole feature and will be summarized as it grows. Your memory of earlier steps is not reliable; the work files are.

- At the start of every step, re-read `progress.md`, `decisions.md`, and only this step's section of `plan.md` - from its `## Step N:` heading to the next `## Step` heading (`sed -n '/^## Step N:/,/^## Step/p'`). Work from what they say, not from what you remember. Read the whole plan only when planning, resuming, revising, or writing the pull request.
- Never let command output into the conversation whole. Run every test and check command with its output sent to a log in the work folder - `<command> > .feature/<branch>/logs/<step>-<check>.log 2>&1; echo "exit $?"` - then read the exit code and the last 40 lines. When those do not show the cause of a failure, grep the log for the failing test names or error lines rather than reading it all. When an agent has to fix a failure, give it the log path, not the output.
- Keep agent reports out of your own reasoning beyond what the next action needs. Anything worth keeping goes into a work file.
- `test-writer` and `coder` are new agents at the start of each step. Within a step, every "send back" continues the same agent with SendMessage (load it with ToolSearch if it is deferred), so it keeps what it already read. If the agent can no longer be reached, dispatch a new one with the step from `plan.md` and what has happened so far.
- `code-reviewer` and `security-reviewer` are new agents every round, so each review judges the diff without anchoring on the last one.
- `planner` is continued during its question-and-answer loop, and new for every plan revision, which it works from `plan.md`.

## 0. Preconditions

If the current branch already has a work folder (below), go to Resume instead. A request in $ARGUMENTS is then ignored; say so.

Stop and tell the user what to run if either fails:

- `.claude/skills/coding/SKILL.md` and `.claude/skills/review/SKILL.md` exist. Without them there is no Checks table and no review procedure - run `/init-claude-configs`.
- The working tree is clean (`git status --porcelain` prints nothing). Uncommitted work would end up in the first step's commit.

Then:

- If on the default branch, create `feature/<short-slug>` from it. Otherwise stay on the current branch.
- The work folder is `.feature/<branch>/`, with `/` in the branch name replaced by `-`. `git check-ignore -q .feature/` must succeed; if it does not, add `.feature/` to `.gitignore` and commit that alone ("Ignore feature work folder").
- Create the work folder with `plan.md`, `progress.md` (a `# Progress` heading only), `decisions.md` (a `# Won't fix` heading only), `findings/`, and `logs/`.

`progress.md` records each milestone the moment it happens, one line each: `- Plan approved`, `- Step N: <commit hash>`, `- Feature review: <commit hash or "no fixes">`. It is what Resume reads.

### Resume

1. Run the first precondition above. Read `plan.md`, `progress.md`, and `decisions.md`.
2. Check every hash in `progress.md` is on this branch (`git merge-base --is-ancestor <hash> HEAD`). If one is not, the branch was rewritten: stop and show the user which.
3. If the working tree is not clean, it holds partial work for the next step. Show `git status --short` and ask whether to keep it as the start of that step or stop so the user can deal with it. Never discard it yourself.
4. Continue from the first milestone `progress.md` does not have: planning (section 1 from the start, when `plan.md` is empty), plan approval (section 1 step 4), the next step (section 2 - from Code when partial work was kept, using `findings/step-N.md` to number any further review rounds), or feature completion (section 3).
5. Tell the user in one line where you are resuming.

## 1. Plan

1. Dispatch `planner` with the feature request verbatim.
2. If it returns questions, ask the user with AskUserQuestion, following `concise`, and send the answers back to the same agent. Repeat until it returns a plan.
3. Write the plan to `plan.md` exactly as returned.
4. Show the user the step list - one line per step with its title, size estimate, and security flag - and ask for approval. Edits go back to the planner, not into `plan.md` by hand. **No test or code is written before a yes.** On a yes, append `- Plan approved` to `progress.md`.

## 2. Each step

Work through the steps in order. For step N, first re-read the work files as Context says, then:

### Tests

Skip to Code for a step whose criteria say it adds no new behaviour.

1. Dispatch `test-writer` with the step's Goal, Interfaces, and Acceptance criteria from `plan.md`, and say the code does not exist yet: it writes against the interfaces, from the criteria, and the tests must fail.
2. Run the new tests yourself with the runner from `testing`. Each must fail because the behaviour or interface is missing - not on a syntax error, a wrong import path, or a broken fixture. Send any test failing for the wrong reason back to `test-writer`.

### Code

1. Dispatch `coder` with the step from `plan.md`, the paths of the new tests, and these rules: make the tests pass, never edit or delete a test, and report a test it believes is wrong instead of changing it.
2. If the coder reports a test as wrong, dispatch `code-reviewer` with that test, the coder's reason, and the step's acceptance criteria, and ask only whether the test matches the criteria. If it agrees the test is wrong, send the fix back to this step's `test-writer`; if not, send the coder back to make it pass.
3. Run every row of the Checks table in `coding` except the slow tests, in order. Send failures back to `coder`. If the same check still fails after three attempts, stop and show the user the failure.

### Review

Only BLOCKERs start another round. At most three rounds per step.

1. `git add -A`, then record `base=$(git write-tree)` - the tree the reviewers are about to see. In round 1 the review base is `HEAD`; in later rounds it is the `base` recorded in the round before, so the reviewers see only the fixes.
2. Dispatch `code-reviewer` and `security-reviewer` in one message, so they run in parallel. Give each: the review base, the step's Goal and Acceptance criteria, the path to `decisions.md` with the instruction not to re-raise anything in it, and for `security-reviewer`, `depth: deep` when the step is security-sensitive and `depth: light` otherwise.
3. Merge their findings by the rules in `.claude/skills/review/SKILL.md` section 4 and append them to `findings/step-N.md` under `## Round <r>`.
4. No BLOCKER: if there are SHOULD-FIX items, send them back to this step's `coder` once, with and the path to `decisions.md` - it fixes each or appends `- <finding> - <reason>` to `decisions.md` - then rerun the checks. Go to Commit.
5. BLOCKERs: send every BLOCKER and SHOULD-FIX item back to this step's `coder`, with and the same `decisions.md` rule, rerun the checks, and start the next round. If BLOCKERs remain after round 3, stop and show the user `findings/step-N.md`.

### Plan revision

If at any point the tests, the code, or a review shows the plan itself is wrong - an interface that cannot work, a missing step, a step that has to be split - dispatch `planner` with `plan.md`, the step number, and what went wrong. Show the user the revised remaining steps and ask for approval before continuing, as in section 1.

### Commit

`git add -A` and commit with the step's commit message from `plan.md`. `.feature/` is ignored, so the work files stay out of the commit. Append `- Step N: <short hash>` to `progress.md`.

## 3. Feature completion

1. Read `.claude/skills/review/SKILL.md` and follow its sections 1 to 4 against the default branch, with the slow tests included - this is the full test suite and the whole-branch review. When dispatching the agents, also give them the path to `decisions.md`, and ask `code-reviewer` and `security-reviewer` to look specifically for what only shows across steps: duplication, architectural drift, inconsistent error handling, and exposure that only appears when steps combine.
2. BLOCKERs go to a new `coder`, continued for every round after, in the same bounded loop as section 2, reviewing only the fixes, at most three rounds, then escalate. Commit the fixes as "Address feature review findings". Append `- Feature review: <short hash>` to `progress.md`, or `- Feature review: no fixes` when there was nothing to commit.
3. Push the branch and open a pull request with `gh pr create`. The body has: the feature summary from `plan.md`, one line per step, the won't-fix decisions copied from `decisions.md`, and every SHOULD-FIX or NIT from the final review that was left open.
4. Delete `.feature/<branch>/` and report the pull request URL, the number of steps, and any escalation the user resolved.
