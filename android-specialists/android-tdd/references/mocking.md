# When (and where) to mock — Kotlin

> Reference for [`android-tdd`](../SKILL.md). Kotlin-native rewrite of mattpocock/skills `tdd/mocking.md` (MIT).
> This is the *decision* (when/where) + design-for-substitutability. The **double taxonomy** (dummy/fake/
> stub/spy/mock), fakes, and test-fixture placement live in `android-testing-expert §8` — don't duplicate them here.

## Mock only at system boundaries
Substitute the things you don't control or can't make deterministic:
- **Network** — Retrofit/OkHttp / Ktor / your API client (never hit the live network in unit tests; prefer `MockWebServer` for wire-level)
- **Persistence** — Room / DataStore / a remote DB (prefer an **in-memory Room DB** or a fake DAO over a mock when feasible)
- **Time & randomness** — `System.currentTimeMillis()`, `Clock`, `UUID.randomUUID()`, RNG → inject a fixed clock/value (deterministic tests only)
- **Coroutine dispatch** — inject a `TestDispatcher` / dispatcher provider; never let production `Dispatchers.IO`/`Main` leak into a unit test
- **File system** — sometimes; prefer a temp directory

**Don't mock what you own** — your own types, internal collaborators, value logic. Mocking internals produces
tests coupled to structure that pass while production breaks (see [tests.md](tests.md)). Prefer the real thing:
real > fake (in-memory) > stub > mock.

## Design for substitutability

**1. Inject dependencies — don't construct them inside.**
```kotlin
// Easy to substitute: the dependency comes in
suspend fun processPayment(order: Order, client: PaymentClient): Receipt =
    client.charge(order.total)

// Hard: builds its own concrete dependency — nothing to swap in a test
suspend fun processPayment(order: Order): Receipt {
    val client = StripeClient(apiKey = Secrets.STRIPE_KEY)   // unmockable
    return client.charge(order.total)
}
```
Inject via the project's `di` (constructor injection through Hilt/Koin, or an interface + test double). A change
that's *hard to test* is a design signal — fix the seam, don't skip the test.

**2. Prefer SDK-style interfaces over one generic `perform()`.**
One method per operation → each is independently stubbable with a single fixed return; no conditional logic
inside the double, and a test's dependencies are visible from the interface it stubs.
```kotlin
// GOOD: one method per operation
interface OrdersApi {
    suspend fun user(id: UserId): User
    suspend fun orders(userId: UserId): List<Order>
    suspend fun createOrder(draft: OrderDraft): Order
}

// BAD: a single generic entry point forces if/else branching inside every stub
interface GenericApi {
    suspend fun request(endpoint: Endpoint, options: RequestOptions): ByteArray
}
```
Benefits: each stub (or `coEvery { … } returns …`) returns one concrete shape · no branching in test setup ·
the interface documents which operations a unit actually uses · type-safe per operation.

Prefer a hand-written **fake** implementing the interface over a MockK mock when the collaborator has state or
several call sites — it stays honest across refactors. Reach for `mockk`/`coEvery`/`verify` only for pure
boundary stubs you don't own.
