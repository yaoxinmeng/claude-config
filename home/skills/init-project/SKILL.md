---
name: init-project
description: Gather requirements for a new project - product features, system architecture, program design, framework preferences, infrastructure - through iterative clarifying questions, then write the reference docs that code will be written against. Use when starting a project from scratch or when a project has code but no written spec.
disable-model-invocation: true
argument-hint: "[one-line description of the project]"
---

Requirements interview for: $ARGUMENTS

Goal: leave the repo with a `docs/spec/` that a competent engineer could build from without asking you anything. Every question you ask must close a gap that would otherwise make two engineers write materially different code. Never write code in this skill.

## 1. Read before asking

Check what already exists so you never ask what the repo can answer:

- `ls`, `README.md`, `CLAUDE.md`, `docs/`, manifests (`pyproject.toml`, `package.json`, `go.mod`, `Cargo.toml`, IaC files)
- If there is code: skim entry points, data models, and the CI config

Summarize what you learned in under ten lines and state which of the five areas below are already answered.

## 2. Interview, one area per round

Cover the areas in this order. Each is a round: ask, record, then confirm the round's summary before moving on. Skip anything already answered.

1. **Product** - who the users are, the problem, the features for the first release, explicit non-goals, success criteria, hard constraints (deadline, budget, compliance, offline, accessibility, locale).
2. **System architecture** - components and their boundaries, sync vs async communication, data flow, external systems and integrations, multi-tenancy, expected scale (users, requests, data volume), availability and latency targets.
3. **Program design** - domain model and key entities, API shape (REST, GraphQL, gRPC, CLI, library), persistence and schema ownership, auth model, error handling and validation strategy, testing strategy, code layout conventions the user already has.
4. **Frameworks and tooling** - language and version, framework per component, ORM, package manager, lint and format tools, test runner, anything the user wants excluded and why.
5. **Infrastructure** - hosting target, environments, deployment method, CI/CD, secrets management, observability (logs, metrics, tracing), backups, cost ceiling, IaC preference.

Rules for each round:

- Use AskUserQuestion, following the `concise` skill: one dimension per question, concrete parameterised options, recommended option first with a one-line reason, and an explicit default so the user can skip.
- Ask at most four questions per call. Batch what is independent; sequence what depends on an earlier answer.
- When the user says "you decide", decide, state the choice and reason in one line, and record it as an **assumption** (see section 4). Do not ask again.
- When an answer contradicts an earlier one, point it out immediately and ask which stands.
- When an answer is vague ("fast", "scalable", "secure"), ask for the number or the concrete scenario behind it.
- End every round with a summary of that area under ten lines and ask the user to confirm or correct it.

## 3. Iterate until no doubts remain

After round 5, reread everything recorded and run this checklist. Any "no" becomes a follow-up question; repeat until all pass.

- Could a feature in Product be implemented two different ways that the user would judge differently? If so, the acceptance criterion is missing.
- Does every component in Architecture have an owner for its data and a stated way of talking to its neighbours?
- Does every entity in Design have a stated lifecycle (created by whom, mutated by whom, deleted when)?
- Does every framework choice have a reason recorded, even if the reason is "user preference"?
- Does Infrastructure name where secrets live and how a deploy is rolled back?
- Is there any decision I made silently rather than as a recorded assumption?
- Would I need to ask the user anything on day one of coding? If yes, ask it now.

Stop when the checklist passes and the user confirms the final consolidated summary. The user may also stop early; then everything unresolved goes to `docs/spec/open-questions.md` rather than being guessed.

## 4. Hand the confirmed requirements to `docs-writer`

You gather requirements; `docs-writer` writes the prose. Do not write these files yourself.

First, record everything confirmed in the interview to `docs/spec/.requirements.md`: every answer per area, every assumption you made on the user's behalf, and every unresolved question. Write it as notes, not prose - the agent turns it into the pages. It is the only context the agent has, so an answer missing here is an answer lost.

Then invoke the `docs-writer` agent once, with a prompt that:

- Names `docs/spec/.requirements.md` as the source of truth, and states that this is a spec for a project that may not be built yet - the requirements record is what it documents, not existing code.
- Lists the files to produce, by including the table below and the rule under it verbatim.

One invocation, not one per file: the pages cross-reference each other and must be written together.

When the agent returns, read what it wrote and check it against the interview record - every confirmed decision present, no invented requirement, no placeholder left. Send corrections back to the same agent rather than editing the files yourself. Delete `docs/spec/.requirements.md` once the pages are correct.

| File | Contents |
|---|---|
| `docs/spec/README.md` | What the project is in two sentences, index of the files below, and a "How to use these docs" note telling agents to read the relevant spec file before implementing a feature and to update it when the design changes. |
| `docs/spec/product.md` | Users, problem, feature list for the first release with one acceptance criterion each, non-goals, constraints, success metrics. |
| `docs/spec/architecture.md` | Component list with responsibilities, a Mermaid diagram of components and data flow, integration contracts, scale and availability targets, multi-tenancy model. |
| `docs/spec/design.md` | Domain model (entities, fields, relationships, lifecycle), API surface, persistence and migrations, auth, error handling, validation, testing strategy, code layout. |
| `docs/spec/stack.md` | Table of `component · language/framework · version · reason`. Tooling for lint, format, test, package management. Explicit exclusions with reasons. |
| `docs/spec/infrastructure.md` | Hosting, environments, deploy and rollback procedure, CI/CD stages, secrets, observability, backups, cost ceiling, IaC tool. |
| `docs/spec/assumptions.md` | Every decision made on the user's behalf: `decision · reason · what would change it`. Empty file is fine; missing file is not. |
| `docs/spec/open-questions.md` | Only if the user stopped early. Each item: question, why it matters, what is blocked until it is answered. |
| `docs/decisions/NNNN-<slug>.md` | One ADR per consequential choice (framework, database, hosting, auth model, anything with a rejected alternative). Sections: Context, Decision, Alternatives considered, Consequences. Numbered from 0001, immutable once merged. |

No file is written for an area with nothing to say beyond a heading; it is folded into `README.md` with one line explaining why.

## 5. Point CLAUDE.md at the spec

Every future session must find the spec without being told. Create or update the project's `CLAUDE.md` (repo root):

- If it does not exist, create it with a `## Project spec` section only.
- If it exists, add or replace a `## Project spec` section. Leave every other section untouched.

The section, in under ten lines, says:

- `docs/spec/README.md` is the index; read the spec file for an area before implementing anything in it.
- Update the spec in the same change as the code when a design changes; add an ADR under `docs/decisions/` for any choice with a rejected alternative.
- Unknowns go to `docs/spec/open-questions.md`, never guessed.
- A one-line pointer to the stack (`docs/spec/stack.md`) so agents don't pick a framework from memory.

Do not restate spec content in `CLAUDE.md` - it would drift. Point, don't copy.

## 6. Finish

Report in under fifteen lines: files written, number of ADRs, number of assumptions, number of open questions, and whether `CLAUDE.md` was created or updated. Then ask one question: whether to commit `docs/` and `CLAUDE.md` now. Do not commit without a yes.

End by telling the user to run `/init-claude-configs` next. It reads `docs/spec/stack.md` and `docs/spec/design.md` and generates the project's `.claude/` skills and agents for that stack, so the first coding session starts with the right runner, layout, and checks instead of guessing them.
