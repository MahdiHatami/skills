# Test Pyramid: Picking the Right Tier

The TDD loop demands fast deterministic feedback. On Android, the test tier you choose decides whether the loop is 100ms or 60s. Always pick the cheapest tier that can answer the question.

## The four tiers

| Tier | Typical run time | When to use |
|---|---|---|
| **JVM unit** (JUnit 4 + MockK) | < 50ms | Pure Kotlin: domain logic, mappers, `UiState` reducers, Flow operators, use cases. **Default.** |
| **Robolectric** (`@RunWith(RobolectricTestRunner::class)`) | ~1s warm | Android-framework code without an emulator: `Context`, `Resources`, `SharedPreferences`, lifecycle, `Intent`. |
| **Compose UI** (`runComposeUiTest`) | ~few s | Composable behavior: state→render, gesture input, semantic tree assertions. Run on JVM via `runComposeUiTest` — no device. |
| **Instrumented** (Espresso / device) | tens of s+ | True device-coupled behavior: real `WindowManager`, real network, real Camera/sensors, real Room migration. |

## Decision tree

```
What does the code under test depend on?

├── Only Kotlin / kotlinx
│   └── JVM unit test
│
├── Android framework types but no rendering
│   ├── Can the dependency be faked (e.g. SharedPreferences interface, FakeContext)?
│   │   └── JVM unit test with the fake
│   └── Otherwise → Robolectric
│
├── A Composable's render or input
│   └── Compose UI test (runComposeUiTest)
│
└── Real device hardware, real network, real WindowManager
    └── Instrumented (last resort in the TDD loop)
```

## Common mistakes

### Writing instrumented tests for ViewModels

A ViewModel is pure Kotlin (assuming you didn't hold a `Context` in it — see [interface-design.md](interface-design.md)). It belongs in the JVM tier. If the ViewModel needs `Application` for resource lookup, **move resource lookup behind an interface** so the test can pass a `FakeStringProvider`. Don't push the test up the pyramid because the production code is sloppy.

### Writing Compose UI tests for state logic

If you're testing "Action X produces UiState Y", that's a JVM ViewModel test. The Compose UI test exists to verify "given UiState Y, the Composable renders Z." Two cheap tests beat one expensive end-to-end one.

### Skipping Robolectric and going straight to instrumented

Robolectric is "good enough" for ~90% of Android-framework dependencies — the cases where it's *not* enough (real `WindowManager`, real native network) are rare and obvious. If you're using `SharedPreferences` or reading a string resource, Robolectric is fine.

### Forgetting `runComposeUiTest` exists

The old `createComposeRule()` ran on a device. `runComposeUiTest { }` (from `androidx.compose.ui.test`) runs in JVM and is dramatically faster. Use it.

## When you genuinely need an instrumented test

Acceptable reasons:

- The test exercises a real Room migration on a real SQLite.
- The test asserts on real `WindowInsets` / IME behavior.
- The test verifies BiometricPrompt or platform Camera capture.

In those cases, write the instrumented test, then ask: **can I extract the platform-coupled bit behind an interface so the rest of this code can be tested at JVM tier?** Usually yes.

## Rule of thumb

If your TDD loop is taking longer than ~5 seconds per cycle, the loop is broken. Find which tests are responsible and push them down the pyramid — or split them.
