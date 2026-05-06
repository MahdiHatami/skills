# Good and Bad Tests (Android)

## Good Tests

**Integration-style:** test through real interfaces, not mocks of internal parts. Assert on the contract (`UiState`, `Flow` emissions, rendered nodes) — never on `verify(mock).someInternalCall()`.

### ViewModel — assert on emitted `UiState`

```kotlin
@Test
fun `submits valid cart and reaches Confirmed`() = runTest {
    val viewModel = CheckoutViewModel(
        cart = FakeCart(items = listOf(product)),
        payments = FakePayments(approve = true),
    )

    viewModel.state.test {
        assertEquals(UiState.Idle, awaitItem())

        viewModel.onAction(Action.Submit(paymentMethod))

        assertEquals(UiState.Submitting, awaitItem())
        assertEquals(UiState.Confirmed(orderId = "ord_1"), awaitItem())
    }
}
```

Characteristics:

- Tests behavior callers/users care about (the visible state machine).
- Uses public API only (`onAction`, `state`).
- Survives internal refactors — rename a private fn, this passes.
- Describes WHAT, not HOW.
- One logical assertion per test (the trajectory).

### Repository — verify through the interface

```kotlin
@Test
fun `getUser returns the same data that was just stored`() = runTest {
    val repo = UserRepository(local = InMemoryUserDao(), remote = FakeUserApi())

    repo.upsert(User(id = "u1", name = "Alice"))

    assertEquals("Alice", repo.getUser("u1").first().name)
}
```

Doesn't bypass the interface to query SQLite directly — verifies through `getUser`, which is what callers actually use.

### Composable — assert on rendered behavior

```kotlin
@Test
fun `tapping submit shows progress and confirmed states`() = runComposeUiTest {
    val state = mutableStateOf<UiState>(UiState.Idle)
    setContent { CheckoutScreen(state = state.value, onSubmit = { state.value = UiState.Submitting }) }

    onNodeWithText("Submit").performClick()

    state.value = UiState.Confirmed("ord_1")
    onNodeWithText("Order ord_1 confirmed").assertIsDisplayed()
}
```

Drives the Composable through its hoisted state — its real interface — and asserts on rendered nodes.

## Bad Tests

**Implementation-detail tests:** coupled to internals. They break under refactor when behavior didn't change.

### Verifying mock interactions instead of behavior

```kotlin
// BAD: tests HOW, not WHAT
@Test
fun `submit calls payments_charge`() = runTest {
    val payments = mockk<Payments>(relaxed = true)
    val viewModel = CheckoutViewModel(cart = FakeCart(), payments = payments)

    viewModel.onAction(Action.Submit(paymentMethod))

    coVerify { payments.charge(any()) } // ← couples test to internal call shape
}
```

Red flags:

- `coVerify` / `verify` on a collaborator you control.
- Test breaks when you rename `charge` to `process`, even though the user-visible state machine hasn't changed.
- Test name describes a method invocation, not a behavior.

### Bypassing the interface

```kotlin
// BAD: bypasses repository to query Room directly
@Test
fun `upsert writes to user_table`() = runTest {
    val repo = UserRepository(local = realDao, remote = FakeUserApi())
    repo.upsert(User("u1", "Alice"))

    val row = realDao.queryRaw("SELECT * FROM user_table WHERE id = 'u1'") // ← internal detail
    assertNotNull(row)
}

// GOOD: verify through the same interface a caller would use
@Test
fun `upsert makes user retrievable`() = runTest {
    val repo = UserRepository(local = realDao, remote = FakeUserApi())
    repo.upsert(User("u1", "Alice"))

    assertEquals("Alice", repo.getUser("u1").first().name)
}
```

The bad version pins the schema. The good version pins the contract.

### Asserting on private state via reflection

```kotlin
// BAD: reaches into the ViewModel
val pending = viewModel::class.java.getDeclaredField("pending")
    .also { it.isAccessible = true }
    .get(viewModel)
assertEquals(1, (pending as List<*>).size)
```

Always wrong. If the private state matters, hoist it into the `UiState` so it's part of the interface.

### Testing the inner Composable

```kotlin
// BAD: pins implementation structure
@Test
fun `CheckoutScreen contains a SubmitButton`() = runComposeUiTest {
    setContent { CheckoutScreen(state = UiState.Idle, onSubmit = {}) }
    onNode(hasTestTag("SubmitButton")).assertExists() // ← SubmitButton is an internal Composable
}
```

If `SubmitButton` is replaced with a generic `Button` that does the same thing, this test fails for no reason. Assert on the rendered text or semantics that the user actually perceives.
