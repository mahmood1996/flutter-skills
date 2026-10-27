---
name: feature-presentation
description: "Use when building presentation for an approved spec."
version: 1.0.0
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [flutter, tdd, clean-architecture, fakes]
    related_skills: [feature-build, feature-domain, feature-design-system, figma-to-flutter, test-driven-development]
---

# Feature Presentation

## Overview

Build the presentation layer (state holders, views, page wiring) test-first against Fakes. Follow `flutter-clean-architecture` for structure and DI, and `figma-to-flutter` for tokens.

Run this skill in its own chat session. All state lives in files (the spec, the slice,
`test/fakes/`), so a fresh session can resume at any time.

## When to Use

Use it when the user asks to build or continue the presentation layer of a feature from an approved spec.

## Preconditions (stop if any fails)

1. `specs/FEATURE.spec.yaml` exists, parses as YAML and has `status: approved`.
   If it is `draft`, stop and send the user back to `feature-spec`.
2. No `open_questions` entry blocks a component of this layer. If one does, stop and ask.
3. `build.domain` is `done` and `build.design_system` is `done` or `skipped`. It touches data only in the page wiring, so `build.data` should be `done` too.
4. `build.presentation` is not `done`. If it is, tell the user and stop.

## Start check

Run `flutter analyze` and `flutter test` for the layers already `done` in `build:`. If they fail, stop and tell the user; do not build on red.

## Slice

Extract only this layer's section of the spec: its components, their scenarios, and the fakes
those scenarios need. Copy each scenario exactly and add `status: pending`. Write it to
`specs/FEATURE.presentation.slice.yaml`. The slice is the ledger; the spec never gets statuses.
If the slice file exists, resume: continue at the first scenario that is not `refactored`.

```yaml
layer: presentation
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

## Layer Guide: presentation

Input: the spec's `presentation:` and `ui:` sections, the finished use cases, and the design
system from `figma-to-flutter`.
Output: `lib/features/FEATURE/presentation/` and `test/features/FEATURE/presentation/`.
This layer touches data only in the page wiring.

Build order: state holders, then views and feature widgets, then page wiring.

**State holders** (`presentation/blocs/`): `sealed class` state (and events for a Bloc) with
`final class` subtypes exactly as the spec lists. The state holder is a `final class` receiving
use cases through its constructor; it calls only use cases, never repositories. Scenarios use
`blocTest` (or plain stream expectations) with fake use cases and assert on the emitted state
sequence and on what the fake recorded. If the use case is a concrete `final class`, it cannot be
faked with `implements`; follow the spec's `fakes:` list, or if it names no fake, build the real
use case over `FakeRepository`. Say which you chose in the report.

**Views and feature widgets** (`presentation/widgets/`): shared widgets come from
`design_system/widgets/`; feature-specific ones live here. `FeatureView` switches exhaustively
over the sealed state. Widget tests pump `FeatureView` inside a `BlocProvider` holding a fake
state holder; never pump `FeaturePage`. Assert on what the user sees and, for actions, on what
the fake state holder recorded. If the state holder is a `final class`, use the real state holder
over fake use cases and say so in the report. Do not weaken class modifiers to make a fake easier.
Use design tokens only, and add `Semantics` to meaningful icons and images.

**Page wiring** (`presentation/pages/`): the only code without a test of its own. `FeaturePage`
composes the dependency graph inside `BlocProvider` (data source, repository implementation, use
cases, state holder), resolves shared dependencies through `GetIt.I`, and returns `FeatureView`.
Confirm it compiles, passes `flutter analyze`, and that route registration (if the spec mentions
one) is in place.

**Fidelity check** after the scenarios pass: compare the screen with the Figma screenshot or the
supplied screenshot (layout, spacing, colors, typography, radii) as `figma-to-flutter` Step 6
describes, and list any differences in the report.

Done when: every presentation scenario is `refactored`, `flutter analyze` is clean for
`presentation/`, no state holder imports a repository, data source or service, and `FeatureView`
is not in `pages/`.

## Gate and record

1. Check the ledger yourself: every scenario is `refactored`, each with `red_output`,
   `green_output` and `refactor_note`.
2. Run `flutter analyze` and `flutter test test/features/FEATURE/presentation`.
3. If green, set `build.presentation` to `done` in the spec. Touch nothing else in the spec except
   scenarios added beyond it (add them under their component).
4. If analyze or tests fail and you cannot fix it in a few attempts, stop, show the failure and
   ask the user. Never mark a layer `done` on red.

## Report and stop

Report: every scenario with its id, final status, `red_output` and `green_output`; scenarios
added beyond the spec; fakes created; anything unclear. Then tell the user the next step:
All layers are done. Run `feature-build` to do the finishing checks.

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
