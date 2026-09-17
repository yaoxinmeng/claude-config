---
name: write-code
description: Coding standards for any code that gets written or changed - DRY, readable names, abstractions only where they pay for themselves, files where the project expects them, and a cleanup pass with the simplifier agent. Load before writing, editing, or generating code in any language.
---

Write code a colleague can read in one pass and change without fear. Match the surrounding code's style, naming, and idiom before applying anything below.

Sections 1-5 are the standard. They are also the criteria to judge existing code against, so load them when reviewing as well as when writing. Section 6 is the cleanup pass for whoever is actually making the change; skip it if you are only reviewing.

## 1. Look before writing

- Find existing code that does the same or similar job (`grep` for the concept, not just the name). Reuse or extend it; never write a second copy.
- Find where the project keeps this kind of file (models, handlers, utils, tests, config) and put the new code there. Never invent a new top-level directory or a `utils`/`helpers`/`misc` dump when a named module fits.
- Read the neighbouring file to copy its conventions: import order, error handling, logging, docstring style, test layout.

## 2. DRY, with judgement

- Extract when the same logic appears twice and changes together. Inline when two blocks only look alike but vary for different reasons.
- Share through a named function or module, never copy-paste-tweak. Constants that appear in more than one place get one definition.
- Do not deduplicate across layers that should stay independent (for example, an API schema and a DB model that happen to match today).

## 3. Readable code

- Names say what a thing is or does; no abbreviations the reader has to decode, no `data`, `tmp`, `handle`, `process` without a noun.
- One function does one thing at one level of abstraction. If it needs a section comment to separate its parts, split it.
- Early returns over nested conditionals. No boolean parameters that switch behaviour; use two functions or an enum.
- Comments explain why, never what. Delete a comment that restates the code.
- Prefer the boring standard-library or framework-idiomatic way over a clever one.

## 4. Abstractions only when earned

- Introduce a class, interface, base class, or generic only when there are at least two concrete uses today, or the project spec names the variation point.
- No speculative parameters, config flags, or hooks "for later". YAGNI applies until a real second caller exists.
- Keep dependencies pointing inward: domain logic never imports from HTTP, CLI, or storage layers.
- Depend on the narrowest thing that works: pass the value, not the object that holds it.

## 5. Robust by default

- Validate at the boundary (input parsing, external calls), trust inside. No defensive checks on values the type system already guarantees.
- Fail loudly with a specific error; never swallow exceptions or return sentinel values that callers can forget to check.
- No hard-coded secrets, paths, or environment-specific values; read them from config.
- Every new behaviour gets a test in the project's existing test layout, covering the happy path and the failure the code guards against.

## 6. Finish

- Run the project's formatter, linter, type checker, and the relevant tests. Fix everything they report, including pre-existing failures you touched.
- Re-read the diff as a reviewer: every added line must be needed for the task. Remove leftover debug output, dead branches, and TODOs you can resolve now.
- If the diff is larger than the problem, or touches more than about 100 lines, ask the `simplifier` agent for what can be deleted or collapsed, and apply what it finds. If you cannot spawn agents - you are one yourself - do that pass inline instead: re-read each symbol you added, grep its call sites, and delete anything with one caller that does not earn its indirection.
