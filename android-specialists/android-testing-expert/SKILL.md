---
name: android-testing-expert
description: "Expert Android testing guidance across the whole stack — JUnit4 vs JUnit5, unit (JVM) vs instrumented tests, Robolectric, test doubles/DI (MockK, Hilt), coroutine/Flow async testing, Compose UI + Espresso, parallelization & flakiness. Fires when work touches tests (@Test, @RunWith, MockK mockk/every/coEvery/verify, Turbine test/awaitItem, runTest, TestDispatcher, MainDispatcherRule, Truth assertThat, Espresso onView, createComposeRule/onNodeWithTag, Robolectric, @HiltAndroidTest, src/androidTest)."
---

# Android Testing Expert (JUnit + coroutines-test + MockK + Compose/Espresso)

> **Generated skill** — original wording, consolidated by `android-skill-consolidate`; provenance,
> sources, and licenses live in `SOURCES.yaml`. Don't hand-edit — change `SOURCES.yaml` and re-consolidate.

**Source of truth:** **Android Developers' official testing documentation** — [Test apps on Android](https://developer.android.com/training/testing),
[kotlinx-coroutines-test](https://developer.android.com/kotlin/coroutines/test), [Compose testing](https://developer.android.com/develop/ui/compose/testing),
plus [MockK](https://mockk.io/) and [Truth](https://truth.dev/) docs — wins on any conflict.
Community sources supply practical idioms and mistake patterns, not semantics.
This skill owns **framework mechanics, doubles/DI, and migration**; it **pairs with `android-tdd`**,
which owns the red→green→refactor discipline and Prove-It reproduction tests. Reach for `android-tdd`
for *process*, this skill for *how the frameworks work*.
Profile: `.claude/android-profile.md` — worktrees don't inherit it; fall back to
`$(git rev-parse --path-format=absolute --git-common-dir)/../.claude/android-profile.md`.

---

## 1. When to use which framework (+ coexistence)
- **JUnit4 is still the Android default** and is effectively mandatory for **instrumented** tests: AndroidX Test's runner (`AndroidJUnitRunner`), `@RunWith(AndroidJUnit4::class)`, Robolectric (`@RunWith(RobolectricTestRunner::class)`), Espresso, and most rules (`ActivityScenarioRule`, `HiltAndroidRule`, Compose test rules) are JUnit4 `TestRule`/`Runner` machinery with **no JUnit5 instrumented equivalent** — they **stay** on JUnit4. [Android: testing]
- **JUnit5 (Jupiter)** is fine for **pure-JVM** module/logic tests (`@Test`/`@Nested`/`@ParameterizedTest`, `@DisplayName`, `@BeforeEach`) — richer parameterization and nesting. It needs the `android-junit5` Gradle plugin, and its runner **does not run on device**. So: JUnit5 for `src/test` JVM logic, JUnit4 for anything touching a device/emulator or an androidTest rule.
- The two **coexist in one module** (`vintage` engine runs JUnit4 tests under JUnit5) — migrate incrementally, never big-bang. Don't rewrite green JUnit4 suites just to change frameworks; write *new* JVM tests in whichever the profile/module already uses. **Match the module's existing framework** unless asked to switch.
- **Assertions are orthogonal to the runner.** Prefer **Truth** (`assertThat(x).isEqualTo(y)`, fluent, best failure messages) or **kotlin.test** (`assertEquals`, multiplatform-friendly); plain JUnit `assertEquals(expected, actual)` works but has weaker diagnostics and **argument order is expected-first** (a classic bug source). Pick one per module and stay consistent.

## 2. Unit (JVM) vs instrumented — the fundamental split
- **Unit tests** live in **`src/test`**, run on the **local JVM** (`./gradlew testDebugUnitTest`) — fast, no device, but the `android.jar` is a **stub that throws** on real calls (`Log`, `Uri.parse`, `Color`…). **Prefer these**: put business logic, ViewModels, use-cases, mappers, and repositories in JVM unit tests. [Android: fundamentals]
- **Instrumented tests** live in **`src/androidTest`**, run on a **device/emulator** (`./gradlew connectedDebugAndroidTest`) with a real framework — slow, flakier, needed for Espresso/Compose UI, real SQLite/Room integration, and code that genuinely needs the platform. Reach for them only when a JVM test can't cover it.
- **Robolectric** runs Android-framework code **on the JVM** (`@RunWith(RobolectricTestRunner::class)` or `@Config`) — use it to test things that touch `Context`/resources/`Log`/shadows **without an emulator**. It's still a `src/test` unit test. Prefer real logic in plain JUnit over Robolectric where you can; reach for Robolectric when you'd otherwise need a device just for framework glue.
- Don't smuggle framework calls into JVM unit tests expecting them to work — either inject an abstraction, or move the test to Robolectric/androidTest. `Log`/`Uri`/`TextUtils` returning `0`/null is the stub, not your bug.

## 3. Assertions & test structure
- Declare with `@Test fun userCanLogOut() { … }`; name for triage — backtick names read well (`` @Test fun `adding an item increases the cart count`() ``) or `@DisplayName` on JUnit5. No subclassing needed. A test with no assertion is assumed to pass.
- **Lifecycle**: JUnit4 `@Before`/`@After` (per test) + `@BeforeClass`/`@AfterClass` (static, per class); JUnit5 `@BeforeEach`/`@AfterEach` + `@BeforeAll`/`@AfterAll`. JUnit4 makes a **fresh instance per test method** (avoids accidental shared state) — keep it that way; don't hoist mutable state to the class.
- **Truth**: `assertThat(list).containsExactly(a, b).inOrder()`, `.hasSize(n)`, `.isInstanceOf(...)`, `.isNull()` — fluent, subject-specific, best diagnostics. **kotlin.test**: `assertEquals`, `assertTrue`, `assertNull`, `assertIs<T>`. Don't assert a boolean you computed yourself (`assertThat(a == b).isTrue()`) — assert the values (`assertThat(a).isEqualTo(b)`) so the failure shows both sides.
- **Exceptions**: `assertThrows(IllegalStateException::class.java) { … }` (JUnit4/5) or kotlin.test `assertFailsWith<T> { … }` — **returns the caught exception** for further assertions (message/cause). Assert the **specific** type, not bare `Exception`. For "must not throw", just call it — an escaping exception fails the test.

## 4. Parameterized tests
- **JUnit4**: `@RunWith(Parameterized::class)` + `@Parameterized.Parameters` supplying an `Array<Array<Any>>` — verbose, one runner per class, and it **can't combine** with another `@RunWith` (e.g. Robolectric) without a delegating runner. **JUnit5** is far nicer: `@ParameterizedTest` + `@ValueSource`/`@CsvSource`/`@EnumSource`/`@MethodSource`, each argument row its **own reported case** (failure names the exact input).
- Prefer a parameterized case over an in-test `for` loop (a loop stops at the first failure and hides which input broke) and over copy-pasted near-identical tests.
- Use **concrete literal expectations**, not values derived from the same logic under test (that masks bugs). Keep argument rows inline/close to the test. Watch mismatched-length argument arrays — align rows as tuples, don't zip two lists that can silently truncate.

## 5. Filtering, disabling & timeouts
- **Categorize/filter**: JUnit5 `@Tag("networking")` (filter via `includeTags`/`excludeTags` or Gradle `useJUnitPlatform { includeTags(...) }`); JUnit4 `@Category`. Instrumented tests use AndroidX annotations `@SmallTest`/`@MediumTest`/`@LargeTest` and `@RunWith`-level filtering — drive sharding/CI selection from these, not by commenting tests out.
- **Skip with a reason, don't comment out**: JUnit4 `@Ignore("reason")`; JUnit5 `@Disabled("reason")` / `@EnabledIf`/`assumeTrue(...)` (assumptions abort-as-skipped, not fail). The reason shows in reports; commented-out tests rot silently.
- **Timeouts**: JUnit5 `@Timeout(1, SECONDS)` or JUnit4 `@Test(timeout = …)`/`Timeout` rule — but for coroutine code prefer `runTest`'s built-in timeout + virtual time over wall-clock timeouts (§6). Never gate a test on `Thread.sleep` (§9).

## 6. Async — coroutine & Flow testing
- Wrap suspend/Flow tests in **`runTest { … }`** (from `kotlinx-coroutines-test`) — it runs on a `TestScope` with a **virtual clock**, so delays are skipped and `advanceUntilIdle()`/`advanceTimeBy()`/`runCurrent()` drive time deterministically. Never `runBlocking` a coroutine test you can `runTest`. [Android: coroutines-test]
- **Dispatcher choice**: `StandardTestDispatcher` (default in `runTest`) **queues** new coroutines — you must `advanceUntilIdle()`/`runCurrent()` to let them run; use it when you want to assert intermediate states step by step. `UnconfinedTestDispatcher` starts eagerly (no advancing needed) — handy for simple "fire and assert" cases and for collectors that must start immediately.
- **Swap `Dispatchers.Main`** with a `MainDispatcherRule` (`Dispatchers.setMain(testDispatcher)` in `@Before`, `resetMain()` in `@After`) — required for any ViewModel using `viewModelScope`. **Inject dispatchers** (constructor param, default `Dispatchers.IO`) instead of hardcoding them, so tests pass a `TestDispatcher`; never hardcode `Dispatchers.IO`/`Default` inside code you need to test.
- **Flow**: use **Turbine** — `flow.test { assertThat(awaitItem()).isEqualTo(…); awaitComplete() }` — `awaitItem`/`awaitError`/`awaitComplete`/`cancelAndIgnoreRemainingEvents`; it **fails if events are left unconsumed** (catches missed emissions). For hot `StateFlow`/`SharedFlow`, collect in the test scope (`backgroundScope`) so it cancels cleanly. Prefer Turbine over manual `toList()`/collector jobs.
- **LiveData** (legacy): add `InstantTaskExecutorRule` so `setValue`/`postValue` run synchronously; use `getOrAwaitValue` helpers, not `Thread.sleep`.

## 7. Compose UI testing
- **`createComposeRule()`** for a Compose-only tree (JVM via Robolectric or on-device), **`createAndroidComposeRule<Activity>()`** when you need a real Activity/host. Set content with `composeTestRule.setContent { … }`.
- **Find** via semantics: `onNodeWithTag("cart_button")`, `onNodeWithText(...)`, `onNodeWithContentDescription(...)`; **assert** `assertIsDisplayed()`/`assertIsEnabled()`/`assertTextEquals(...)`; **act** `performClick()`/`performTextInput(...)`/`performScrollTo()`. Prefer stable **`testTag`s** (`Modifier.testTag(...)`) for hooks — pull the convention from the profile's **`test_tags`** field; don't match on user-visible copy that localization will change.
- Compose test rules **auto-sync** with the recomposition/animation clock — don't `Thread.sleep`. For genuinely async UI, use **`waitUntil { … }`** or `waitForIdle()`, and `mainClock.autoAdvance`/`advanceTimeBy` to control animations deterministically.
- Test the **ViewModel/state** for logic; use Compose UI tests for rendering, interaction, and semantics — not to re-verify business rules a JVM test already covers.

## 8. Test doubles, DI & fixtures
- Double taxonomy (Fowler): Dummy · **Fake** (working shortcut, e.g. in-memory repo) · Stub (canned returns) · Spy (records calls) · Mock (self-verifying). **Prefer fakes over mocks for repositories/data sources** — a hand-written in-memory fake is less brittle than a wall of `every {}`/`verify {}` and survives refactors. **Prefer state verification over behavior verification**; reserve `verify` for genuine interaction contracts.
- **MockK** (Kotlin-native): `mockk<Repo>()` / `mockk(relaxed = true)`; stub with `every { repo.load() } returns x` and **`coEvery { repo.loadAsync() } returns x`** for suspend funs; verify with `verify { … }` / **`coVerify { … }`**, `verify(exactly = n)`, `verifyOrder`. Use `slot`/`capture` for argument capture. `relaxed`/`relaxedUnitFun` cut boilerplate but hide unstubbed calls — prefer explicit stubs where the return matters. Don't mock types you own and can fake cheaply; don't mock data classes/value types.
- **Constructor injection** is the enabler — pass collaborators (repos, dispatchers, clocks) in, so tests supply doubles. Never reach for a hidden singleton/`object`/global inside testable code.
- **Hilt tests**: `@HiltAndroidTest` + `HiltAndroidRule` (ordered before the Compose/Activity rule); swap bindings with **`@BindValue`** fields or a `@TestInstallIn` module replacing the production module; call `hiltRule.inject()` in `@Before`. Use a `HiltTestApplication` (custom `AndroidJUnitRunner`). Inject **fake dispatchers/clocks** the same way.
- **Deterministic fixtures only** — no `System.currentTimeMillis()`/`Date()`/`Random`/`UUID.randomUUID()` in fixtures; inject a fixed `Clock`/seeded RNG. Give fixture factories a default for every field (`fun user(id: String = "1", …)`) so tests set only what matters. Keep fakes/fixtures in `testFixtures`/a shared test module, not copy-pasted per test.

## 9. Parallelization & flakiness
- Assume tests run in **arbitrary order and (for JVM) in parallel** — so **no order dependence, no shared mutable static/`object` state, no reliance on a previous test's side effects**. Reset singletons and clear DataStore/`SharedPreferences`/Room between tests. This surfaces hidden coupling and speeds CI. [Android: testing]
- **Flakiness checklist**: no `Thread.sleep`-based waiting (use idling/`waitUntil`/virtual time) · deterministic dispatchers (`TestDispatcher`, not real `IO`/`Default`) · fixed clock/RNG · no real network (fake the API / `MockWebServer`) · no real system time or locale/timezone assumptions · isolate shared emulator state.
- **Instrumented isolation**: enable the **Android Test Orchestrator** (`clearPackageData: true`) so each test runs in its own instrumentation with cleared app data — kills cross-test state leakage on device. For Espresso async, register **`IdlingResource`**s (or use Espresso's built-in sync) rather than sleeping.
- When a test is genuinely, temporarily flaky, quarantine it explicitly (`@Ignore("flaky — TICKET-123")` or `@FlakyTest`) with a tracked reason and a plan to fix — don't leave a silent `sleep`-padded test that fails 1-in-20.

## 10. Instrumented essentials — Espresso / UI Automator / screenshot (still needed)
- **Espresso** (in-app View UI): `onView(withId(R.id.x)).perform(click()).check(matches(isDisplayed()))`; `onData(...)` for adapter views; auto-syncs with the main thread + idling resources — never `sleep`. Use it for XML/View-based screens (Compose uses §7). [Android: espresso]
- **UI Automator** for **cross-app / system UI** (permission dialogs, notifications, other apps) — `UiDevice`/`By.res(...)`/`bySelector` — the only option when a flow leaves your process.
- **Screenshot / snapshot testing**: **Paparazzi** renders Composables/Views **on the JVM** (no emulator, fast, deterministic) — great for CI; **Roborazzi** does screenshot tests via Robolectric and reuses your Compose/Espresso interactions. Commit golden images; pin a fixed device config/font-scale so diffs are meaningful.
- **Keep instrumented** (§2): real device integration, Espresso/UI Automator, and platform-dependent flows that JVM/Robolectric can't faithfully cover.

## 11. Coexistence & migration (JUnit4 → JUnit5, JVM modules)
| JUnit4 | JUnit5 (Jupiter) |
|---|---|
| `@RunWith(...)` + `Runner` | (no runner) `@ExtendWith(...)` extensions |
| `@Before` / `@After` | `@BeforeEach` / `@AfterEach` |
| `@BeforeClass` / `@AfterClass` (static) | `@BeforeAll` / `@AfterAll` |
| `@Ignore("reason")` | `@Disabled("reason")` |
| `@RunWith(Parameterized::class)` + `@Parameters` | `@ParameterizedTest` + `@MethodSource`/`@CsvSource` |
| `@Test(expected = X::class)` | `assertThrows(X::class.java) { … }` |
| `@Rule TestRule` | `@ExtendWith` / `@RegisterExtension` |
| `@Test(timeout=…)` | `@Timeout(…)` |
| nested classes via runner | `@Nested` inner classes |

**Instrumented / androidTest tests stay on JUnit4** — AndroidX rules, Robolectric, Espresso, Compose, and Hilt test all need the JUnit4 runner/rule model; there is no on-device JUnit5 runner. Migrate **only pure-JVM** suites, leaf-first, one reviewable module at a time, running the `vintage` engine so unconverted JUnit4 tests keep passing during transition. **No float tolerance is built into equality** — use Truth's `assertThat(x).isWithin(tol).of(y)` for doubles.

## 12. Common mistakes / anti-patterns (completion criterion)
A critique pass is done only when the bolded rules in §§1–9 and the items below have each been
**ruled in or out** against the diff. This section lists only what those rules don't cover.
- Framework calls (`Log`, `Uri.parse`, `Color`) in a plain JVM unit test (belongs in Robolectric/androidTest); JUnit5 annotations on an instrumented test (won't run); `assertEquals(actual, expected)` argument order flipped.
- `runBlocking`/`Thread.sleep` where `runTest` + virtual time belongs; hardcoded `Dispatchers.IO`/`Default` instead of an injected `TestDispatcher`; missing `MainDispatcherRule` for a `viewModelScope` ViewModel; unconsumed Turbine events.
- Over-mocking a repository you could fake; `relaxed = true` masking an unstubbed call that mattered; asserting on localized text instead of a `testTag`; testing a Composable to re-verify logic a ViewModel test already covers.

## Currency
Version-gated: JUnit5 on Android needs the `android-junit5` Gradle plugin (JUnit4 needs none); `kotlinx-coroutines-test` APIs changed across `1.6`/`1.7` (`runTest`, `StandardTestDispatcher`, `advanceUntilIdle` replaced the old `runBlockingTest`/`TestCoroutineDispatcher` — don't use the deprecated forms); MockK, Turbine, Truth, Paparazzi/Roborazzi versions track the profile's `kotlin_version`/AGP. Gate examples by the profile. Maintainer notes + doc refs live in `SOURCES.yaml`.
