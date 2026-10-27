---
name: feature-design-system
description: "Use when building design_system for an approved spec."
version: 1.0.0
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [flutter, tdd, clean-architecture, fakes]
    related_skills: [feature-build, figma-to-flutter, test-driven-development]
---

# Feature Design System

## Overview

Build the shared tokens and widgets a feature needs: visuals per `figma-to-flutter`, behavior test-first.

Run this skill in its own chat session. All state lives in files (the spec, the slice,
`test/fakes/`), so a fresh session can resume at any time.

## When to Use

Use it when the user asks to build or continue the design system layer of a feature from an approved spec.

## Preconditions (stop if any fails)

1. `specs/FEATURE.spec.yaml` exists, parses as YAML and has `status: approved`.
   If it is `draft`, stop and send the user back to `feature-spec`.
2. No `open_questions` entry blocks a component of this layer. If one does, stop and ask.
3. No dependency on other layers. It may run before or in parallel with `domain`.
4. `build.design_system` is not `done`. If it is, tell the user and stop.

## Start check

Run `flutter analyze` and `flutter test test/design_system` if they exist. If they fail, stop and tell the user; do not build on red.

## Slice

Extract only this layer's section of the spec: its components, their scenarios, and the fakes
those scenarios need. Copy each scenario exactly and add `status: pending`. Write it to
`specs/FEATURE.design_system.slice.yaml`. The slice is the ledger; the spec never gets statuses.
If the slice file exists, resume: continue at the first scenario that is not `refactored`.

```yaml
layer: design_system
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

## Layer Guide: design_system

Input: the spec's `ui.screens[].widgets.design_system` and `inputs.ui`. Follow `figma-to-flutter`
for tokens, assets, widgets, naming and the dependency rules. Output: `lib/design_system/` and
`test/design_system/widgets/`. If the spec adds no new tokens or shared widgets, set
`build.design_system: skipped` and stop.

Build in two parts.

**Part 1: tokens, assets and widget shells (not test-first).** Colors, radii, spacing, paddings,
font styles, assets and widget layout are visual. Build them as `figma-to-flutter` Steps 2 to 5
say and verify them by its Step 6 (screenshot comparison, dependency check, `flutter analyze`).
Do not write assertions for colors or pixels.
The shell has the full constructor (content, callbacks, variant flags) and the final visual
layout, but **behavior is not implemented**: callbacks are accepted and not wired, and flags such
as `isLoading` or `isEnabled` have no effect yet. This keeps the first behavior test red.

**Part 2: behavior, test-first.** Derive scenarios from each shared widget's constructor
parameters and write them into the slice as `pending`. Ids are `DS-<Widget>-<n>`. Typical ones:
- a callback fires when the widget is tapped (assert on a value the test captured);
- a disabled widget ignores taps;
- a loading widget shows a progress indicator and ignores taps;
- a variant shows the expected content;
- a meaningful icon or image has a `Semantics` label.
One behavior per scenario, simplest first. Pump the widget inside a `MaterialApp` and assert on
what the user sees or on what the test captured. Never import a Bloc, model or repository.
Run the loop once per scenario, then the widget's behavior is wired in the shell.

Done when: every design_system scenario is `refactored`, the dependency check from
`figma-to-flutter` Section 2 is empty, `flutter analyze` is clean for `lib/design_system/`, and
no hardcoded color, radius, spacing, padding, text style or asset path remains in a widget.

## Gate and record

1. Check the ledger yourself: every scenario is `refactored`, each with `red_output`,
   `green_output` and `refactor_note`.
2. Run `flutter analyze`, `flutter test test/design_system` and the dependency check from `figma-to-flutter`.
3. If green, set `build.design_system` to `done` in the spec. Touch nothing else in the spec except
   scenarios added beyond it (add them under their component).
4. If analyze or tests fail and you cannot fix it in a few attempts, stop, show the failure and
   ask the user. Never mark a layer `done` on red.

## Report and stop

Report: every scenario with its id, final status, `red_output` and `green_output`; scenarios
added beyond the spec; fakes created; anything unclear. Then tell the user the next step:
`design_system` must be `done` or `skipped` before `presentation`. Next: `feature-domain` in a new session.

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
