---
name: tdd-android
description: Test-driven development for Android/Kotlin with red-green-refactor loop. Tier-aware feedback loops (JVM > Robolectric > Compose UI > instrumented), Flow/coroutines testing with Turbine and virtual time, MockK at boundaries. Use when user wants to build Android features or fix Android bugs using TDD, mentions Compose/ViewModel/Repository tests, or asks for test-first development on a Kotlin/Android codebase.
---

# Test-Driven Development (Android)

Sister skill to `/tdd`, applied to Android/Kotlin codebases. Same philosophy, plus
test-tier discipline that protects the feedback loop on a platform where the wrong
tier means a 60-second emulator boot per cycle.

## Philosophy

Same as `/tdd`:

- Test behavior through public interfaces, not implementation details.
- Good tests describe **what** the system does, not **how**.
- Survive refactors. Break only when behavior breaks.

Android-specific corollaries:

- A **ViewModel's** interface is its `UiState` + `Action` (or `Intent`/`Event`) contract — not its method names.
- A **Composable's** interface is its parameters and the state it hoists — not its inner Composables.
- A **Repository's** interface is its `suspend` / `Flow` signatures — not the data source it talks to.

See [tests.md](tests.md) for examples and [mocking.md](mocking.md) for boundary rules.

## The Test Pyramid Is The Feedback Loop

The `/tdd` skill insists on fast deterministic feedback. On Android the wrong test tier kills that.

Default tier from cheapest to most expensive:

1. **Pure JVM unit test** — millis. Use for anything that doesn't touch the Android framework: domain logic, mappers, ViewModel state, Flow operators, use cases.
2. **Robolectric** — ~1s warm. Use for Android-framework code without an emulator: `SharedPreferences`, `Context`-bound code, lifecycle, `Resources`.
3. **Compose UI test** (`runComposeUiTest`) — a few seconds. Use for Composable behavior: node assertions, gesture input, state→render verification.
4. **Instrumented** (Espresso / device) — minutes. Reserve for true device-coupled behavior: real `WindowManager`, real network stack, Camera/sensors, real SQLite migrations.

**Rule:** write the test at the highest (cheapest) tier that can answer the question. Push down only when forced. If you find yourself writing instrumented tests in the TDD loop, stop and ask whether the test could be split: business logic to JVM, framework glue to Robolectric, render assertion to Compose UI test.

See [test-pyramid.md](test-pyramid.md) for the decision tree.

## Anti-Pattern: Horizontal Slices

Same rule as `/tdd` — **DO NOT write all tests first, then all implementation.**

Android-flavoured horizontal slicing:

- Writing every ViewModel test, then every Repository test, then every UseCase test.
- Producing a wall of `@Test fun ()` stubs and only then filling in implementations.
- Stubbing every `UiState` transition before the Composable can render any of them.

Both produce tests that verify imagined shapes, not actual behavior.

**Correct:** one vertical slice per cycle. `Action` input → ViewModel state → Repository call → DataSource → assertion via Turbine. Repeat.

```
WRONG (horizontal):
  RED:   vmTest1, vmTest2, repoTest1, repoTest2, ...
  GREEN: vmImpl1, vmImpl2, repoImpl1, repoImpl2, ...

RIGHT (vertical):
  RED→GREEN: slice1 (Action → state → repo → datasource)
  RED→GREEN: slice2 (next Action)
  ...
```

## Coroutines & Flow

Two non-negotiables for the feedback loop:

- **Use `runTest` with virtual time.** Real `delay(...)` in tests is the easiest way to ruin determinism. `runTest` + a `TestDispatcher` collapses minute-long timeouts into microseconds.
- **Assert on `Flow` with Turbine.** `flow.test { ... }` gives you `awaitItem()`, `awaitComplete()`, `cancelAndIgnoreRemainingEvents()` — explicit, deterministic, no race-prone `take(n).toList()`.

See [flow-testing.md](flow-testing.md) for patterns.

## Workflow

### 1. Planning

When exploring the codebase, use the project's domain glossary so test names and contract types match the project's language, and respect ADRs in the area you're touching.

Before writing any code:

- [ ] Confirm with user what interface changes are needed (`UiState`, `Action`, Repository contract)
- [ ] Confirm with user which behaviors to test (prioritize)
- [ ] **Pick the test tier for each behavior** (JVM / Robolectric / Compose UI / Instrumented)
- [ ] Identify opportunities for [deep modules](../tdd/deep-modules.md) (small interface, deep implementation)
- [ ] Design interfaces for [testability](interface-design.md)
- [ ] Get user approval on the plan

Ask: "What should the public interface look like? Which behaviors are most important to test? Which tier?"

**You can't test everything.** Confirm with the user exactly which behaviors matter most. Focus testing effort on critical paths and complex logic, not every possible edge case.

### 2. Tracer Bullet

Write ONE test that exercises ONE end-to-end behavior at the chosen tier:

```
RED:   Write test for first behavior → fails
GREEN: Minimal code to pass → passes
```

This is your tracer bullet — proves the path works end-to-end.

### 3. Incremental Loop

For each remaining behavior:

```
RED:   Write next test → fails
GREEN: Minimal code to pass → passes
```

Rules:

- One test at a time
- Only enough code to pass the current test
- Don't anticipate future tests
- Tests assert on **observable** behavior — `UiState` transitions, `Flow` emissions, rendered nodes — not internals

### 4. Refactor

After GREEN, look for [refactor candidates](../tdd/refactoring.md):

- [ ] Extract duplication
- [ ] Deepen modules (move complexity behind a UseCase or Repository)
- [ ] Apply SOLID principles where natural
- [ ] Run tests after each refactor step

**Never refactor while RED.** Get to GREEN first.

## Checklist Per Cycle

```
[ ] Test describes behavior, not implementation
[ ] Test runs at the cheapest tier that can answer the question
[ ] Test uses public interface only (UiState/Action/Flow contract — not internal methods)
[ ] Test uses runTest + virtual time if it touches coroutines
[ ] Test would survive an internal refactor
[ ] Code is minimal for this test
[ ] No speculative features added
```
