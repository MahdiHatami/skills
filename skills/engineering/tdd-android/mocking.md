# When to Mock (Android / Kotlin)

Mock at **system boundaries** only:

- Network APIs (Retrofit, Ktor)
- Databases (sometimes — prefer in-memory Room or a fake DAO)
- Time (`Clock` / `Instant.now`)
- Randomness
- Platform sources you don't own (`LocationManager`, `SensorManager`, `BiometricPrompt`)
- File system / `Context` resources (sometimes)

Don't mock:

- Your own ViewModels, UseCases, Repositories
- Internal collaborators
- Anything you control and could write a fake for

**Prefer fakes over mocks.** A `FakeUserRepository` that holds an in-memory `MutableMap` is more honest about its contract than a `mockk<UserRepository>` with `coEvery { ... } returns ...` — and it survives refactors of the interface because the compiler tells you when it drifts.

## Designing for Mockability (and Fake-ability)

At system boundaries, design interfaces that are easy to fake:

### 1. Constructor injection over service locators

Pass dependencies in. Don't fetch them from a global.

```kotlin
// Easy to test
class CheckoutViewModel(
    private val cart: CartRepository,
    private val payments: Payments,
    private val clock: Clock,
)

// Hard to test
class CheckoutViewModel : ViewModel() {
    private val cart = CartRepositoryImpl()       // ← reaches into globals
    private val payments = StripePayments(BuildConfig.STRIPE_KEY)
}
```

### 2. SDK-style interfaces over generic clients

Each external operation gets its own function. One mock returns one shape.

```kotlin
// GOOD: each function is independently fake-able
interface UserApi {
    suspend fun getUser(id: String): UserDto
    suspend fun listOrders(userId: String): List<OrderDto>
    suspend fun createOrder(req: CreateOrderRequest): OrderDto
}

// BAD: mocking requires conditional logic by URL
interface HttpClient {
    suspend fun <T> get(url: String): T
    suspend fun <T> post(url: String, body: Any): T
}
```

The SDK approach means:

- Each fake/mock returns one specific shape, no `when (url)` branching.
- No conditional logic in test setup.
- It's obvious which endpoints a test exercises.
- Type safety per endpoint.

### 3. `Clock` and `Dispatchers` are inputs, not globals

```kotlin
// GOOD
class TokenRefresher(
    private val clock: Clock = Clock.System,
    private val dispatchers: DispatcherProvider,
)

// BAD
class TokenRefresher {
    fun expiresAt(token: Token) = Instant.now().plusSeconds(token.ttl)  // ← unfreezable time
}
```

In tests, pass a `FixedClock(Instant.parse("2025-01-01T00:00:00Z"))` and a `TestDispatcher` from the same `runTest` scope.

## MockK Tips (when you do reach for it)

- Prefer `mockk<T>(relaxed = true)` only when the test genuinely doesn't care about return values; otherwise stub explicitly so the test fails when the contract changes.
- Use `coEvery { ... } returns ...` for `suspend` functions; `every { ... }` will compile but won't match.
- Prefer `slot<T>()` over `coVerify` to capture and **assert on the input** — that's behavior. Asserting that a method was *called* is implementation testing.
- Never `coVerify` on your own UseCases / ViewModels. If you feel the need to, you're testing HOW. Assert on the resulting `UiState` or `Flow` emission instead.
