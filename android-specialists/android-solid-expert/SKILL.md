---
name: android-solid-expert
description: "Expert SOLID + clean-decoupling review for Android/Kotlin — the 5 principles by name (SRP, OCP, LSP, ISP, DIP), composition-root DI (Hilt/Koin/manual), decoupling patterns (Decorator / Composite / Adapter / Facade), and framework isolation. Fires when a change adds/alters classes, interfaces, repositories, use-cases, view-models, or DI/module wiring. Delegates *which architecture pattern* to the architecture lens."
---

# SOLID / Decoupling Expert (Android / Kotlin)

> **Generated skill** — original wording, consolidated by `android-skill-consolidate`; provenance,
> sources, and licenses live in `SOURCES.yaml`. Don't hand-edit — change `SOURCES.yaml` and re-consolidate.

**Source of truth:** SOLID is **general** software engineering (Robert C. Martin). Google's official
**Guide to app architecture** (`developer.android.com`) is the Android-idiomatic frame — UI → domain
(optional use-cases) → data, unidirectional state, repositories as the source of truth. The Kotlin
translation leans on **small interfaces + composition** over inheritance. Gate to the project's
`architecture` + conventions in `rules_file`.
Profile: `.claude/android-profile.md` — worktrees don't inherit it; fall back to
`$(git rev-parse --path-format=absolute --git-common-dir)/../.claude/android-profile.md`.

**North star (YAGNI):** SOLID serves *change*, not ceremony. An abstraction earns its keep with
≥2 implementations **or** a real test seam; otherwise it's **speculative generality**.

---

## The five principles (rule · Kotlin idiom · smell → fix)

**1. SRP — Single Responsibility.** One reason to change / one actor a type answers to. *Smell:* a
view-model that fetches **and** parses **and** caches **and** formats; a `Manager`/`Helper`/`Util` doing
everything. *Fix:* one use-case/repository per responsibility; the view-model only orchestrates state.

**2. OCP — Open/Closed.** Add behavior by adding a type, not by editing existing code. *Kotlin:* an
interface + a new implementation / injected strategy; an extension function or interface default method.
*Smell:* a `when (kind) { … }` you must edit for every new case. *Fix:* polymorphism / strategy / a
`sealed interface` + exhaustive `when` where the closed set is genuinely fixed.

**3. LSP — Liskov Substitution.** An implementation must be usable everywhere the abstraction is, with no
surprises. *Kotlin:* prefer interface composition over open-class inheritance. *Smell:* an implementation
that `throw`s `NotImplementedError`/`TODO()` or no-ops part of the interface ("unsupported"). *Fix:* split
the interface (→ ISP).

**4. ISP — Interface Segregation.** Many small, client-focused interfaces over one fat one. *Kotlin:*
`FeedLoader`, `ImageCache` — depend on just the one a caller needs. *Smell:* a god interface with 15
methods; implementations stubbing half of them. *Fix:* split by what each client actually uses.

**5. DIP — Dependency Inversion.** Policy depends on **abstractions it owns**, not on concrete
details. *Kotlin:* the **domain declares** `interface FeedRepository`; the data module *implements* it;
the Hilt/Koin module (composition root) binds them. *Smell:* a view-model that `import`s Retrofit/Room or
constructs a concrete client. *Fix:* invert — depend on a domain interface, inject the impl.

## Dependency injection & the composition root
- **Constructor injection.** The **composition root** — Hilt `@Module`s / Koin `module {}` / a manual
  `AppContainer` — is the *only* place that knows concrete types and assembles the graph. Everything else
  takes abstractions (constructor `@Inject` / `by inject()`).
- *Smells:* service locators / `object` singletons reached from policy code; `Context`/`Application`
  reached statically; **constructor over-injection** (≥4–5 deps ⇒ the type does too much → split it, or
  hide a subsystem behind a Facade).
- Constructor injection + a `@Binds`/`bind()` to an interface gives testability without ceremony (no live
  `Retrofit`, `SharedPreferences`, `System.currentTimeMillis()`, real `Dispatchers` reached from inside).

## Decoupling patterns (the Essential-Developer toolkit)
- **Decorator** — add a cross-cutting concern (logging, analytics, main-thread dispatch, caching) with
  the *same* interface in and out, without touching the decorated type.
- **Composite** — combine implementations behind one interface (e.g. remote-**with-fallback-to**-cache `FeedRepository`).
- **Adapter** — bridge a concrete/SDK type (Retrofit service, Room DAO) to the domain's interface, keeping the SDK at the edge.
- **Facade** — hide a multi-step subsystem behind a simple interface for the composition root.

## Framework isolation
- The **domain is pure Kotlin** — no `import android.*`/Compose/Retrofit/Room in domain types. The
  domain/use-case layer must not know about `Context`, `Cursor`, Retrofit responses, or `@Entity` rows.
  Frameworks are *replaceable details* behind interfaces the domain owns; map DTO/Entity → domain model at
  the boundary. *Smell:* `import android.content.Context` in a use-case; a Room `@Entity` or a Retrofit/
  `@Serializable` DTO leaking into the domain/UI. (Pairs with the testing skill's mockability + the
  coroutines skill's dispatcher-injection rules.)

## Composition over inheritance (the Kotlin translation)
- Small interfaces, `data class`/`value class` models, delegation (`by`), interface default methods and
  extension functions for shared behavior, `sealed interface` for closed hierarchies — this is how SOLID
  lands idiomatically in Kotlin. Keep types `final` (the default); open for inheritance only deliberately.

## Review checklist (what this skill flags)
Per changed/added type: single responsibility? depends on abstractions or concretes? a framework
leaking into the domain? interface the right size (ISP) / honestly substitutable (LSP)? extended vs
modified (OCP)? wired in a DI module at the composition root, not reached as a singleton/`Context`? Output
each finding as `{ severity, file:line, principle, problem, fix }`. Severity: **Critical** (framework leak
into domain, concrete dependency that blocks testing a high-value path) · **Important** (God type, fat
interface, over-injection, OCP when-edit) · **Nit** (naming, minor seam). Don't invent abstraction needs —
flag *speculative* abstractions too.

## Boundaries (avoid duplication)
- **Pattern selection** (MVVM vs MVI vs Clean-MultiModule vs MVVM+UseCase) → the architecture lens, not here.
- **Coroutine isolation / dispatcher confinement** → `android-coroutines-expert`. **Compose state ownership** →
  `android-compose-expert`. **Test seams / doubles** → `android-testing-expert`. This skill is the
  principle-level "is it decoupled and substitutable?" lens that sits beneath all of them.
