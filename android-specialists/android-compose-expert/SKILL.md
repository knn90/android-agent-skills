---
name: android-compose-expert
description: "Expert Jetpack Compose guidance — state, recomposition/performance, layout, navigation, animation, accessibility, Material 3. Fires when work touches Compose (@Composable, remember/mutableStateOf, Modifier, LazyColumn, recomposition)."
---

# Compose Expert

> **Generated skill** — original wording, consolidated by `android-skill-consolidate`; provenance,
> sources, and licenses live in `SOURCES.yaml`. Don't hand-edit — change `SOURCES.yaml` and re-consolidate.

**Source of truth:** Google's official Jetpack Compose documentation / Android Developers guides / Material 3
guidelines are the **primary authority** (see `SOURCES.yaml`) and win on any conflict, gated to the profile's
`min_sdk`. The community sources below fill in detail.
**Architecture:** follow the project's `architecture` from `.claude/android-profile.md`; where the project
is unopinionated, Compose's native **state hoisting + unidirectional data flow** (state down, events up — no
reflexive ViewModel per composable) is the default. Don't impose MVVM/MVI where the project doesn't ask for it.
Profile: `.claude/android-profile.md` — worktrees don't inherit it; fall back to
`$(git rev-parse --path-format=absolute --git-common-dir)/../.claude/android-profile.md`.

---

## 1. State & data flow
- **Baseline stack:** transient UI state is `mutableStateOf(...)` held in `remember { }`; hoist it to the lowest caller that needs it (state down, events up). Screen state comes from a ViewModel as a `StateFlow`, read with `collectAsStateWithLifecycle()` — prefer it over `collectAsState()`, which keeps collecting while the UI is STOPPED and wastes work / leaks updates.
- `remember` survives recomposition but **not** configuration change or process death; use `rememberSaveable` for state that must survive rotation/process-death (value must be `Bundle`-saveable / `@Parcelize`, or supply a custom `Saver`).
- **Single source of truth:** a value lives in exactly one place; derive everything else. Never mirror a `StateFlow`/hoisted value into a second local `mutableStateOf` — the copy drifts (silent bug). Update state only through the owner's event callbacks.
- **Never mutate state during composition** (`state.value = …` in the composable body) — it schedules another recomposition and can loop. Mutate inside event lambdas or effects (`LaunchedEffect`/`SideEffect`), never inline in `body`.
- Don't **hoist too high**: hoist to the lowest common ancestor of a state's readers and writers. Over-hoisting couples unrelated composables and widens the recomposition scope; under-hoisting blocks reuse/testing.
- Use `derivedStateOf` for state **computed from other state that changes more often than the result** (e.g. `isScrolled` from raw scroll offset) so readers recompose only when the derived value flips. Wrap it: `remember { derivedStateOf { … } }`.
- Mark state/model classes handed to composables `@Immutable` / `@Stable` so the compiler can skip; unstable params (raw `List`/`Map` interfaces, types from other modules, lambdas capturing unstable values) force recomposition even when nothing visible changed.

## 2. Composition & reuse
- Extract sections into small, focused `@Composable` functions — one responsibility each. Small composables recompose independently and keep the recomposition scope tight.
- **Slot APIs** for containers: take `content: @Composable () -> Unit` (or named typed slots like `header:`/`content:`/`actions:`) instead of hard-coding children — the standard Compose reuse pattern (`Scaffold`, `topBar`, etc.).
- **Stateless vs stateful:** a stateful composable owns its `remember`ed state; extract a stateless overload that takes `value` + event lambdas. Preview/test/reuse target the stateless one; the stateful wrapper just wires state.
- **Don't pass whole ViewModels down** the tree — hoist state at the top and pass the specific values plus event lambdas (`onClick: () -> Unit`). Deep children stay previewable and unit-testable and don't depend on the VM type.
- Every reusable composable takes `modifier: Modifier = Modifier` as its **first optional parameter** and applies it to its root element — omitting it robs the caller of layout/size/padding control.
- One public composable per file, named after the file; keep helper composables `private`. Split files and flag a composable body longer than roughly one screen.
- Keep composable bodies **pure and cheap**: no I/O, no heavy compute, no allocation, no inline `filter`/`sortedBy`/formatter creation — move that to `remember`, the ViewModel, or helpers so `body` "reads like UI".
- Design for **preview-ability** from the start: a composable that needs live DI/network to render can't be previewed — inject data as parameters (see §10).

## 3. Layout
- Compose layout is **declarative**: `Row`/`Column`/`Box` with `Modifier.weight`, `Arrangement`, and `Alignment`. Don't read `LocalConfiguration`/screen size to make layout decisions you can express with modifiers and constraints — views must work in any container (sheet, dialog, split-pane, embedded).
- Avoid `Modifier.onGloballyPositioned` / `onSizeChanged` to *drive* layout you could do declaratively — they fire **after** layout and force extra passes / recomposition loops; reserve them for genuinely reporting a measured size upward.
- `BoxWithConstraints` is **last-resort** (it forces subcomposition and defers content) — reach for it only when child layout truly depends on the measured `maxWidth`/`maxHeight`, not for things `weight`/`fillMaxWidth` already do.
- One full-width child → `Modifier.fillMaxWidth()` (+ `Alignment`), not `Row { child; Spacer(Modifier.weight(1f)) }`.
- Give sizes **flexibility**; avoid fixed `dp` that breaks across screen sizes and font scaling. Don't cap text in a hardcoded-height box — let it wrap/`wrapContentHeight`. Use `sp` for text, `dp` for layout dimensions.
- Use **intrinsic measurements** (`Modifier.height(IntrinsicSize.Min)`) to align siblings by content instead of magic numbers; consume system bars with `WindowInsets`/`windowInsetsPadding`, never a hardcoded status-bar height.

## 4. Performance
- **Recomposition scope** = the smallest enclosing restartable composable that *read* the changed state. Read state as low/late as possible (in the child that uses it, or via a lambda) so a change restarts a small subtree, not the whole screen.
- **Stability drives skipping:** mark data models `@Immutable`/`@Stable`; an unstable parameter (unstable class, raw `List`/`Map`, cross-module type) makes the composable non-skippable. Prefer `ImmutableList`/`persistentListOf` (kotlinx.collections.immutable) over raw `List` for params.
- `remember` **expensive calculations** — formatters, sorted/filtered lists, derived objects — keyed on their inputs (`remember(query) { list.filter { … } }`). Never recompute them, and never allocate, in a composition hot path.
- **Stable keys** in lazy lists: `items(list, key = { it.id })` with real domain IDs. **Never use the index** as the key for dynamic/reorderable content — it breaks item animations and per-item state on insert/remove and can misassociate state.
- **Defer state reads** with lambda-based modifiers: `Modifier.offset { IntOffset(x, y) }`, `graphicsLayer { }`, `drawBehind { }` read their value at the layout/draw phase and **skip recomposition** — use them for scroll-, gesture-, and animation-driven values instead of the value-based `Modifier.offset(x.dp)`.
- Avoid **unstable lambdas** in hot paths: don't allocate a new capturing lambda every recomposition; hoist stable callbacks / method references, and `remember` a lambda when it must capture changing values.
- `derivedStateOf` to convert a high-frequency source (scroll offset) into a low-frequency output (a boolean) so downstream recomposes only on the threshold flip — don't store raw continuous scroll offset in `mutableStateOf` that UI reads directly.
- **Measure, don't guess:** Layout Inspector recomposition counts + the Compose compiler metrics/reports (stability, skippability) find the real offenders; a composition trace confirms.

## 5. Navigation
- **Navigation-Compose:** one `NavHost` per graph with a single `NavController`; define **typed routes** (type-safe `@Serializable` route classes, or `composable(route)` + `navArgument` with declared types). Keep exactly one NavHost per nav graph.
- **Hoist navigation out of composables:** pass `onNavigateToX: () -> Unit` lambdas down; **never pass the `NavController` deep** into the tree — it couples children to navigation and kills preview/test. The `NavController` lives at the NavHost call site.
- Route args carry **lightweight, serializable** data (ids/primitives), never heavy objects/`Parcelable` blobs of state — fetch by id in the destination's ViewModel. The back stack owns state; each destination gets a `ViewModel` scoped to its `NavBackStackEntry`. Reset the back stack on logout/account change.
- **Deep links** via `deepLinks = listOf(navDeepLink { uriPattern = … })`. Present dialogs/bottom sheets as destinations (`dialog { }` / bottom-sheet nav) or drive them from hoisted state — not ad-hoc `if (showSheet)` scattered across the tree.

## 6. Animation
- Pick the smallest tool: `animate*AsState` (`animateDpAsState`/`animateColorAsState`/`animateFloatAsState`) for a single fire-and-forget value; `updateTransition` for several values driven by one state; `AnimatedVisibility` for enter/exit; `rememberInfiniteTransition` for looping.
- Target-based animations are **gated to value changes** by construction — don't drive them from a manually-ticked frame loop. Use `Animatable` for imperative/gesture-coordinated animation with `snapTo`/`animateTo` and proper cancellation.
- Prefer **transform** animations (`graphicsLayer`/`scale`/`offset`/`rotate`) over animating layout modifiers (`padding`/`size`/`Arrangement`) — transforms skip re-measure/re-layout. Scope the animation to the smallest subtree; never animate every scroll frame through recomposition.
- `AnimatedContent`/`Crossfade` for swapping content. Drive animation from `remember`ed animation objects/state, **not** `delay()`-then-set. Specs: `spring()` for interactive/natural motion, `tween(easing = FastOutSlowInEasing)` for appearance, `LinearEasing` only for progress/continuous.

## 7. Accessibility
- Every meaningful non-text element needs a `contentDescription` (`Image`/`Icon`); **decorative** elements pass `contentDescription = null` so TalkBack skips them. Use real clickable components (`Modifier.clickable`, `Button`) so they get role/focus/traits for free — don't reinvent taps with raw `pointerInput`.
- `Modifier.semantics { }` to add/override semantics (`stateDescription`, `role = Role.Button`, `heading()`); `Modifier.semantics(mergeDescendants = true) { }` (or `clearAndSetSemantics { }`) to collapse a composite control into a single TalkBack focus target.
- **Touch targets ≥ 48dp** (`Modifier.minimumInteractiveComponentSize()` or `sizeIn(minWidth = 48.dp, minHeight = 48.dp)`); respect **font scaling** — size text in `sp` and don't trap it in a fixed-height container. Verify with TalkBack **and** a large font-scale.
- Add `Modifier.testTag("…")` for UI tests (expose the tags from the profile's `test_tags`) — tags are semantics-only and don't affect TalkBack. Add accessibility semantics once the UI is interactive, not as an afterthought.

## 8. Coroutines in Compose
- Launch effects from effect handlers, **never from the composition body**: `LaunchedEffect(key)` for a coroutine tied to composition (re-launches on key change, auto-cancels on leave), `rememberCoroutineScope()` to launch from a callback, `produceState`/`snapshotFlow` to bridge a suspend fn or `Flow` into Compose state; collect flows with `collectAsStateWithLifecycle()`. Treat `kotlinx.coroutines.CancellationException` as **normal** — never surface it as a user error, and always rethrow it. Deep coroutine work (Dispatchers, structured concurrency, `Flow` operators, thread-confinement, cancellation internals) → `android-coroutines-expert`, routed by the caller.

## 9. Modern API / always-never (currency)
- **Material 3** (`androidx.compose.material3`) over Material 2. **`Modifier` order is semantic** — `.padding().background()` differs from `.background().padding()`, and `.clickable()` before vs after `.padding()` changes the ripple bounds; order deliberately. **Edge-to-edge** (`enableEdgeToEdge()`) + consume insets via `Scaffold` `contentPadding` / `Modifier.windowInsetsPadding` / `WindowInsets`, never hardcode status/nav-bar heights. `LazyColumn`/`LazyRow`/`LazyVerticalGrid` for large or unbounded lists, **not** `Column`/`Row` + `verticalScroll` (which composes every child up front). Use `stringResource()`/`dimensionResource()`/`painterResource()` over hardcoded literals. Check deprecations before reaching for an older component.

## 10. Testing & previews
- `@Preview` for each meaningful state (default/empty/error/loading); `@PreviewParameter` with a `PreviewParameterProvider` to sweep states from one function; add **dark-mode** (`uiMode = UI_MODE_NIGHT_YES`), `fontScale`, and `widthDp` previews to catch large-font and size breakage. Previews are **self-contained** — pass fake/sample data directly, no live network/DB/DI; preview the **stateless** composable.
- Put core logic in the **ViewModel/domain layer** so it's unit-testable (JUnit + Turbine for `StateFlow`); reach for Compose UI tests (`createComposeRule`, `onNodeWithTag`) only when a unit test can't cover it.

## 11. Material 3 / Material You
- **Adopt dynamic color / expressive components only when asked** — never proactively restyle existing UI. Dynamic color (`dynamicLightColorScheme`/`dynamicDarkColorScheme` from wallpaper) is **Android 12+** — gate with `if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S)` and fall back to a static brand `ColorScheme`.
- Theme through `MaterialTheme` — read `MaterialTheme.colorScheme` / `.typography` / `.shapes`; **never hardcode `Color(0xFF…)` or dp text sizes** inside components. Support light/dark by supplying both schemes and honoring the system setting.
- Material 3 **Expressive** components/motion are newer and opt-in — use them only when asked, confirm the artifact is present in the project's Compose BOM, and gate experimental APIs (`@OptIn(ExperimentalMaterial3…Api::class)`) deliberately rather than blanket-suppressing.

## Review completion criterion
A critique pass is done only when the always/never rules in §§1–7 and §§9–11 have each been
**ruled in or out** against the diff — not when the obvious findings run dry.

## Contested / judgment calls
- **Immutable collections vs strong skipping:** community guidance says "always wrap `List` params in `ImmutableList` for stability" — correct for skippability, but it adds a dependency and is partly moot under **strong-skipping mode** (default with the Compose compiler on Kotlin 2.0+), which skips composables with unstable params by instance equality. Check whether the project has strong skipping enabled before mandating immutable collections everywhere.
- **`derivedStateOf` is frequently over-applied** — it earns its keep only when the input changes **more often** than the output (scroll offset → boolean). For a cheap 1:1 derivation it adds overhead over a plain `remember(input) { … }`; don't reach for it reflexively.
