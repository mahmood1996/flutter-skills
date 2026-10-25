---
name: test-driven-development
description: "TDD: enforce RED-GREEN-REFACTOR, tests before code."
version: 1.2.0
author: Hermes Agent (adapted from obra/superpowers)
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [testing, tdd, development, quality, red-green-refactor, flutter, fakes]
    related_skills: [systematic-debugging, subagent-driven-development]
---

# Test-Driven Development (TDD)

## Overview

Write the test first. Watch it fail. Write minimal code to pass. Refactor. Then, and only then, the next test.

**Core principle:** If you didn't watch the test fail, you don't know if it tests the right thing.

**Violating the letter of the rules is violating the spirit of the rules.**

## When to Use

**Always:**
- New features
- Bug fixes
- Refactoring
- Behavior changes

**Exceptions (ask the user first):**
- Throwaway prototypes
- Generated code (`*.g.dart`)
- Configuration files

Thinking "skip TDD just this once"? Stop. That's rationalization.

## The Iron Laws

```
1. NO PRODUCTION CODE WITHOUT A FAILING TEST FIRST
2. ONE TEST AT A TIME: never write a second test until the first
   one is green AND refactored
```

Write code before the test? Delete it. Start over.
Wrote several tests before the first implementation? Keep the first, delete the rest, and
rewrite them one at a time later. The tests you delete will be better when you rewrite them,
because the implementation will have taught you what matters.

**No exceptions:**
- Don't keep deleted code as "reference"
- Don't "adapt" it while writing tests
- Don't look at it
- Delete means delete

Implement fresh from tests. Period.

## The Loop (one scenario or behavior per pass)

```
pick next behavior
  -> RED:      write ONE test, run it, see it fail on the assertion
  -> GREEN:    write minimal code, run it, see it pass, run the layer's tests
  -> REFACTOR: clean up THIS behavior's code now, run the layer's tests again
  -> next behavior
```

Refactor happens inside every pass, not at the end of a batch. "Refactor later, after all
tests pass" is the horizontal-slice mistake in disguise.

### Scenario status ledger

If you are given a list of scenarios with a `status` field (for example a spec slice), the
list is your ledger. Keep it truthful:

```
pending -> red -> green -> refactored
```

- Work only on the first `pending` scenario.
- At most ONE scenario may be `red` or `green` at any moment. Every earlier scenario is
  `refactored`, every later one is `pending`.
- Update the status in the file right after the matching test run, never before it and
  never in bulk.
- `red` requires you to have seen the failing run. `green` requires you to have seen the
  passing run. Keep the one failing output line for each scenario and include it in your report.
- Do not open the next scenario while the current one is not `refactored`.

No ledger given? Keep the same discipline with a plain list in your own notes.

## Red-Green-Refactor Cycle

### RED — Write Failing Test

Write one minimal test showing what should happen.

**Good test** (London School: the unit under test, with a hand-written Fake for its direct collaborator):
```dart
test('returns the orders when the repository has orders', () async {
  final repository = FakeOrderRepository(orders: [order]);
  final useCase = GetOrders(repository);

  final result = await useCase();

  expect(result, Right<Failure, List<Order>>([order]));
});
```
Clear name, real unit under test, Fake seeded through the constructor, asserts the outcome.

**Bad test:**
```dart
test('works', () async {
  final repository = MockOrderRepository();
  when(() => repository.orders()).thenAnswer((_) async => Right([order]));
  await GetOrders(repository)();
  verify(() => repository.orders()).called(1);
});
```
Vague name, mocking package, asserts the interaction instead of the outcome.

**Requirements:**
- One behavior per test
- Clear descriptive name ("and" in name? Split it)
- Collaborators are hand-written Fakes in `test/fakes/`; never a mocking package
  (`mockito`, `mocktail`) and never interaction expectations like `verify(...)`
- The unit under test is always the real code
- Assert on outcomes and on what the Fake recorded (noun getters like `requests`, `callCount`)
- Name describes behavior, not implementation

### Verify RED — Watch It Fail

**MANDATORY. Never skip.**

```bash
# Run only the new test
flutter test test/features/FEATURE/LAYER/NAME_test.dart --plain-name "test name"
```

Confirm:
- The test fails on the assertion
- Failure message is the expected one
- It fails because the behavior is missing

**First test of a new unit does not compile?** That is the expected first RED. Create only the
minimal stubs that fix the compile errors (class, signature, interface), then run again until
it fails on the assertion. Do not implement behavior yet.

**Test passes immediately?** You are testing existing behavior. Fix the test.

**Test errors for a typo or setup mistake?** Fix the error, re-run until it fails correctly.

### GREEN — Minimal Code

Write the simplest code to pass the test. Nothing more.

**Good:**
```dart
Future<Either<Failure, List<Order>>> call() => repository.orders();
```

**Bad:** adding caching, logging or error mapping that no test asked for.

Don't add features, refactor other code, or "improve" beyond the test.

**Cheating is OK in GREEN:**
- Hardcode return values
- Copy-paste
- Duplicate code
- Skip edge cases

The next scenario forces the generalization, and REFACTOR cleans it up.

### Verify GREEN — Watch It Pass

**MANDATORY.**

```bash
# The new test
flutter test test/features/FEATURE/LAYER/NAME_test.dart --plain-name "test name"

# Then the whole layer, to check for regressions
flutter test test/features/FEATURE/LAYER
```

Confirm:
- Test passes
- Other tests still pass
- Output pristine (no errors, warnings)

**Test fails?** Fix the code, not the test.

**Other tests fail?** Fix regressions now.

### REFACTOR — Clean Up (every pass, before the next test)

After green only, and only for the code this scenario touched:
- Remove duplication
- Improve names
- Extract helpers
- Simplify expressions
- Simplify the Fake if it grew clever

Run the layer's tests again. Keep tests green throughout. Don't add behavior.

**If tests fail during refactor:** Undo immediately. Take smaller steps.

Nothing to clean up? Say so and move on, but still run the tests.

### Repeat

Next failing test for next behavior. One cycle at a time.

## Avoid Horizontal Slices

Do **not** write all tests first and then all implementation. That is horizontal slicing: RED
becomes "write a pile of imagined tests" and GREEN becomes "make the pile pass." It produces
brittle tests, because the tests are designed before the implementation has taught you what
behavior and interface actually matter. Writing all the tests, making them all green, and
refactoring at the end is the same mistake.

```text
WRONG:
  RED:   test1, test2, test3, test4
  GREEN: impl1, impl2, impl3, impl4
  REFACTOR: everything

RIGHT:
  RED -> GREEN -> REFACTOR: test1 -> impl1 -> clean1
  RED -> GREEN -> REFACTOR: test2 -> impl2 -> clean2
  RED -> GREEN -> REFACTOR: test3 -> impl3 -> clean3
```

A tracer bullet is one end-to-end behavior slice. It proves the path works, teaches you about
the interface, and keeps each next test grounded in what you just learned.

## Common Rationalizations

| Excuse | Reality |
|--------|---------|
| "Too simple to test" | Simple code breaks. Test takes 30 seconds. |
| "I'll test after" | Tests passing immediately prove nothing. Tests-after answer "what does this do?", tests-first answer "what should this do?" |
| "I'll write all the tests first, it's faster" | That is horizontal slicing. One test, one pass, then the next. |
| "I'll refactor once everything is green" | Refactor now, per scenario. Late refactoring never happens or breaks everything at once. |
| "Already manually tested" | Ad-hoc is not systematic. No record, can't re-run. |
| "Deleting X hours is wasteful" | Sunk cost fallacy. Keeping unverified code is technical debt. |
| "Keep as reference, write tests first" | You'll adapt it. That's testing after. Delete means delete. |
| "Need to explore first" | Fine. Throw away the exploration, start with TDD. |
| "Test hard = design unclear" | Listen to the test. Hard to test = hard to use. |
| "TDD will slow me down" | TDD is faster than debugging. |
| "Existing code has no tests" | You're improving it. Add tests for the code you touch. |
| "I'll update all the statuses at the end" | A status is a record of a run you just saw. Update it as you go. |

## Red Flags — STOP and Start Over

If you catch yourself doing any of these, delete the code (or the extra tests) and restart with TDD:

- Code before test
- Test after implementation
- More than one new test in a single edit or before the first is green
- More than one scenario in `red` or `green` at once
- Test passes immediately on first run
- Can't explain why test failed
- Statuses changed without a test run just before
- Refactoring postponed to "the end"
- Tests added "later"
- Rationalizing "just this once"
- "I already manually tested it"
- "Keep as reference" or "adapt existing code"
- "Already spent X hours, deleting is wasteful"
- "This is different because..."

**All of these mean: delete the code or the extra tests. Start over with TDD.**

## Verification Checklist

Before marking work complete:

- [ ] Every new function/method has a test
- [ ] Tests were written and run one at a time
- [ ] Watched each test fail before implementing
- [ ] Each test failed for the expected reason (behavior missing, not typo)
- [ ] Wrote minimal code to pass each test
- [ ] Refactored after each green, not only at the end
- [ ] All tests pass
- [ ] Output pristine (no errors, warnings)
- [ ] Collaborators are hand-written Fakes, no mocking packages
- [ ] Edge cases and errors covered
- [ ] Every scenario in the ledger is `refactored`, each with a recorded failing output line

Can't check all boxes? You skipped TDD. Start over.

## When Stuck

| Problem | Solution |
|---------|----------|
| Don't know how to test | Write the wished-for API. Write the assertion first. Ask the user. |
| Test too complicated | Design too complicated. Simplify the interface. |
| Need to mock everything | Code too coupled. Use dependency injection and a small Fake. |
| Test setup huge | Extract helpers. Still complex? Simplify the design. |
| A Fake is getting clever | It should only do what the current scenarios need. Simplify it. |

## Hermes Agent Integration

### Running Tests

Use the `terminal` tool to run tests at each step:

```python
# RED: verify failure
terminal('flutter test test/features/FEATURE/LAYER/NAME_test.dart --plain-name "test name"')

# GREEN: verify pass
terminal('flutter test test/features/FEATURE/LAYER/NAME_test.dart --plain-name "test name"')

# Layer: verify no regressions (after GREEN and again after REFACTOR)
terminal("flutter test test/features/FEATURE/LAYER")
```

### With delegate_task

A sub-agent may never read this skill, so put the loop in the context itself:

```python
delegate_task(
    goal="Implement [feature] using strict TDD, one scenario at a time",
    context="""
    LOOP (repeat per scenario, one at a time, in order):
    1. Take the first scenario with status: pending. Only one scenario may be
       red or green at a time.
    2. Write exactly ONE test. Run it. It must fail on the assertion.
       Set status: red and keep the one-line failing output.
    3. Write the minimal code. Run that test. Set status: green.
    4. Run the layer's tests. Refactor THIS scenario's code now. Run them again.
       Set status: refactored.
    5. Only then start the next scenario.
    Never write a second test while a scenario is red or green.
    Fakes are hand-written, in test/fakes/. No mocking packages.

    Project test command: flutter test test/features/FEATURE/LAYER
    REPORT: every scenario with its one-line failing output from step 2.
    """,
    toolsets=['terminal', 'file']
)
```

### With systematic-debugging

Bug found? Write a failing test reproducing it. Follow the TDD loop. The test proves the fix and prevents regression.

Never fix bugs without a test.

## Testing Anti-Patterns

- **Mocking packages and interaction expectations** — use hand-written Fakes and assert on outcomes
- **Testing implementation details** — test behavior and results, not internal method calls
- **Happy path only** — always test edge cases, errors, and boundaries
- **Brittle tests** — tests should verify behavior, not structure; refactoring shouldn't break them
- **Batching** — several tests (or several scenarios in progress) at once

## Final Rule

```
Production code -> a test exists and failed first
One test at a time -> red, green, refactor, then the next
Otherwise -> not TDD
```

No exceptions without the user's explicit permission.
