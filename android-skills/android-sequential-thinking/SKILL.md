---
name: android-sequential-thinking
description: "Step-by-step analysis with revision/branching for complex Android problems — multi-layer data-flow bugs, cache/normalisation issues, navigation routing tangles, state-transition holes, Kotlin Coroutines/Flow races, money/PII correctness audits. Internal reasoning aid, not a planner."
argument-hint: "[problem to analyse step-by-step]"
---

# Sequential Thinking — Android

## Step 0 — Load profile (light)
Skim `.claude/android-profile.md` for `architecture`, `state_type`, `navigation`,
`networking`, `high_rigor_domains` so the patterns below map onto this app's vocabulary.
If missing, skim the main checkout's copy —
`$(git rev-parse --path-format=absolute --git-common-dir)/../.claude/android-profile.md`
(the profile is usually gitignored, so worktrees don't inherit it).

## When to Apply
At least one of:
- **Multi-step data flow** — Repository → … → Composable; trace where state diverges.
- **Cache / normalisation** (if `networking` uses a cache) — query returns right data, UI shows stale.
- **Navigation routing** — navigate/popBackStack/dialog ordering, retained references, race after `suspend`.
- **State transitions** (`state_type`) — loading→loaded→error edges; empty vs error vs initial; pagination handoff.
- **Kotlin Coroutines / Flow** — `Job.cancel()` propagation, main-thread violations, thread-confinement, shared-state reentrancy.
- **Hypothesis-driven debugging** — symptom is N layers from cause.
- **`high_rigor_domains` correctness audits** — wrong logic ships real-money / PII bugs.

## When NOT to Use
| Case | Use instead |
|---|---|
| Simple one-step answer | Just answer |
| Brutal trade-off comparison | `android-brainstorm` |
| Plan + phases | `android-plan` |
| Bug needs log/CI investigation | Direct `Bash`/`Read` |

---

## Core Process

1. **Loose estimate** — `Thought 1/N: [framing]`.
2. **Structure each thought** — build on prior context, one aspect each, state assumptions/
   uncertainties, signal what the next thought addresses.
3. **Adjust N as understanding changes** — expand or contract.
4. **Revision**
   ```
   Thought 5/8 [REVISION of Thought 2]: <corrected understanding>
   - Original: … - Why revised: … - Impact: …
   ```
5. **Branching**
   ```
   Thought 4/7 [BRANCH A from 2]: <approach A>
   Thought 4/7 [BRANCH B from 2]: <approach B>
   ```
   Compare, converge with rationale.
6. **Hypothesis & verification**
   ```
   Thought 6/9 [HYPOTHESIS]: <cause/solution>
   Thought 7/9 [VERIFICATION]: <checked file:line — found …>
   ```
   Every [HYPOTHESIS] must be paired with a [VERIFICATION] carrying a receipt — a real
   `path:line`, not abstract reasoning.
7. **Finish** — `Thought N/N [FINAL]` names the root cause (`path:line`), the fix, and the
   test gap that would have caught it (or the pattern's own FINAL shape). Stop there —
   don't pad to N.

---

## Reusable Patterns (adapt names to `state_type`/`navigation`/`networking`)

### State-transition audit
```
1: Define expected sequence (initial → loading → loaded/empty/error)
2: Locate the state holder (path:line)
3: Trace each emit point — does each branch exit cleanly?
4: Pagination — cursor handoff between pages
5 [HYPOTHESIS]: skipped state / lost cursor / double-emit
6 [VERIFICATION]: read tests — is this edge covered?
N [FINAL]: root cause + fix + test gap
```

### Cache / normalisation debug  (only if networking caches)
```
1: Identify query/fragment + cache key policy
2: What writes that key? (mutation, manual store update)
3: Observers — collection scope tied to which lifecycle owner?
4 [HYPOTHESIS]: stale entity after partial update
5 [VERIFICATION]: read generated query + key fn (never edit generated_paths)
N [FINAL]: cache fix + invalidation strategy
```

### Coroutines / Flow race / cancellation
```
1: Identify coroutine boundaries (launched / awaited / cancelled — which scope/Job)
2: Dispatchers.Main vs IO/Default — any UI write off main? any blocking on Main?
3: Cancellation propagation — does inner suspend honour isActive / ensureActive()?
4: Reentrancy — re-collectable while a previous suspend is in-flight? shared mutable state guarded (Mutex)?
5 [HYPOTHESIS]: e.g. cursor advanced while previous fetch in-flight
6 [VERIFICATION]: read the ViewModel + repository; confirm guard
N [FINAL]: race surface + guard/cancellation fix
```

### Navigation routing knot
```
1: Diagram screens + desired transitions
2: Locate the NavController/Navigator — what callbacks pass to children?
3: When captured? holding Context/View/lifecycle beyond scope? leak?
4: Presentation order — dialog over bottom sheet? navigate during pop/transition?
5 [HYPOTHESIS]: missed popBackStack / transition timing / back-stack state mismatch
6 [VERIFICATION]: trace one path end-to-end
N [FINAL]: routing fix + invariant a test must protect
```

### Money / PII correctness audit  (high_rigor_domains)
```
1: What value flows here? (BigDecimal? Double? string from backend?)
2: Precision boundaries — where formatting/rounding happens
3: Sign / direction — refund vs charge, credit vs debit
4: Edge cases — zero, negative, very large, multi-currency
5: PII — what's logged? crosses analytics / crash reporting?
6 [HYPOTHESIS]: precision loss / leaked PII / wrong sign
7 [VERIFICATION]: run the math on paper + grep log emissions on the value
N [FINAL]: pass/fail per case + remediation
```

## Constraints
- **DO NOT** implement during thinking — just reason.
- **Collapse the chain to conclusion + key reasoning** — surface the full trace only when
  the user asks for it or `high_rigor_domains` correctness reasoning needs an audit trail.
