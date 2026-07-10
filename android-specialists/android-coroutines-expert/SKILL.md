---
name: android-coroutines-expert
description: "Expert Kotlin Coroutines & Flow reference — fires when work touches suspend, Dispatchers (Main/IO/Default), withContext, launch/async, coroutineScope/supervisorScope, viewModelScope/lifecycleScope, Job/cancellation, Flow/StateFlow/SharedFlow, collect/collectAsStateWithLifecycle, stateIn/shareIn, flowOn, Mutex, callbackFlow/suspendCancellableCoroutine, runTest, or structured-concurrency / main-safety review."
---

# Kotlin Coroutines Expert

> **Generated skill** — original wording, consolidated by `android-skill-consolidate`; provenance,
> sources, and licenses live in `SOURCES.yaml`. Don't hand-edit — change `SOURCES.yaml` and re-consolidate.

**Source of truth:** Coroutines are a *language + first-party library* feature, so the **Kotlin
Coroutines guide (kotlinlang.org), the `kotlinx.coroutines` API docs, and Android Developers'
"Coroutines best practices" / "Background work with coroutines" guides are the primary authority**
and win on any conflict. Community sources supply practical LLM-mistake patterns and review
heuristics, not semantics. **Baseline: `kotlinx.coroutines` 1.8+/1.9, Kotlin 2.x / K2 compiler,
`lifecycle-runtime-compose` for `collectAsStateWithLifecycle`.** Some APIs (`collectAsStateWithLifecycle`,
`SharingStarted.WhileSubscribed`, `runTest`/`StandardTestDispatcher`) are version-gated — gate by the
profile's `kotlin_version` / `async` fields and prefer the current, non-deprecated forms.
Profile: `.claude/android-profile.md` — worktrees don't inherit it; fall back to
`$(git rev-parse --path-format=absolute --git-common-dir)/../.claude/android-profile.md`.

---

## 1. Structured-concurrency model (the foundation)
- Every coroutine runs inside a **`CoroutineScope`** and carries a **`CoroutineContext`** = an indexed set of elements (**`Job`** + **dispatcher** + `CoroutineName` + `CoroutineExceptionHandler`). A child **inherits the parent context**, replacing only the `Job` with a new child `Job` — this parent→child link is what "structured" means. [Kotlin: coroutines-basics]
- The enforced invariant: **a scope does not complete until all its children complete**; **cancelling a scope cancels every child**; and a child's **uncaught failure cancels the parent and its siblings** (unless the parent uses a `SupervisorJob`). Cancellation and failure flow through the `Job` tree.
- **Concurrency ≠ parallelism, and a coroutine ≠ a thread.** A **dispatcher** decides which thread(s) run the work; suspension frees the thread. Thread **confinement**, not locks, is the default safety tool.
- Coroutines prevent **no** shared-mutable-state hazard for you — you still avoid data races via confinement / `Mutex` / atomics. And **cancellation is cooperative, not preemptive** — you still sequence and check correctly.

## 2. suspend functions & main-safety
- A `suspend` function marks **suspension points**; after resuming, work may continue on a **different thread** (never assume thread identity). `suspend` does **not** mean "off the main thread" — a `suspend` fun called on `Dispatchers.Main` runs on main until it hits a dispatcher switch. [Kotlin: composing-suspending-functions]
- **Main-safety is the contract:** a `suspend` function must be **safe to call from `Dispatchers.Main`** — it must not block. Push blocking/CPU work to `withContext(Dispatchers.IO)` / `withContext(Dispatchers.Default)` **inside** the function, so callers never need to know or wrap. [Android: coroutines best practices]
- **Never block a thread inside a coroutine** — no `Thread.sleep`, blocking sockets, synchronous disk/DB, or `runBlocking`. Use `delay` and suspending APIs; blocking starves the dispatcher's thread pool (and freezes the UI on Main → ANR).
- Prefer `suspend`/`Flow` over callbacks when both exist (the compiler sequences results; no forgotten callbacks). The outer `withContext` / `try` wraps the awaited work.

## 3. Shared mutable state & thread confinement — the #1 race
- Coroutines can run **concurrently on multiple threads** (`IO`/`Default` pools) — concurrent access to shared `var`/collection is a **data race**. Default fix: **confine** mutable state to one coroutine or one single-threaded dispatcher (`Dispatchers.Default.limitedParallelism(1)` / an actor-like `Channel`), not blanket locking.
- **The headline bug — check-then-act across a suspension.** Between `if (cache[k] == null)` and `cache[k] = load()` there is a suspension point, so another coroutine can interleave → **duplicate work + overwrite**. Fix: guard with a `Mutex`, or **dedup in-flight work by storing the `Deferred`/`Job`** and awaiting it.
- **`Mutex().withLock { }`** for suspend-friendly mutual exclusion — do **not** hold `synchronized`/`ReentrantLock` across a suspension point (it blocks the thread and the lock isn't tied to the coroutine). **Kotlin's `Mutex` is NOT reentrant** — re-locking it from the same coroutine deadlocks.
- For simple counters/flags use **atomics** (`AtomicInteger`, `atomicfu`) or **`@Volatile`** for single-writer visibility. `@Volatile` gives **visibility, not atomicity** of compound ops (`x = x + 1` still races) — use an atomic's `updateAndGet`/`compareAndSet`.

## 4. Dispatchers.Main & UI state
- **Update UI state only on the main thread.** `viewModelScope` and `lifecycleScope` default to **`Dispatchers.Main.immediate`** — launch there, switch **out** to `IO`/`Default` for work, and let state emission land back on main. [Android: coroutines best practices]
- **`Dispatchers.Main.immediate`** runs synchronously if already on main (skips an extra dispatch / a frame of latency) — prefer it over `Dispatchers.Main` for state emissions that may already be on main.
- `MutableStateFlow.value = …` is **thread-safe to set from any thread**, but Compose/`LiveData`/View bindings expect main — keep the *coroutine that drives UI state* main-confined rather than sprinkling `withContext(Main)` around each emission.
- **Don't blanket-wrap a whole ViewModel method in `withContext(Dispatchers.IO)`** to silence a "blocking on main" worry — wrap **only the blocking call**; keep the coroutine main by default so state emission stays main (the coroutine analog of "never blanket-apply an isolation annotation").

## 5. Flow context preservation & capture
- **Context preservation:** a `Flow`'s `emit` must run in the **same coroutine that runs the builder** — emitting from a different coroutine (`flow { launch { emit(x) } }` or from a callback thread) throws **`IllegalStateException: Flow invariant is violated`**. When you must emit from multiple coroutines / a callback, use **`channelFlow { send(x) }`** (or `callbackFlow`), which is concurrency-safe. [Kotlin: flow]
- Change a flow's **upstream** execution context with **`flowOn(dispatcher)`** — it affects operators *upstream* of it, not the collector. **Never call `withContext` inside a `flow { }` builder** to change emit context (it violates preservation); use `flowOn`.
- **Captured values, not snapshots:** a coroutine closure captures a **reference** to a `var`, so a value mutated after `launch` is seen by the coroutine. Snapshot loop indices into a `val` — the `for (i in xs) launch { use(i) }` capture pitfall (all coroutines may see the final `i` on some targets) → `for (i in xs) { val n = i; launch { use(n) } }`.

## 6. Dispatchers & offloading work
- **`withContext(dispatcher) { }` switches the dispatcher for a block and switches back** — it does **not** create a new scope or coroutine, so it stays structured and can't leak. Use it (not `launch(IO)` inside) to move work off main.
- **`Dispatchers.IO`** for **blocking I/O** (network, disk, DB) — a large elastic pool sized for blocking. **`Dispatchers.Default`** for **CPU-bound** work (parsing, sorting, bitmap/JSON transforms) — a pool sized to cores. Don't run CPU work on `IO` (starves it) or blocking I/O on `Default` (starves the CPU pool). `Dispatchers.Main` for UI only.
- **Inject dispatchers** via a `DispatcherProvider` (constructor param defaulting to the real dispatcher) so tests substitute a `TestDispatcher` — **never hardcode `Dispatchers.IO`/`Main`/`Default`** deep inside code you need to test.
- **Don't ping-pong dispatchers** — each `withContext` has a real cost; batch a loop's work inside **one** `withContext(Default)` rather than switching per item.

## 7. Structured concurrency in practice
- **Prefer `coroutineScope` / `supervisorScope` over detached launches.** `coroutineScope { }`: any child failure **cancels all children and rethrows** (all-or-nothing). `supervisorScope { }`: children **fail independently**. Use `async` + `awaitAll()` for a fixed parallel decomposition that returns values; `coroutineScope { launch; launch }` for fan-out side effects. [Kotlin: composing-suspending-functions]
- **`GlobalScope` is banned** — it has **no lifecycle**, leaks coroutines, and escapes structured cancellation. Use a lifecycle scope (`viewModelScope`, `lifecycleScope`) or a scope **you own and cancel**.
- **`launch`/`async` in a loop without an enclosing scope is a smell** (loses joined completion + cancellation) → `xs.map { async { work(it) } }.awaitAll()` for results, or a `coroutineScope { }` wrapping the launches for side effects, so all are awaited and cancellation propagates.
- **`async` surfaces exceptions on `await`**, not at the launch site — but a failing `async` **inside a `coroutineScope` cancels the scope even before you `await`**. A root `async` in `supervisorScope`/an owned scope must have its exception handled **at `await`** (wrap in `try/catch`), or it's lost.

## 8. Cancellation (cooperative)
- **Cancellation is cooperative.** `cancel()` moves the `Job` to *cancelling*; the coroutine actually stops only at a **suspension point** or when it checks. All `kotlinx` suspend funcs (`delay`, `withContext`, `yield`, channel ops) **check and throw `CancellationException`**. A pure **CPU loop with no suspension never cancels** unless you call **`ensureActive()`**, check **`isActive`**, or **`yield()`**. [Kotlin: cancellation-and-timeouts]
- **Don't swallow `CancellationException`** — a broad `catch (e: Exception)` (or `catch (e: Throwable)`) eats it and **breaks structured cancellation** (loops run forever, cleanup misfires). **Rethrow it:** `catch (e: CancellationException) { throw e }` first, or catch only the specific exceptions you handle.
- Run cleanup that **must survive cancellation** in **`withContext(NonCancellable) { }`** — only for finalization/rollback, **never** to make ordinary work uncancellable.
- **Structured children cancel automatically** with the parent; a **scope you own** (`CoroutineScope(...)`) must be **cancelled explicitly**. `viewModelScope` auto-cancels in `onCleared`; `lifecycleScope` with the lifecycle — don't hand-roll a scope you forget to cancel.

## 9. Bridging callbacks (continuations)
- Wrap a one-shot callback/listener API with **`suspendCancellableCoroutine { cont -> … }`** and **resume EXACTLY once on every path**: `cont.resume(v)` / `cont.resumeWithException(e)`. **Zero** resumes → the coroutine **hangs forever**; a **second** resume throws `IllegalStateException: Already resumed`. Audit early returns, error callbacks, and "callback may never fire" timeouts. [Kotlin: coroutines interop]
- Use the **`Cancellable`** variant (not `suspendCoroutine`) so cancellation propagates — register **`cont.invokeOnCancellation { }`** to unregister the listener / cancel the underlying call, and guard late callbacks with `if (cont.isActive)`.
- **Don't wrap an API that already has a suspend/Flow adapter** — Retrofit `suspend` functions, Room `suspend`/`Flow` DAOs, and Play-Services `Task.await()` exist; use them. For a **stream** of callbacks use **`callbackFlow { … awaitClose { } }`**, not a single continuation.

## 10. Flow — cold vs hot, StateFlow / SharedFlow
- **`Flow` is cold:** the builder runs **per collector**, from scratch, on `collect` — nothing happens until a terminal operator (`collect`/`first`/`toList`). Intermediate operators (`map`/`filter`/`transform`) are cold and lazy. [Kotlin: flow]
- **Hot streams:** **`StateFlow`** (always has a value, conflated, deduped by `equals` — for *state*) and **`SharedFlow`** (configurable `replay`/`extraBufferCapacity` — for *events*, `replay = 0`). Convert cold→hot with **`stateIn`/`shareIn(scope, SharingStarted.WhileSubscribed(5_000), initial)`** — `WhileSubscribed(5000)` keeps upstream alive **5 s after the last collector** (survives config changes) then stops; `Eagerly`/`Lazily` **never stop upstream** (leak risk).
- **Back-pressure:** a slow collector suspends the emitter by default. **`buffer(n)`** decouples them; **`conflate()`** keeps only the latest; **`collectLatest`/`flatMapLatest`** cancel stale work when a new value arrives; **`distinctUntilChanged()`** drops repeats (`StateFlow` already dedupes).
- **Combining:** `combine` (latest of each), `zip` (pairwise), `flatMapLatest`/`flatMapMerge`/`flatMapConcat`. **Handle errors with the `catch` operator** placed **downstream of the failing operator** (`catch` only sees *upstream* errors) — don't `try/catch` around `collect` for upstream failures; use `catch` (+ `onCompletion` for cleanup). `catch` above a `flowOn` still respects context.
- **`SharedFlow` config is load-bearing:** set `replay`/`extraBufferCapacity`/`onBufferOverflow` deliberately — an oversized replay/buffer retains memory; an **event** flow should be `replay = 0` (else late collectors re-see old events). A **hot flow collected in a scope that outlives its consumer leaks** — scope it to a lifecycle.

## 11. CoroutineContext & scoped values
- `CoroutineContext` combines elements with `+`; children inherit and override **per-launch**: `launch(Dispatchers.IO + CoroutineName("sync")) { }`. `coroutineContext[Job]`/`[CoroutineName]` read them. Don't fabricate a new scope just to change context — pass context to `launch`/`withContext`.
- **Propagate a `ThreadLocal` into coroutines with `threadLocal.asContextElement(value)`** — a plain `ThreadLocal` does **not** follow suspension across threads; the context element re-establishes it on each dispatch. Set it in the context; don't mutate globals mid-coroutine.
- Use a **custom `CoroutineContext.Element`** (or `asContextElement`) to inject scoped/cross-cutting values (request id, trace, test config) instead of shared mutable globals — the coroutine-scoped analog of dependency injection. Reads are cheap; keep the element **immutable** so children can't corrupt a parent's value.

## 12. Lifecycle-aware collection & incremental adoption
- **Collect UI flows lifecycle-aware:** `repeatOnLifecycle(Lifecycle.State.STARTED) { flow.collect { } }` (or `flowWithLifecycle`) in Views/Fragments, and **`collectAsStateWithLifecycle()`** in Compose — a plain `lifecycleScope.launch { flow.collect { } }` keeps collecting while the UI is in the background, **wasting work and risking crashes**. [Android: coroutines best practices]
- **Scope every coroutine to a lifecycle:** `viewModelScope` (cancels in `onCleared`), `lifecycleScope`, or a scope you cancel. Never `GlobalScope`; never launch from something that outlives the work.
- **Avoid `runBlocking` in production** — it blocks the calling thread and defeats structured concurrency; on the main thread it **ANRs**. It's for `main()` entrypoints and tests only.
- **Adoption, not a compiler flag:** coroutine semantics don't change with K2 — the discipline is **main-thread confinement + structured scoping**. Migrate incrementally and reviewably: wrap callback APIs (`suspendCancellableCoroutine`/`callbackFlow`), move blocking work behind main-safe `suspend` funcs, replace `LiveData`/`RxJava` with `StateFlow`/`Flow` **feature-by-feature** — never big-bang.

## 13. Testing async code
- Framework choice and mechanics → **android-testing-expert**. Coroutine-specific: run tests in **`runTest { }`** (`kotlinx-coroutines-test`) — a `TestScope` with a **virtual clock**, so `delay` is skipped and `advanceUntilIdle()`/`advanceTimeBy()`/`runCurrent()` drive time deterministically. **Never `Thread.sleep` or real `delay` to "wait"** (flaky); never `runBlocking` a test you can `runTest`.
- **Swap `Dispatchers.Main`** with a `MainDispatcherRule` (`Dispatchers.setMain`/`resetMain`) for any `viewModelScope` code; **inject a `TestDispatcher`** rather than hardcoding `IO`/`Default`. Choose `StandardTestDispatcher` (queues — you `advanceUntilIdle()` to assert intermediate state) vs `UnconfinedTestDispatcher` (eager — "fire and assert") deliberately. Use **Turbine** for `Flow` (`flow.test { awaitItem() }`).
- Test **cancellation** via the real mechanism — assert a `CancellationException` propagates / that the work actually stopped — not via sleeps. Depth on doubles/DI/Turbine/Hilt-test lives in **android-testing-expert** §6/§8.

## 14. Anti-pattern checklist (completion criterion)
A critique pass is done only when **every anti-pattern below has been ruled in or out** against the diff:
- Shared mutable state read/written from multiple coroutines without `Mutex`/confinement/atomics; check-then-act across a suspension; non-reentrant `Mutex` re-locked (§3)
- Blocking a thread — `Thread.sleep`, blocking I/O, `runBlocking` on main (§2/§12)
- `GlobalScope`; a coroutine not scoped to a lifecycle; a hot `Flow` collector that outlives its consumer (§7/§12)
- Blanket `withContext(IO)` around a whole method instead of the blocking call; needless dispatcher ping-pong (§4/§6)
- Swallowed `CancellationException`; a CPU loop that never checks `isActive`/`ensureActive()` (§8)
- Continuation not resumed exactly once on every path; `suspendCoroutine` where `suspendCancellableCoroutine` belongs (§9)
- Emitting from another coroutine inside a `flow { }` (context-preservation violation); `withContext` inside a flow builder instead of `flowOn` (§5/§10)
- `stateIn`/`shareIn` started `Eagerly` and never stopped (upstream leak); missing `WhileSubscribed`; oversized `SharedFlow` replay (§10)
- Plain `lifecycleScope.launch { collect }` instead of `repeatOnLifecycle`/`collectAsStateWithLifecycle` (§12)
- Hardcoded dispatchers instead of an injected `DispatcherProvider`; reasoning by thread instead of by dispatcher/confinement (§6/§2)

## Contested / judgment calls
- **UI ↔ async boundary:** keep async work in ViewModels/repositories; expose an **immutable `UiState` via `StateFlow`** and let the UI collect it synchronously (`collectAsStateWithLifecycle`). Defer the exact `state_type`/`architecture` to the project profile.
- **Room / persistence:** Room `suspend` and `Flow` DAOs are **already main-safe** (Room dispatches to its own executor) — **don't** wrap them in `withContext(IO)`; observing a Room `Flow` is handled off-main by Room. Never `runBlocking` a DAO on main, and don't pass mutable entities across coroutines assuming thread-safety — return immutable copies.
