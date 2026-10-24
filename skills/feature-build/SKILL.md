---
name: feature-build
description: "Use when building a Flutter feature from an approved spec."
version: 1.0.0
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [flutter, tdd, clean-architecture, orchestration, fakes]
    related_skills: [feature-spec, flutter-clean-architecture, test-driven-development, figma-to-flutter]
---

# Feature Build

## Overview

Implement an approved spec one layer at a time. Each layer is built test-first against
**Fakes** of the layer below it, verified, and recorded before the next layer starts.
The spec comes from the `feature-spec` skill.

**Core principle:** the orchestrator never trusts a sub-agent's "all green". It runs
analyze and tests itself after every layer.

## When to Use

Use it whenever the user:
- asks to implement or build a Flutter feature from a spec;
- says "build the feature", "implement the spec" or "continue the feature build";
- wants to resume a partly built feature.

Use it even if the user does not mention TDD or layers. If there is no approved spec yet,
use `feature-spec` first.

## Preconditions (check first, stop if any fails)

1. `specs/FEATURE.spec.yaml` exists and parses as YAML.
2. `status: approved`. If it is `draft`, stop and send the user back to `feature-spec`.
   Do not "just start".
3. No `open_questions` block a component you are about to build. If one does, stop and ask.
4. Read `build:` in the spec and skip layers already marked `done`, so an interrupted
   build resumes.

## Order and Gates

```
design_system -> domain -> data -> presentation
```

`design_system` depends on nothing else and may run in parallel with `domain`. It must be
`done` before `presentation` starts. For design_system, load `figma-to-flutter` and follow
it (skip if the spec adds no new tokens or shared widgets).

For each layer:

1. **Slice.** Extract only that layer's section of the spec: its components, their
   scenarios, and the fakes those scenarios need. Write it to
   `specs/FEATURE.LAYER.slice.yaml`.
2. **Delegate.** Dispatch one sub-agent per layer (see Delegation below) with the matching
   layer guide from the end of this skill pasted into its context.
3. **Gate.** When the sub-agent returns, run the checks yourself with the terminal tool:
   `flutter analyze` and `flutter test test/features/FEATURE/LAYER`, plus
   `dart run build_runner build --delete-conflicting-outputs` after DTO work.
4. **Record.** If green, set that layer to `done` under `build:` in the spec. Touch no
   other part of the spec.
5. **Stop on red.** If analyze or tests fail and the layer cannot fix it in a few
   attempts, stop, show the failure, and ask the user. Never move past red.

### Delegation

```python
delegate_task(
    goal="Implement the LAYER layer of FEATURE using strict TDD with Fakes",
    context="""
    Follow the test-driven-development and flutter-clean-architecture skills.
    Read the spec slice at specs/FEATURE.LAYER.slice.yaml.
    Work one scenario at a time, in spec order: write one test, run it and watch it fail
    for the expected reason, write minimal code, run it green, run the layer's tests.
    Fakes: hand-written, in test/fakes/, created only when a scenario needs them.
    No mock packages. Names: nouns for query methods, no type names ending in er/or.

    LAYER GUIDE:
    [paste the matching "Layer Guide" section from the feature-build skill]

    Project test command: flutter test test/features/FEATURE/LAYER
    Report: scenarios done, scenarios added beyond the slice, fakes created, anything unclear.
    """,
    toolsets=['terminal', 'file']
)
```

Give each sub-agent only its slice and its own layer guide, never the whole spec or the
other layers' guides.

## TDD with Fakes (applies to every layer)

The build order is inside-out, but each unit is still tested London School style: in
isolation, against **Fakes of its direct collaborators**.

- Use cases against `FakeRepository`.
- Repository implementations against a fake data source.
- Data sources against `FakeAppHttpClient`.
- State holders against fake use cases.
- Views against a fake state holder inside a `BlocProvider`.

**Cycle.** One scenario at a time, in spec order, vertical slices only. For each scenario:
write one test, run it, confirm it fails for the expected reason, write the minimal code,
run it, confirm green, run the layer's tests for regressions, then refactor. Never write
several tests before the first implementation. `test-driven-development` defines the cycle;
follow it exactly, including deleting code written before its test.

**Fakes.**
- Live in `test/fakes/NAME.dart`, one class per file, `final class FakeName implements Interface`.
  Every interface is an `abstract interface class`, so `implements` always works.
- Create a fake only when the first scenario needs it, with only the behaviors that scenario
  needs. Grow it scenario by scenario, within the contract listed under `fakes:` in the spec.
- Seed data through constructor parameters. Configure failure through a constructor
  parameter or a settable noun-named field (`failure`, `exception`), not a verb method.
- Expose what it recorded through noun getters (`requests`, `callCount`, `savedNotes`).
- Assert on outcomes and on what the fake recorded. Do not use mocking packages
  (`mockito`, `mocktail`) and do not use interaction expectations.
- A fake contains no logic beyond what a scenario needs. If a fake gets clever, simplify it.
- If `contract_tests: true`, write one shared test suite per repository interface and run it
  against both the fake and the real implementation (real one over a fake HTTP client), so the
  fake cannot drift.

**Scenarios are a minimum.** If a layer discovers a missing edge case, it writes the
scenario, runs the cycle for it, and reports it. After the layer passes, add it to the spec
under that component (ids continue the sequence, for example `GO-4`).

## Project Rules

- Folder structure, class modifiers, error pipeline and DI: follow
  `flutter-clean-architecture`. Do not restate or override it.
- Design tokens and shared widgets: follow `figma-to-flutter`. No hardcoded colors, radii,
  spacing, paddings, text styles or asset paths in widgets.
- Naming: query methods and getters are nouns (`orders`, `label`); type names do not end in
  "er" or "or". Where the architecture skill's examples use a verb-named method (for example
  `toDomainModel`), apply the noun rule (`domainModel`) unless the codebase already uses the
  other form.
- Production code is never written without a failing test first. Generated code
  (`*.g.dart`) and the `FeaturePage` DI wiring are the only exceptions; cover the wiring
  through the view and state holder tests.

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

Cycle for each use case, scenarios in order: write the test for the first scenario (it needs the
use case, the repository contract and `FakeRepository`), create only what fails to compile,
minimally; run it and confirm it fails on the assertion; write the minimum code to pass
(hardcoding is allowed, the next scenario forces generalization); run the use case tests, then
the domain folder; repeat; refactor when green.

Done when: every domain scenario passes, `flutter analyze` is clean for `domain/`, and
`grep -rnE "package:flutter|dio|http|data/|presentation/" lib/features/FEATURE/domain/` is empty.

## Layer Guide: data

Input: the spec's `data:` section, the finished domain contracts, and the `fakes:` entries for
`AppHttpClient` and data sources.
Output: `lib/features/FEATURE/data/` and `test/features/FEATURE/data/`. Domain must be `done`.

Build bottom-up so each unit is tested against a fake of what it calls: DTOs, then data sources,
then repository implementations.

**DTOs** (`data/models/`): `@JsonSerializable()` `final class` with `part 'NAME.g.dart'`, fields
and nullability exactly as the spec. Never parse JSON by hand. Expose the conversion to the
domain model as a noun-named member (`domainModel`) unless the codebase uses `toDomainModel`.
Scenarios feed real JSON maps (no fake needed), parse with `fromJson`, convert, and assert on the
domain model. Run `build_runner` as soon as the annotated class exists.

**Data sources** (`data/datasources/`, or `data/services/` for action-oriented gateways):
`abstract interface class` contract plus a `final class` implementation depending only on
`AppHttpClient`. Wrap calls in `try/catch` and throw the exceptions in the spec's `status_map`
(from `core/error/exceptions.dart`). Return DTOs, never domain models. Endpoint method and path
come from the spec. One `status_map` entry per scenario, success first: build `FakeAppHttpClient`
to answer with the scenario's status and body, call the data source, assert on the returned DTOs
or the thrown exception, and on `FakeAppHttpClient.requests` when the request shape is the
behavior. Add a small `RecordedRequest` value class to `test/fakes/` only when a scenario needs
it. If the shared client contract lacks a method a scenario needs, stop and ask; changing
`core/` is outside this layer.

**Repository implementations** (`data/repositories/`): `final class` implementing the domain
repository over the data source contract. Catch the data source exceptions, convert DTOs to
domain models, return `Either<Failure, T>`, and implement exactly the spec's
`exception_to_failure` table. Scenarios use a fake data source seeded with DTOs or set to throw;
assert on the returned `Either`. For streams, assert that errors arrive as `Left` and the stream
stays alive. With `contract_tests: true`, keep the shared suite green on both fake and real; a
scenario that passes on the real one but not the fake means the fake drifted.

Done when: every data scenario passes, `*.g.dart` files are generated and not hand-edited,
`flutter analyze` is clean for `data/`, and `data/` imports nothing from `presentation/`.

## Layer Guide: presentation

Input: the spec's `presentation:` and `ui:` sections, the finished use cases, and the design
system from `figma-to-flutter`.
Output: `lib/features/FEATURE/presentation/` and `test/features/FEATURE/presentation/`.
Domain and design system must be `done`. This layer touches data only in the page wiring.

Build order: state holders, then views and feature widgets, then page wiring.

**State holders** (`presentation/blocs/`): `sealed class` state (and events for a Bloc) with
`final class` subtypes exactly as the spec lists. The state holder is a `final class` receiving
use cases through its constructor; it calls only use cases, never repositories. Scenarios use
`blocTest` (or plain stream expectations) with fake use cases and assert on the emitted state
sequence and on what the fake recorded. If the use case is a concrete `final class`, it cannot be
faked with `implements`; follow the spec's `fakes:` list, or if it names no fake, build the real
use case over `FakeRepository`. Do not decide silently: say which you chose in the report.

**Views and feature widgets** (`presentation/widgets/`): shared widgets come from
`design_system/widgets/`; feature-specific ones live here. `FeatureView` switches exhaustively
over the sealed state. Widget tests pump `FeatureView` inside a `BlocProvider` holding a fake
state holder; never pump `FeaturePage`. Assert on what the user sees and, for actions, on what
the fake state holder recorded. If the state holder is a `final class`, it cannot be faked from
another library: use the real state holder over fake use cases and say so in the report. Do not
weaken the class modifiers to make a fake easier. Use design tokens only, and add `Semantics` to
meaningful icons and images.

**Page wiring** (`presentation/pages/`): `FeaturePage` composes the dependency graph inside
`BlocProvider` (data source, repository implementation, use cases, state holder), resolves
shared dependencies through `GetIt.I`, and returns `FeatureView`. It has no dedicated test;
confirm it compiles, passes `flutter analyze`, and that route registration (if the spec mentions
one) is in place.

Fidelity check after the scenarios pass: compare the screen with the Figma screenshot or the
supplied screenshot (layout, spacing, colors, typography, radii) as `figma-to-flutter` Step 6
describes, and list any differences in the report.

Done when: every presentation scenario passes, `flutter analyze` is clean for `presentation/`, no
state holder imports a repository, data source or service, and `FeatureView` is not in `pages/`.

## Finishing

When all layers are `done`:
1. Run `dart format .`, `flutter analyze` and the full `flutter test`.
2. Run the dependency check from `figma-to-flutter` on `lib/design_system/`.
3. Report: layers built, scenarios implemented per component, scenarios added beyond the spec,
   fakes created, anything the user should review.

## Common Mistakes

- Starting from a draft spec.
- Giving a sub-agent the whole spec or the other layers' guides.
- Writing all of a layer's tests first, then all of its code.
- Trusting a sub-agent's "all green" without running analyze and tests.
- A fake that mirrors the real implementation's logic.
- Using a mock package "just for this one test".
- Editing spec sections other than `build:` and added scenarios.
