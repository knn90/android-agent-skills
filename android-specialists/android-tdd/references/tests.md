# Good vs bad tests (Kotlin)

> Reference for [`android-tdd`](../SKILL.md). Kotlin-native rewrite of mattpocock/skills `tdd/tests.md` (MIT).
> The *one rule*: test observable behavior through the **public interface**, never internal structure.

## Good — integration-style, behavior through the public API
Exercises real code paths the way a caller would; survives internal refactors; the name says **what**, not how.

```kotlin
@Test
fun `user can checkout with a valid cart`() = runTest {
    val cart = Cart()
    cart.add(product)
    val result = checkout(cart, using = paymentMethod)
    assertThat(result.status).isEqualTo(Status.Confirmed)   // outcome the caller cares about
}
```
Characteristics: behavior callers care about · public API only · one logical assertion · survives refactors · describes WHAT.

## Bad — coupled to implementation
Breaks when you refactor even though behavior is unchanged — the signal it was testing *how*, not *what*.

```kotlin
// BAD: asserts an internal interaction, not an outcome
@Test
fun `checkout calls paymentService process`() = runTest {
    val payment = mockk<PaymentService>(relaxed = true)
    checkout(cart, using = payment)
    verify(exactly = 1) { payment.process(any()) }   // breaks on any refactor; proves nothing about the result
}
```
Red flags: mocking internal collaborators · testing private methods · asserting call counts/order · name describes HOW · verifying through a back channel instead of the interface.

## Verify through the interface, not a back channel
```kotlin
// BAD: reaches past the interface into the store
@Test
fun `createUser writes row`() = runTest {
    createUser(name = "Alice")
    val rows = db.query("SELECT * FROM users WHERE name = ?", "Alice")
    assertThat(rows).isNotEmpty()                    // couples the test to the schema
}

// GOOD: round-trips through the public API
@Test
fun `created user is retrievable`() = runTest {
    val created = createUser(name = "Alice")
    val fetched = userFor(created.id)
    assertThat(fetched.name).isEqualTo("Alice")
}
```

For Compose, the "interface" of a screen is its **view-model's exposed `StateFlow`/`UiState`** — test that,
not the `@Composable` (see `android-testing-expert §12`). For MockK matchers, parameterized cases, and Turbine
Flow assertions, see `android-testing-expert §3–5`.
