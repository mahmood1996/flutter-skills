---
name: feature-spec
description: "Use when planning a Flutter feature from Figma and Swagger."
version: 1.0.0
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [flutter, spec, planning, clean-architecture, tdd]
    related_skills: [feature-build, flutter-clean-architecture, figma-to-flutter, test-driven-development]
---

# Feature Spec

## Overview

Turn a UI source and an API definition into one approved YAML spec. The spec is the
contract that `feature-build` implements, one layer at a time. Mistakes are cheapest to fix
here, before any code exists, so be precise and surface every doubt.

**This skill never writes production code or test code.** Its only output is
`specs/FEATURE.spec.yaml`, where FEATURE is the snake_case feature name.

## When to Use

Use it whenever the user:
- wants to plan, scope or spec a Flutter feature before coding;
- shares a Figma link or screenshot together with a Swagger/OpenAPI source;
- says "write the spec", "prepare the feature for implementation", or similar.

Use it even if the user never says "spec". Hand the approved result to `feature-build`.

## Inputs

| Input | Accepted forms |
|---|---|
| UI | Figma selection link, or a screenshot |
| API | Swagger/OpenAPI URL, or a `swagger.json` / `swagger.yaml` file |

If one input is missing, ask for it once. If the feature name is not obvious, derive a
snake_case name from the screen title and state it in your summary.

## Workflow

### 1. Read the project
Look at `lib/core/` (failures, exceptions, shared clients) and `lib/features/`. Reuse what
exists: failures, exceptions, domain models, `AppHttpClient`. Mark reused items
`reuse: true` instead of redefining them.

### 2. Normalize the UI into a description
- **Figma**: use the Figma MCP for design context, variable definitions and a screenshot
  of the selection. Set `inputs.ui.tokens: figma_variables`.
- **Screenshot**: view the image. There are no tokens, so set `inputs.ui.tokens: estimated`
  and add an assumption saying the design system step will estimate values.

From either source list: screens, the states each screen needs (loading, empty, error,
content, plus any the design shows), every field displayed, and every user action. A
screenshot usually shows one state. Do not invent the others silently: include the standard
four and add an assumption that they are not designed.

### 3. Read the API definition
Fetch or open the Swagger source. If it is large, pick the operations the feature needs
from the UI actions and fields; if that is ambiguous, ask which operations, once. Record them
in `inputs.api.operations`. For each operation note request schema, response schema,
required and nullable fields, and every documented status code.

### 4. Map the UI to the API
For each UI field find its API source. Anything shown with no API source, or any relevant
API field that is not shown, goes into `open_questions` with a `blocks` entry. Ambiguity is
an open question, never a guess.

### 5. Write the spec
Create `specs/FEATURE.spec.yaml` using the format below. Set `status: draft` and every
`build` entry to `pending`.

Derive scenarios mechanically where possible:
- each documented status code becomes a data source scenario and a failure mapping
- each nullable field becomes a DTO scenario
- each screen state becomes a state holder scenario and a view scenario
- each use case gets a success, an empty (if it returns a collection) and a failure scenario

### 6. Validate
Parse the file with the terminal tool:

```bash
python3 -c "import yaml,sys; yaml.safe_load(open(sys.argv[1])); print('ok')" specs/FEATURE.spec.yaml
```

Then check by hand:
- every `usecase` named in `ui.screens[].actions` exists under `domain.usecases`
- every repository method used by a use case exists on its repository
- every datasource method referenced by a repository impl exists
- every fake named in a scenario appears in `fakes`
- every scenario id is unique

Fix inconsistencies before presenting.

### 7. Stop at the approval gate
Show the user: the `summary`, the components per layer, the number of scenarios per
component, all `assumptions` and all `open_questions`. Ask them to approve or correct.
Set `status: approved` only after the user says so. If unresolved open questions block
components, name them and do not approve.

## Spec format

Pure YAML. No prose outside `summary`, `assumptions` and `open_questions`. Every component
carries its own scenarios, so a layer slice is self-contained. This skeleton shows one
component of each kind; repeat the pattern for every component in the feature.

```yaml
spec_version: 1
feature: orders                   # snake_case, becomes lib/features/orders/
status: draft                     # draft | approved (feature-build refuses draft)

inputs:
  ui:  {kind: figma, source: "https://figma.com/...", tokens: figma_variables}  # kind: figma | screenshot; tokens: figma_variables | estimated
  api: {kind: swagger_url, source: "https://example.com/swagger.json",
        operations: [{operationId: listOrders, method: GET, path: /orders}]}

ui:
  screens:
    - id: orders_screen
      name: OrdersScreen
      states: [loading, empty, error, content]
      fields: [{ui: order number, source: Order.orderNumber}]
      actions:
        - {id: refresh, trigger: pull to refresh, usecase: GetOrders}   # usecase: null => navigation only
      widgets:
        design_system: [AppCard]            # built through figma-to-flutter
        feature: [OrderCard, OrdersEmptyView]

domain:
  models:
    - name: Order
      fields: [{name: number, type: String, nullable: false, source: Order.orderNumber}]
  failures:
    - {name: ServerFailure, reuse: true}
    - {name: UnauthorizedFailure, reuse: false, when: "API returns 401"}
  repositories:
    - name: OrdersRepository
      methods:
        - {name: orders, params: [], returns: "Future<Either<Failure, List<Order>>>",
           errors: [ServerFailure, UnauthorizedFailure]}
  usecases:
    - name: GetOrders
      repository: OrdersRepository
      method: orders
      params: []
      returns: "Future<Either<Failure, List<Order>>>"
      scenarios:
        - id: GO-1
          name: returns orders when repository succeeds
          given: FakeOrdersRepository seeded with two orders
          when: call()
          then: returns Right with the two orders

data:
  dtos:
    - name: OrderDto
      schema: "#/components/schemas/Order"
      fields: [{json: orderNumber, type: String, nullable: false}]
      converts_to: Order
      scenarios:
        - {id: OD-1, name: converts a full payload to an Order, given: json with all fields,
           when: parsed and converted, then: Order fields equal the json values}
  datasources:
    - name: OrdersRemoteDataSource
      methods:
        - name: orders
          endpoint: {method: GET, path: /orders}
          request_dto: null
          response_dto: "List<OrderDto>"
          status_map: {200: returns dtos, 401: UnauthorizedException, 500: ServerException,
                       network_error: ServerException}
          scenarios:
            - {id: RDS-1, name: returns dtos when api answers 200,
               given: FakeAppHttpClient answering 200 with two orders,
               when: orders(), then: returns two OrderDto}
  repository_impls:
    - name: OrdersRepositoryImpl
      implements: OrdersRepository
      calls:
        - method: orders
          datasource: OrdersRemoteDataSource.orders
          exception_to_failure: {ServerException: ServerFailure,
                                 UnauthorizedException: UnauthorizedFailure}
          scenarios:
            - {id: ROI-1, name: maps ServerException to ServerFailure,
               given: FakeOrdersRemoteDataSource set to throw ServerException,
               when: orders(), then: returns Left with ServerFailure}

presentation:
  state_holders:
    - name: OrdersCubit
      kind: cubit                     # cubit | bloc
      usecases: [GetOrders]
      states:
        - {name: OrdersLoading}
        - {name: OrdersLoaded, fields: ["List<Order> orders"]}
      scenarios:
        - {id: OC-1, name: emits loading then loaded when orders exist,
           given: FakeGetOrders seeded with two orders, when: load(),
           then: "states are [OrdersLoading, OrdersLoaded with two orders]"}
  views:
    - name: OrdersView                # pumped inside a BlocProvider with a fake state holder
      scenarios:
        - {id: OV-1, name: shows one card per order when loaded,
           given: state is OrdersLoaded with two orders, when: view is pumped,
           then: two OrderCard widgets are visible}

fakes:                                # contract per fake; feature-build writes them lazily in test/fakes/
  - {name: FakeOrdersRepository, fakes: OrdersRepository,
     behaviors: [seedable, can_fail_with_failure], records: []}
  - {name: FakeAppHttpClient, fakes: AppHttpClient,
     behaviors: [answers_with_status_and_body, can_throw_network_error], records: [requests]}

contract_tests: false                 # true => same suite runs against fake and real repository

summary: >
  One paragraph for the human reviewer.
assumptions:
  - Orders are not paginated because the swagger has no paging parameters.
open_questions:
  - {id: Q1, question: "Is Order.note shown in the design?", blocks: [domain.models.Order]}

build:                                # written only by feature-build
  design_system: pending              # pending | done | skipped
  domain: pending
  data: pending
  presentation: pending
```

## Scenario Rules

1. **Behavior, not implementation.** "Returns the mapped orders when the API answers 200"
   is a scenario. "Calls `remote.orders` once" is not, unless the call itself is the
   behavior (for example "saves the note"); then state it as what the fake recorded.
2. **One behavior per scenario.** An "and" in the name means split it.
3. **Ordered simplest first.** The first scenario of each component is the tracer bullet.
4. **Given/when/then mention only fakes and public API**, naming the fake and its state.
5. **A minimum, not the full set.** `feature-build` may add edge cases it discovers.

## Naming Rules

- Query methods are nouns (`orders`, `label`), not verbs (`getOrders`, `fetchOrders`).
- Type names do not end in "er" or "or". Pick a noun for what the thing is
  (`OrdersRemoteDataSource`, not `OrdersFetcher`; `OrderDtoConversion`, not `OrderMapper`).
  Use case names follow the convention already in the codebase; check before choosing.
- Files `snake_case`, classes `UpperCamelCase`, members `lowerCamelCase`.

## Common Mistakes

- Writing test code or implementation snippets into the spec.
- Guessing an ambiguous field instead of adding an open question.
- Defining a failure or exception that already exists in `core/error/`.
- Scenarios that describe how something works instead of what it does.
- Marking the spec `approved` without the user's explicit confirmation.
