---
name: feature-data
description: "Use when building the data layer of an approved spec."
version: 1.0.0
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [flutter, tdd, clean-architecture, fakes]
    related_skills: [feature-build, feature-domain, test-driven-development, flutter-clean-architecture]
---

# Feature Data

## Overview

Build the data layer (DTOs, data sources, repository implementations) test-first against Fakes. Follow `flutter-clean-architecture` for structure and the error pipeline.

Run this skill in its own chat session. All state lives in files (the spec, the slice,
`test/fakes/`), so a fresh session can resume at any time.

## When to Use

Use it when the user asks to build or continue the data layer of a feature from an approved spec.

## Preconditions (stop if any fails)

1. `specs/FEATURE.spec.yaml` exists, parses as YAML and has `status: approved`.
   If it is `draft`, stop and send the user back to `feature-spec`.
2. No `open_questions` entry blocks a component of this layer. If one does, stop and ask.
3. `build.domain` is `done`.
4. `build.data` is not `done`. If it is, tell the user and stop.

## Start check

Run `flutter analyze` and `flutter test` for the layers already `done` in `build:`. If they fail, stop and tell the user; do not build on red.

## Slice

Extract only this layer's section of the spec: its components, their scenarios, and the fakes
those scenarios need. Copy each scenario exactly and add `status: pending`. Write it to
`specs/FEATURE.data.slice.yaml`. The slice is the ledger; the spec never gets statuses.
If the slice file exists, resume: continue at the first scenario that is not `refactored`.

```yaml
layer: data
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

## Layer Guide: data

Input: the spec's `data:` section, the finished domain contracts, and the `fakes:` entries for
`AppHttpClient` and data sources.
Output: `lib/features/FEATURE/data/` and `test/features/FEATURE/data/`.

Build bottom-up so each unit is tested against a fake of what it calls: DTOs, then data sources,
then repository implementations. Run the loop once per scenario for each unit, in slice order.

**DTOs** (`data/models/`): `@JsonSerializable()` `final class` with `part 'NAME.g.dart'`, fields
and nullability exactly as the spec. Never parse JSON by hand. Expose the conversion to the
domain model as a noun-named member (`domainModel`) unless the codebase uses `toDomainModel`.
Scenarios feed real JSON maps (no fake needed), parse with `fromJson`, convert, and assert on the
domain model. Run `dart run build_runner build --delete-conflicting-outputs` as soon as the
annotated class exists.

**Data sources** (`data/datasources/`, or `data/services/` for action-oriented gateways):
`abstract interface class` contract plus a `final class` implementation depending only on
`AppHttpClient`. Wrap calls in `try/catch` and throw the exceptions in the spec's `status_map`
(from `core/error/exceptions.dart`). Return DTOs, never domain models. One `status_map` entry per
scenario, success first: build `FakeAppHttpClient` to answer with the scenario's status and body,
call the data source, assert on the returned DTOs or the thrown exception, and on
`FakeAppHttpClient.requests` when the request shape is the behavior. If the shared client
contract lacks a method a scenario needs, stop and ask; changing `core/` is outside this layer.

**Repository implementations** (`data/repositories/`): `final class` implementing the domain
repository over the data source contract. Catch the data source exceptions, convert DTOs to
domain models, return `Either<Failure, T>`, and implement exactly the spec's
`exception_to_failure` table. Scenarios use a fake data source seeded with DTOs or set to throw;
assert on the returned `Either`. With `contract_tests: true`, write one shared suite per repository
interface and run it against both the fake and the real implementation (over a fake HTTP client).

Done when: every data scenario is `refactored`, `*.g.dart` files are generated and not
hand-edited, `flutter analyze` is clean for `data/`, and `data/` imports nothing from
`presentation/`.

## Gate and record

1. Check the ledger yourself: every scenario is `refactored`, each with `red_output`,
   `green_output` and `refactor_note`.
2. Run `flutter analyze`, `dart run build_runner build --delete-conflicting-outputs` and `flutter test test/features/FEATURE/data`.
3. If green, set `build.data` to `done` in the spec. Touch nothing else in the spec except
   scenarios added beyond it (add them under their component).
4. If analyze or tests fail and you cannot fix it in a few attempts, stop, show the failure and
   ask the user. Never mark a layer `done` on red.

## Report and stop

Report: every scenario with its id, final status, `red_output` and `green_output`; scenarios
added beyond the spec; fakes created; anything unclear. Then tell the user the next step:
Next: `feature-presentation` in a new session.

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
