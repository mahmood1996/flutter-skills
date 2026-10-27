---
name: feature-build
description: "Use when asked to build a Flutter feature from a spec."
version: 1.0.0
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [flutter, tdd, clean-architecture, fakes]
    related_skills: [feature-spec, feature-design-system, feature-domain, feature-data, feature-presentation]
---

# Feature Build

## Overview

Router and finisher for building a Flutter feature from an approved spec. It never writes
production or test code. Each layer is built by its own skill, **in its own chat session**, so
no session accumulates the history of the others.

```
design_system -> domain -> data -> presentation
```

`design_system` may run before or in parallel with `domain`. It must be `done` or `skipped`
before `presentation`.

## When to Use

Use it when the user says "build the feature", "implement the spec", "continue the feature build",
or asks what to do next. If there is no approved spec, use `feature-spec` first.

## Steps

1. Check `specs/FEATURE.spec.yaml`: it exists, parses as YAML and has `status: approved`.
   If it is `draft`, stop and send the user to `feature-spec`.
2. Read `build:`. Each layer is `pending`, `done` or `skipped` (`design_system` only).
3. If a slice file `specs/FEATURE.LAYER.slice.yaml` exists for a layer that is not `done`, say
   which scenario is the first not `refactored`: that layer is resumable.
4. Tell the user the next skill and to start a new chat session for it:

| Next pending layer | Skill |
|---|---|
| design_system | `feature-design-system` |
| domain | `feature-domain` |
| data | `feature-data` (needs domain `done`) |
| presentation | `feature-presentation` (needs domain and design_system `done` or `skipped`) |

5. Stop. Do not start the layer in this session.

## Finishing

When every layer is `done` (or `skipped`):
1. Run `dart format .`, `flutter analyze` and the full `flutter test`.
2. Run the dependency check from `figma-to-flutter` on `lib/design_system/`.
3. Read each `specs/FEATURE.LAYER.slice.yaml` and report: layers built, scenarios implemented per
   component, scenarios added beyond the spec, fakes created, layers that had to be redone, and
   anything the user should review.

## Common Mistakes

- Starting a layer inside the router session.
- Starting from a draft spec.
- Skipping the finishing checks.
- Editing the spec beyond reading `build:`.
