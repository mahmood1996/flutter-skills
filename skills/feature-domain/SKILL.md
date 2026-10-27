---
name: feature-domain
description: "Use when building the domain layer of an approved spec."
version: 1.0.0
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [flutter, tdd, clean-architecture, fakes]
    related_skills: [feature-build, feature-data, test-driven-development, flutter-clean-architecture]
---

# Feature Domain

## Overview

Build the domain layer (failures, models, repository contracts, use cases) test-first against Fakes. Follow `flutter-clean-architecture` for structure, class modifiers and the error pipeline.

Run this skill in its own chat session. All state lives in files (the spec, the slice,
`test/fakes/`), so a fresh session can resume at any time.

## When to Use

Use it when the user asks to build or continue the domain layer of a feature from an approved spec.

## Preconditions (stop if any fails)

1. `specs/FEATURE.spec.yaml` exists, parses as YAML and has `status: approved`.
   If it is `draft`, stop and send the user back to `feature-spec`.
2. No `open_questions` entry blocks a component of this layer. If one does, stop and ask.
3. No previous layer is required.
4. `build.domain` is not `done`. If it is, tell the user and stop.

## Start check

Run `flutter analyze` and `flutter test` for any layer already `done` in `build:` (design_system if present). If they fail, stop and tell the user; do not build on red.

## Slice

Extract only this layer's section of the spec: its components, their scenarios, and the fakes
those scenarios need. Copy each scenario exactly and add `status: pending`. Write it to
`specs/FEATURE.domain.slice.yaml`. The slice is the ledger; the spec never gets statuses.
If the slice file exists, resume: continue at the first scenario that is not `refactored`.

```yaml
layer: domain
components:
  - name: ComponentName
    scenarios:
      - id: XX-1
        # ...the scenario exactly as in the spec...
        status: pending      # pending -> red -> green -> refactored
        red_output: ""       # one failing line
        green_output: ""     # one passing line
        refactor_note: ""
```

## The Loop (one scenario per pass, in slice order)

Load `test-driven-development` first. It explains the reasoning and holds the Fakes rules.

1. Take the first scenario that is not `refactored`. At most ONE scenario may be `red` or `green`.
2. **RED.** Write exactly ONE test. Run it. It must fail on the assertion.
   A brand-new unit does not compile: that is the expected first RED. Create only the minimal
   stubs that fix the compile errors (class, signature, interface). A stub returns a wrong
   default value, never throws, and has no logic. Run again until the test fails on the
   assertion. Set `status: red` and save the one-line failing output as `red_output`.
3. **GREEN.** Write the minimal code (hardcoding is allowed). Run that test. Set
   `status: green` and save the one-line passing output as `green_output`.
4. **REFACTOR.** Run the layer's tests, clean up THIS scenario's code, run the layer's tests
   again. Nothing to clean up? Write that in `refactor_note`. Set `status: refactored`.
5. Only then start the next scenario.

Rules:
- Status moves only `pending -> red -> green -> refactored`. Never skip a step (no
  `red -> refactored`), never update in bulk, always update right after the run that justifies it.
- Never write a second test while a scenario is `red` or `green`. Never refactor "later".
- Code written before its test must be deleted.
- Fakes are hand-written in `test/fakes/`. No mocking packages, no `verify(...)`.
- If you notice you batched, delete the tests and code of every scenario that lacks a
  recorded RED, reset those statuses to `pending`, and redo them one at a time.

## Layer Guide: domain

Input: the spec's `domain:` section and the `fakes:` entries for repository interfaces.
Output: `lib/features/FEATURE/domain/` and `test/features/FEATURE/domain/`.
Pure Dart: no Flutter UI, no networking packages, no imports from `data/` or `presentation/`.

Build:
1. **Failures**: reuse `core/error/failures.dart` entries marked `reuse: true`. Add new ones as
   `final class X extends Failure`.
2. **Models**: immutable `final class` entities, `const` constructors, no JSON logic, nullability
   exactly as the spec says. Add a dedicated test only if a model has logic (equality, derived
   getters); otherwise the use case tests cover it.
3. **Repository contracts**: `abstract interface class` with the spec's methods. Create them when
   the first use case test needs them; the compile error is the expected RED.
4. **Use cases**: `final class` with a `call()` method delegating to the repository.

Test each use case against `FakeRepository`. For the first scenario, create only what fails to
compile, minimally, then continue at the assertion failure.

Done when: every domain scenario is `refactored`, `flutter analyze` is clean for `domain/`, and
`grep -rnE "package:flutter|dio|http|data/|presentation/" lib/features/FEATURE/domain/` is empty.

## Gate and record

1. Check the ledger yourself: every scenario is `refactored`, each with `red_output`,
   `green_output` and `refactor_note`.
2. Run `flutter analyze` and `flutter test test/features/FEATURE/domain`.
3. If green, set `build.domain` to `done` in the spec. Touch nothing else in the spec except
   scenarios added beyond it (add them under their component).
4. If analyze or tests fail and you cannot fix it in a few attempts, stop, show the failure and
   ask the user. Never mark a layer `done` on red.

## Report and stop

Report: every scenario with its id, final status, `red_output` and `green_output`; scenarios
added beyond the spec; fakes created; anything unclear. Then tell the user the next step:
Next: `feature-data` in a new session.

## Context rule

If the session is getting long, stop after the current scenario is `refactored`, say which
scenario is next, and tell the user to continue in a new session with this same skill. The
slice file is the memory.

## Common Mistakes

- Starting from a draft spec.
- Writing several tests first, or filling in statuses afterwards.
- Jumping from `red` to `refactored`, or recording no `green_output`.
- A stub that throws or contains logic.
- Refactoring once at the end of the layer.
- Marking the layer `done` without running analyze and tests yourself.
- A fake that mirrors the real implementation's logic.
- Editing spec sections other than `build:` and added scenarios.
