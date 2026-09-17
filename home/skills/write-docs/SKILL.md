---
name: write-docs
description: Standards for any prose documentation that gets written or changed - how-to guides, architecture explanations, ADRs, specs, READMEs. House style, one purpose per page type, and what never belongs in a doc. Load before writing or editing anything under `docs/` or a README.
---

Write documentation a new teammate could follow. Match the voice, heading depth, and formatting of the pages around the one you are writing before applying anything below.

## 1. Look before writing

- Read the code before writing about it. Never document intended behaviour; document what the code does. If they differ, report the gap instead of papering over it. The exception is `docs/spec/` (see Page types): a spec records agreed requirements for code that may not exist yet, and its source is the requirements record you were handed, never guesswork.
- Check `docs/` for an existing page on the topic. Update it in place rather than adding a second page that will drift from the first.
- Verify every command and snippet by running it. An untested snippet is worse than no snippet. When the code does not exist yet, write no snippet at all rather than one you cannot run.

## 2. House style

- **Lead with the answer.** First sentence says what the reader gets. No "In this document we will explore".
- **Second person, active voice, present tense.** "Run the migration", not "The migration should be run".
- **Short.** A how-to that runs past one screen is two how-tos.
- **Every claim concrete.** Real commands, real paths, real payloads - no `<your-value>` where an actual example would do.
- **Say why, not what.** The code shows what it does. Docs exist for the reasons a reader can't recover from reading it: the constraint, the tradeoff, the thing that bit us.
- **No marketing.** No "simply", "just", "easy", "powerful", "seamless", "robust".
- **Own the caveats.** Known limitations and sharp edges go in the doc, not in a ticket.
- **Length follows content.** A page is as long as what it has to say, never padded to look thorough. Cut a heading with nothing under it.
- **No filler sections.** No "Introduction" restating the title, no "Conclusion" restating the page, no "Future work" you invented.
- **Say it once.** Link, or point to `path:line`, instead of repeating something another page already covers.
- **Describe what is, not what you did.** "The queue retries 3 times", not "I added retries to the queue".
- **Gloss unfamiliar terms on first use.** Spell out an acronym or internal name the first time it appears: "ADR (architecture decision record)".
- **Table or list over prose** whenever the content is a set of parallel items.

## 3. Page types (one purpose per page - never mix)

- `docs/spec/*.md` - the requirements the code is built against, one area per file: `product.md`, `architecture.md`, `design.md`, `stack.md`, `infrastructure.md`, `assumptions.md`, `open-questions.md`, plus a `README.md` index. These are written from a requirements record, and kept current afterwards. When a design changes, update the spec file in the same change as the code. Never invent a requirement to fill a gap - add it to `open-questions.md` instead.
- `docs/how-to/*.md` - a task, start to finish, numbered steps, a stated end state. Title is the task: "Add a database migration".
- `docs/explanation/*.md` - how a subsystem fits together and why it's shaped that way. No steps. Include a diagram in Mermaid where structure matters. Explains the code as built; the spec says what was asked for. If they disagree, report it, don't reconcile it silently.
- `docs/decisions/NNNN-*.md` - ADRs, numbered from 0001. Sections: Context, Decision, Alternatives considered, Consequences. Immutable once merged; supersede, never rewrite.
- `README.md` - what this repo is, how to run it, where to go next. Under 100 lines. Everything else is a link.

## 4. Never

- Document a private function, or restate a signature in prose.
- Hand-write anything under `docs/reference/` - it's generated and your edits are erased.
- Leave a TODO in a doc. Either find the answer or write what's unknown and why.
- Edit application code to make a doc true. Report the mismatch instead.
