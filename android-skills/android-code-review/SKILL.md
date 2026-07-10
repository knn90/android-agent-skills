---
name: android-code-review
description: "Adversarial, multi-lens code review for an Android project (Kotlin / Jetpack Compose / Views); routes slices of the diff to installed android-* specialist skills. Use for PR, commit, pending-diff, or whole-codebase review."
argument-hint: "[#PR | COMMIT | --pending | codebase]"
model: best
effort: xhigh
---

# Code Review — Android

Adversarial, profile-driven review for native Android Kotlin. **Precision over recall** — a focused list
of real, high-confidence findings beats a wall of nits.

**Principles:** YAGNI + KISS + DRY. Technical correctness over social comfort. **Honest, brutal, concise.**

## Step 0 — Load profile
Read `.claude/android-profile.md`: `architecture`, `state_type`, `di`, `navigation`, `networking`,
`localization`, `test_tags`, `crash_reporting`, `verify_command`, `high_rigor_domains`,
`generated_paths`, **`specialists`**, `rules_file`, `test_roots`, `plans_dir`. If missing, read the main
checkout's copy — `$(git rev-parse --path-format=absolute --git-common-dir)/../.claude/android-profile.md`. Still missing → run `android-project-init`.

---

## Phase 0 — Resolve scope (do this FIRST; everything keys off it)

Build ONE canonical `scope` and **print a banner before any finding** — if the scope is wrong, every
finding is suspect.

| Input | Mode | Diff |
|---|---|---|
| `#123` / PR URL | PR | `gh pr diff 123` (base via `gh pr view --json baseRefName`) |
| `abc1234` (7+ hex) | Commit | `git show <sha>` |
| `--pending` | Pending | `git diff` + `git diff --cached` |
| *(no args, recent edits)* | Default | files edited this session |
| `codebase` | Codebase | full scan — adapt the stages per [codebase-mode.md](codebase-mode.md) |

- **Base-branch fallback:** `gh pr view --json baseRefName` → `git rev-parse origin/HEAD` → `origin/main`|`origin/master`.
- **Buckets:** `modified` (review in full, all severities) · `tests-for-modified` (coverage → main report) · `related/context` (read for context only) · `deleted` (spec reasoning only).
- **Exclude always:** `generated_paths`, `build/` dirs, `.gradle/`, generated Hilt/KSP/KAPT output, `*.g.kt`.
- **Adjacent quarantine:** issues in context-only files go in an "Adjacent observations" section and **do NOT count** toward the verdict.
- **Banner:** `Scope: PR #123 · base: main · modified: 7 · tests: 2 · HIGH-RIGOR: yes`.
- **Trivial:** tiny diff (≤2 files, ≤30 lines, not HIGH-RIGOR) → review locally: skip the parallel fan-out and Stage 3.

**HIGH-RIGOR:** diff touches any domain listed in the profile's `high_rigor_domains` — never trivial; log `[HIGH-RIGOR]`, and alongside Stage 3 run the **money/PII correctness audit** from `android-sequential-thinking`. (The project defines its own domains via the profile; the skill assumes none.)

---

## Stage 1 — Spec compliance (skip if no plan/spec)
Read the plan (`{plans_dir}/{slug}/plan.md`) and/or the PR/issue (`gh pr view --json title,body,closingIssuesReferences`). Emit a **requirement-coverage table**:

| Requirement / AC | Status | Where |
|---|---|---|
| <criterion> | ✅ met / ⚠️ partial / ❌ missing | `file:line` |

Flag **scope creep** (unrelated refactors bundled in) and **missing work** (`TODO`, `TODO("unimplemented")`, empty bodies, stubbed mocks). No spec available → mark "not assessed" (don't invent one). **Stage 1 FAIL → stop, report, ask.**

---

## Stage 2 — Multi-lens review (run the lenses in parallel)

### 2.0 — Specialist routing (FIRST — this is the point)
**This table is the canonical signal→specialist map for the suite** (`android-execute` Phase 3 mirrors
it — update both together). For each domain the diff touches, route it to the matching specialist below
**if that specialist skill is installed** in this project — **on by default, no profile entry needed**. Spawn a **read-only
`general-purpose` Agent** that loads the specialist's `SKILL.md` and reviews **only its slice** of the
diff; the general lens (2.1) then skips that domain. The profile's `specialists:` is an **optional
override**: if set, restrict routing to that list; `specialists: none` turns routing off.

| Diff signal | Route to (if installed) | Reviews |
|---|---|---|
| `@Composable`, `remember`/`mutableStateOf`, `Modifier`, `LaunchedEffect`, recomposition/state | **android-compose-expert** | composable structure, state hoisting, recomposition perf, navigation, a11y, Material 3 |
| `suspend`, `Flow`/`StateFlow`, `viewModelScope`/`CoroutineScope`, `Dispatchers`, `launch`/`async`, `collect` | **android-coroutines-expert** | data races, dispatcher confinement, structured concurrency, cancellation, Flow correctness |
| `@Test`, MockK, Turbine, `runTest`, Espresso, `createComposeRule`, files under `test_roots` | **android-testing-expert** | JUnit / MockK / Turbine idioms, coverage, flakiness, parallelization |
| new/changed classes, interfaces, use-cases, repositories; DI / Hilt module wiring; large classes; cross-module changes | **android-solid-expert** | SOLID (SRP/OCP/LSP/ISP/DIP), decoupling (Decorator/Composite/Adapter), composition-root DI, framework isolation |
| bug-fix diff, or behavior changes landing with their tests in one diff | **android-tdd** | TDD discipline: reproduction test exists for the bug, tests exercise public behavior not internals, no test-after backfill smells |
| *(future specialists, e.g. networking/persistence)* | **android-`<domain>`-expert** | its domain |

Specialist agent prompt:
```
Load .claude/skills/<specialist>/SKILL.md and check this slice against every section of it — report
violations found, and end with one line naming the sections that came up clean, so every section is
accounted for. Review ONLY the <domain> aspects of this diff: <the relevant hunks>.
Skip what a general reviewer covers (naming, layering, localization).
Output each finding as { severity, file:line, problem, fix, confidence: high|med|low }. Be specific; cite the rule.
```
No specialists installed / none match → skip 2.0 (the general lens still covers the floor).

### 2.1 — General Android lens (always; the floor when a specialist isn't installed)
Spawn a `general-purpose` Agent (fill `{…}` from profile). **Authoritative: `rules_file` + Kotlin coding conventions + Android's Guide to app architecture** — read them.

```
Review this Android Kotlin diff with senior rigor. Flag { severity, file:line, problem, fix, confidence }.

THREADING & CONCURRENCY (highest-value bugs — unless android-coroutines-expert handled it):
- UI state updated on the main thread (Dispatchers.Main); flag blocking calls (network/DB/file) on the main thread.
- viewModelScope work is cancelled with the ViewModel; no leaked CoroutineScope / GlobalScope. Re-check state after every suspension point.
- Flows collected with lifecycle awareness (repeatOnLifecycle / flowWithLifecycle), not in a naked launch that outlives the view.
MEMORY & LIFETIME: Context/Activity leaks (long-lived refs, static holders); unregistered listeners/receivers/observers; coroutine leaks; ViewModel outliving its scope.
SECURITY & PRIVACY: token leakage; Android Keystore-backed encryption (Tink/DataStore), never plain SharedPreferences, for secrets; cleartext traffic / missing Network Security Config; PII in logs/analytics/{crash_reporting}.
MONEY (if a money/finance domain is configured in high_rigor_domains): BigDecimal not Double/Float; rounding; sign errors; currency parsing.
ACCESSIBILITY: contentDescription / TalkBack labels, semantics {} order, font scaling (unless android-compose-expert handled it).
HYGIENE: comments WHY-only; no dead/commented code; no back-compat shims for code this diff removed.

PROFILE-GATED (run only those that apply):
- state_type != none → uses {state_type}; transition holes (loading→loaded/empty/error) covered.
- di != manual → injected via {di}; no singleton reach-in from view-models.
- navigation != none → nav events emitted from the VM as one-shot values; NavController.navigate called from the UI, not the VM.
- networking != none → DTO→domain mapping at the boundary; UI doesn't import DTO / generated types.
- localization != none → no hardcoded user-facing strings; new keys added; generated resources untouched.
- test_tags != none → tested UI exposes a testTag from {test_tags}.
- Always: never edit generated_paths.
```

### 2.2 — Architecture lens (profile-gated)
Check against the project's `architecture`: **dependency direction** (View → VM → UseCase/Repo; Model independent), **single source of truth** (no `mutableStateOf` mirroring VM `StateFlow` state), **God view-model / massive reducer** (extract UseCases), **presentation isolation** (no Android framework / `Context` import in a VM; navigation as value types/events), **stale-async overwrite / missing cancellation**, business logic living in Composables/Views. Align to the *existing* pattern — don't propose an architecture switch for a small change. (When `android-solid-expert` is installed, 2.0 already covered the SOLID/decoupling depth; this lens then focuses on pattern-fit + the project's `architecture`, not re-deriving the principles.)

### 2.3 — Reuse & simplification lens
Flag both directions: **over-engineering** (duplicated logic that an existing helper covers; redundant/derived state stored; parameter sprawl; needless abstraction/indirection; stringly-typed where a sealed class/enum exists) **and over-simplification** (distinct concerns collapsed into one unclear unit). Only findings that materially improve maintainability/correctness/cost — never churn for style. (If the repo has a `simplify` skill, this lens can defer the *applying* to it; here it only flags.)

---

## Stage 3 — Adversarial red-team
Try to **break** the change across four lenses (parallel agents for large diffs). Define regression relative to the **stated intent** (what should change vs what must stay unchanged):

1. **Intent & regression** — behavior outside stated scope; broken edge/fallback paths; contract drift between callers & callees; adjacent flows that should have changed together but didn't.
2. **Security & privacy** — authn/authz gaps, unsafe input, injection, secret/token exposure, risky defaults, trust of unverified data, PII leakage.
3. **Performance & reliability** — duplicate work / redundant I/O; new work on hot paths (startup/render/request); leaks, retry storms, subscription drift; ordering/race/cancellation brittleness; bitmap/image decode on the main thread.
4. **Contracts & coverage** — API/schema/type/flag mismatches; migration & back-compat fallout; missing/weak tests for changed behavior; **detectability** (would a future regression even be observable — logs/metrics/assertions?).

Each finding: `{ severity, file:line, problem, exploit/scenario, fix, confidence }`.

---

## Synthesis (precision over recall)
Treat all lens output as raw input, then:
- **Dedup** across lenses; **drop** weak/speculative/style-only items and anything that conflicts with the stated intent — each dropped finding gets a one-line drop reason.
- "May be wrong but intent unclear" → a **question**, not a finding.
- **Normalize:** `{ file:line, category (spec|coroutines|compose|testing|architecture|security|perf|contracts|simplification|hygiene), severity, why it matters, fix, confidence }`.
- **Order:** high-severity high-confidence first. If nothing material, **say so** — don't manufacture feedback.

## Agent-loop feedback (self-improving rules)
Group findings by rule. A rule firing **≥2×** across the diff = a recurring pattern → propose a one-line **directive** for the project's `rules_file` (e.g. "Always inject the dispatcher via constructor, never hardcode `Dispatchers.IO` in view-models"). Especially useful for AI-generated diffs — it turns repeated misses into standing rules. (Propose only; don't edit `rules_file` without approval.)

## Verification gate
**Iron law: no "review passed" without fresh verification.** Tests/build → run `{verify_command}` (unset → build-only via `./gradlew assembleDebug`, say so). Bug-fix → the original reproducer no longer reproduces. Stop if you catch yourself thinking "should pass" — run it.

## Severity model
- **Critical** — crash · data race · state corruption · `BigDecimal` precision loss in money (Double/Float for money) · credential leak · PII in logs → **must fix before merge** (verdict cannot be APPROVED while one remains).
- **Important** — state-transition hole · layer/dependency violation · memory leak · missing test for changed behavior · missing testTag on tested UI · missing localization key → **fix before merge or file a ticket**.
- **Nit** — style, naming, API cleanliness → author's call.

## Report format
```markdown
Scope: <banner>
# Code Review — <mode>   HIGH-RIGOR: yes/no   Verdict: APPROVED / CHANGES REQUESTED / BLOCKED

## Summary   Critical: N · Important: N · Nit: N
## Stage 1 — Spec       <coverage table> · scope-creep: <…>
## Findings (by severity)   [Sev] category — file:line — problem → fix (confidence)
   - Specialist findings grouped under their sub-heading (android-compose-expert / …)
## Agent-loop feedback   <recurring pattern → proposed rules_file directive>  (or none)
## Adjacent observations (out of scope)
## Critical open items · Recommended follow-ups
```

## Constraints
- **Read-only review** — find and recommend; do NOT apply fixes (that's `android-execute` / a `simplify` skill).
- **DO NOT** rely on memory — read the actual diff + `rules_file`. Specialists win on their domain.
