# Testing Coroutines and Flow

Two non-negotiables for the feedback loop:

1. **`runTest` + virtual time** — never let a real `delay(...)` execute in a test.
2. **Turbine for `Flow` assertions** — never collect into a `List` and check size.

## `runTest` and virtual time

`runTest` from `kotlinx-coroutines-test` provides a `TestScope` whose dispatcher advances time virtually. A `delay(60_000L)` completes in microseconds.

```kotlin
@Test
fun `retries after backoff`() = runTest {
    val client = FakeClient(failuresBeforeSuccess = 2)
    val sut = WithRetry(client, backoff = 1.minutes)

    val result = sut.fetch()  // would be 2 minutes of real wall time

    assertEquals("ok", result)
}
```

If your test is slow, check that:

- You're calling `runTest`, not just `runBlocking`.
- Production code uses an injected `CoroutineDispatcher`, not `Dispatchers.IO` directly.
- You're not invoking `Thread.sleep`.

## `TestDispatcher` discipline

Inject the dispatcher into production code so tests can substitute the test one:

```kotlin
class TokenRefresher(
    private val dispatchers: DispatcherProvider,
    private val clock: Clock,
)

// in test
@Test
fun ...() = runTest {
    val sut = TokenRefresher(
        dispatchers = TestDispatcherProvider(testScheduler),
        clock = FixedClock(Instant.parse("2025-01-01T00:00:00Z")),
    )
}
```

Two flavours of `TestDispatcher`:

- `StandardTestDispatcher` — does **not** run launched coroutines eagerly. Test calls `advanceUntilIdle()` / `runCurrent()` to step through. Best for testing scheduling.
- `UnconfinedTestDispatcher` — runs launched coroutines eagerly (depth-first). Best when you don't care about scheduling order, just behavior.

When in doubt, start with `UnconfinedTestDispatcher` — it surprises you less.

## Turbine for `Flow` assertions

```kotlin
@Test
fun `emits Idle then Loading then Success`() = runTest {
    val viewModel = FooViewModel(repo = FakeRepo(returns = "x"))

    viewModel.state.test {
        assertEquals(UiState.Idle, awaitItem())

        viewModel.onAction(Action.Load)

        assertEquals(UiState.Loading, awaitItem())
        assertEquals(UiState.Success("x"), awaitItem())

        cancelAndIgnoreRemainingEvents()
    }
}
```

Turbine APIs you'll use most:

- `awaitItem()` — next emission, fails the test if there isn't one within `timeout` (default 3s).
- `awaitComplete()` — Flow completed.
- `awaitError()` — Flow threw.
- `expectMostRecentItem()` — drain the queue, return the last emission. Useful for `StateFlow` where you don't care about every intermediate state.
- `cancelAndIgnoreRemainingEvents()` — explicit close at the end of the test block.
- `skipItems(n)` — advance past N emissions you don't care about.

### Don't do this

```kotlin
// BAD: race-prone, depends on timing
val emissions = viewModel.state.take(3).toList()
assertEquals(UiState.Success("x"), emissions.last())
```

If a fourth emission arrives before `take(3)` cancels — or if the third arrives after the test scope dies — this either times out or asserts on the wrong item. Turbine eliminates the race.

### `StateFlow` gotcha

`StateFlow.test { }` immediately emits the current value. Always read it first:

```kotlin
viewModel.state.test {
    awaitItem()  // ← initial state, often UiState.Idle
    viewModel.onAction(Action.Load)
    assertEquals(UiState.Loading, awaitItem())
    ...
}
```

Forgetting this leads to "expected `Loading` but got `Idle`" failures that look like a behavior bug but are a test setup bug.

## `WhileSubscribed` timeout

If a `StateFlow` is created with `stateIn(scope, SharingStarted.WhileSubscribed(5_000), initial)`, a test that subscribes, unsubscribes, and re-subscribes within the 5s window will get a stale cached value. Either:

- Use `SharingStarted.Eagerly` in tests, or
- Pass `WhileSubscribed(0)` so unsubscribe drops state immediately, or
- Keep the subscription alive across the assertions (preferred — that's how production behaves).

## Single-flight / debounce

When testing debounce (e.g. typing into a search box):

```kotlin
@Test
fun `debounces rapid input`() = runTest {
    val searches = mutableListOf<String>()
    val sut = SearchPipeline(onQuery = { searches += it }, debounce = 300.milliseconds)

    sut.type("a")
    sut.type("ab")
    sut.type("abc")

    advanceTimeBy(299.milliseconds)
    assertEquals(emptyList<String>(), searches)

    advanceTimeBy(2.milliseconds)
    assertEquals(listOf("abc"), searches)
}
```

Virtual time makes debounce tests deterministic and instant. Real `delay` would make this a flake-prone 301ms wait.
