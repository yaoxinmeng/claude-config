---
name: write-tests
description: Standards for any test that gets written or changed, in any language or framework - what deserves a test, one behaviour per test, assertions that can fail, mocking only at boundaries, and tests that pass alone and in any order. Load before writing or editing tests, or when asked to raise coverage.
---

Write tests that fail for exactly one reason and say which behaviour broke. Match the repo's existing test layout, naming, and fixtures before applying anything below.

## 1. Look before writing

- Find how tests run here: the project's task runner, lockfile, or CI config, not a command you assume. Never add a test framework, plugin, or dependency to make a test possible.
- Read the project's test configuration for markers, plugins, timeouts, and coverage settings already in force.
- Read the existing tests next to the target and every shared setup file on the path (`conftest.py`, `setup.ts`, fixtures module, test base class). Reuse those fixtures and helpers before defining new ones.
- Put the new test where this project keeps tests for that file - a mirrored test tree or a test file beside the module. Copy whichever the repo uses; never introduce a second layout.
- Read the code under test in full, then its direct callers, to learn which behaviour actually matters.

## 2. What to test

- One behaviour per test: the happy path, each documented failure (the error it raises, the error value it returns), and each edge the code visibly guards against - empty input, boundary value, missing key.
- Test through the public interface. Never reach into a private helper or internal state to hit a branch; if a branch is unreachable publicly, report it as dead code instead.
- Skip accessors, plain data fields, and anything the type system already proves.
- Don't chase a coverage number. A line covered by a test that cannot fail is worse than an uncovered line.

## 3. Assertions that can fail

- Assert on the value the behaviour produces, not that something ran. No test whose only assertion is "not null", a bare smoke call, or a snapshot nobody reads.
- One assertion target per test. Assert the specific error type and message, never a catch-all error class.
- Write the expected value as a literal. No logic in tests: no loops, conditionals, or helpers that recompute what the code under test computes.
- If the assertion would still pass with the code under test removed or mocked out, the test is worthless - rewrite it.

## 4. Names and shape

- Name the test after the behaviour: unit, scenario, expected result, for example `test_parse_config_missing_file_raises_file_not_found`.
- Arrange, act, assert in that order, separated by blank lines, with no comments labelling the sections.
- Table-driven or parameterised cases when several inputs share one expectation, each case labelled. Separate tests when the expectations differ.

## 5. Isolation

- Mock only at the boundary: network, filesystem, clock, randomness, subprocess, third-party SDK. Never mock the module under test or its pure collaborators.
- Use the framework's temporary-directory facility for files, never the real filesystem or a shared fixture path.
- No sleeps, no real waiting. Freeze or inject the clock using the repo's existing fixture if there is one.
- No ordering between tests, no shared mutable state. Every test must pass alone and in any order.

## 6. Finish

- Run the tests you wrote, then the whole suite, to confirm nothing else broke.
- If a test you wrote exposes a real bug, leave the failing test in place and report the bug. Never weaken an assertion to make it pass.
- When you change behaviour, update the test that guards it rather than deleting or loosening it.
