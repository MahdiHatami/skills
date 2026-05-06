# Interface Design for Testability (Kotlin / Android)

Good interfaces make testing natural:

## 1. Accept dependencies, don't create them

```kotlin
// Testable
class ProcessOrder(
    private val payments: Payments,
    private val clock: Clock,
)

// Hard to test
class ProcessOrder {
    private val payments = StripePayments()  // ← can't substitute a fake
}
```

## 2. Return values; don't mutate hidden state

```kotlin
// Testable: pure function, easy assertion
fun calculateDiscount(cart: Cart): Discount

// Hard to test: mutates the input, no return
fun applyDiscount(cart: Cart) {
    cart.total -= discount
}
```

For ViewModels, the same rule: an `Action` produces a new `UiState` value. The state is observable; the trajectory is the test.

## 3. Model `UiState` as a sealed hierarchy, not a bag of nullables

```kotlin
// GOOD: state shapes are exhaustive, mutually exclusive
sealed interface UiState {
    data object Idle : UiState
    data object Submitting : UiState
    data class Confirmed(val orderId: String) : UiState
    data class Failed(val message: String) : UiState
}

// BAD: every test has to assert combinations of nullables
data class UiState(
    val isLoading: Boolean = false,
    val orderId: String? = null,
    val error: String? = null,
)
```

A sealed `UiState` makes test assertions one-liners: `assertEquals(UiState.Confirmed("ord_1"), awaitItem())`. The bag-of-nullables version requires multi-field equality with implicit invariants ("when `isLoading` is true, `error` must be null") that are not enforced anywhere.

## 4. Expose `Flow`/`StateFlow`, not callbacks

```kotlin
// GOOD: testable with Turbine, composable with operators
class UserRepository {
    fun observeUser(id: String): Flow<User>
}

// BAD: callback hell, race-prone tests
class UserRepository {
    fun observeUser(id: String, listener: (User) -> Unit): Subscription
}
```

## 5. Small surface area

Fewer methods = fewer tests needed. Fewer params = simpler test setup.

If a class exposes 12 methods, ask which ones callers actually use. The rest are probably implementation that leaked through the interface.

## 6. Don't smuggle Android types into business logic

```kotlin
// GOOD: pure Kotlin module, JVM unit tests run in millis
data class Reminder(val message: String, val triggerAt: Instant)

// BAD: forces every test through Robolectric
data class Reminder(val message: String, val triggerAt: Calendar)
//                                                    ^^^^^^^^ android.icu / android.text references
```

Same principle for `Bitmap`, `Drawable`, `Context`, `Uri` — keep them at the edge so the core can be tested at the cheapest tier.

## 7. Prefer `suspend` over `Result<T>` over throwing

For `suspend` functions, structured concurrency already cancels children on failure — exceptions are fine and the test asserts via `assertFailsWith<T> { ... }`. Reserve `Result<T>` for boundaries where you want callers to handle the failure explicitly without `try/catch` churn.

What to avoid: returning `T?` to signal "could fail". Tests then can't distinguish "no result yet" from "failed".
