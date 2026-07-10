---
name: android-grill
description: "Grill an Android implementation plan — stress-test it one decision at a time, each question carrying a recommended answer, codebase checked first. Auto-run by android-resolve between plan and approval; standalone on a plan dir or ticket id. Does NOT implement."
argument-hint: "[plan-dir | TICKET-ID]"
model: best
effort: high
---

# Grill — Android plan stress-test

**Interrogate the human** until every hidden assumption in the plan is either answered by the
code or decided on the record. Output: a **hardened plan** + a decisions log.

## Step 0 — Load profile

Read `.claude/android-profile.md`: `architecture`, `state_type`, `di`, `navigation`, `networking`,
`localization`, `test_tags`, `feature_flags`, `verify_command`, `high_rigor_domains`,
`plans_dir`, `rules_file`, ticket fields. If missing, read the main checkout's copy —
`$(git rev-parse --path-format=absolute --git-common-dir)/../.claude/android-profile.md`. Still missing →
run `android-project-init`.

## What this is (and is NOT)

| Is | Is NOT |
|---|---|
| Interactive — **you** decide; the skill proposes a recommended answer per question | An autonomous reviewer that decides for you → that's `android-plan red-team` |
| Excavates assumptions a plan left implicit, resolves them on the record | Trade-off ideation from scratch → `android-brainstorm` |
| Edits an **existing** plan to remove ambiguity | Writing the plan → `android-plan` · finding code → `android-scout` |

Pairs with `android-plan red-team`: red-team **finds** holes hostilely (no user); grill **closes** them
with you. Run red-team first on a hard plan, then grill to resolve what it surfaced.

## When to Use / NOT

| Use | Skip |
|---|---|
| Plan has Open Questions, likely-unmitigated Risks, or fuzzy ACs | Trivial single-file plan, ACs concrete, no open questions |
| `high_rigor_domains` touched (security/rollback/flag decisions) | Every decision is already answered by code / ticket / profile |
| State shape (success/error/empty/loading) or flag rollout undecided | You explicitly passed `--no-grill` / `--skip-approval` upstream |

**Auto-skip is a feature.** If, after Step 2, every open decision resolves to ≥ ~95% confidence
from the code/ticket/profile, do **not** ask anything — print one line ("Plan is unambiguous —
nothing to grill") and hand straight back. Never invent questions to justify the step — but skip
on the ≥95% evidence, not on looks: complete-looking plans hide the costliest unstated assumptions.

---

## Argument parsing
- **plan-dir** — path to a plan directory under `{plans_dir}` (contains `plan.md`). Default: the
  plan `android-resolve` just produced, else the most recent dir in `{plans_dir}`.
- **TICKET-ID** (matches `ticket_pattern`) → find the matching plan dir under `{plans_dir}`.
- If no plan exists → say so; point at `android-plan`. Grill never plans from nothing.

---

## Flow
```
0. Load profile
1. LOAD        plan.md + per-phase files + scope.md + rules_file + ticket
2. BUILD       extract candidate decisions → answer from code FIRST → confidence-gate (auto-skip if all ≥95%)
3. ORDER       dependency order (blockers first) + impact (high-impact first)
4. GRILL       one question at a time, each with a recommended answer; update state after each
5. WRITE       fold decisions back into plan.md + per-phase files; write grill.md log
6. HANDOFF     back to the approval gate (or standalone menu)
```

### Step 1 — Load
Read the plan dir: `plan.md` (Overview, ACs, Approach, Phases, **Open Questions**,
**Risks & Mitigations**, **Out of Scope**, Feature Flag, Testing Strategy) + per-phase files.
Reuse upstream context (DRY): `scope.md` (the scout map) if present, `rules_file`, and the
fetched ticket. Don't re-scout — read what `android-resolve`/`android-plan` already gathered.

### Step 2 — Build the decision list (check the code FIRST)
Harvest candidate decisions from the plan, each a node in a small decision tree:
- **Open Questions** verbatim.
- **Risks** whose mitigation is "TBD"/likely/unowned.
- **Out of Scope** — confirm each boundary is intended, not an oversight.
- **State shape** (`state_type`): are success / error / **empty** / **loading** all specified?
- **Feature flag** (`feature_flags` != none): name, default-off, rollout/kill-switch decided?
- **`high_rigor_domains` touch** → security surface, rollback safety, analytics, migration.
- **Ambiguous ACs / phases** — any phase whose "done" you can't test objectively.
- **Cross-module ripple** — DI lifecycle, navigation ownership, cache invalidation left implicit.

> **For every candidate, try to answer it from code / `scope.md` / ticket / profile / `rules_file`
> before adding it to the ask-list.** Anything the repo already answers is dropped (cite where) —
> asking it burns trust. This is the engine of the auto-skip.

Assign each surviving decision a **0–100% confidence** (your best read). All ≥ ~95% → auto-skip
(see above). Otherwise keep the < ~95% set — that's the grill list.

### Step 3 — Order
Sort the grill list by **dependency** (a decision that constrains others is asked first — e.g.
"is this behind a flag?" before "what's the flag's default rollout?") then by **impact**
(what most changes the implementation). Re-evaluate after each answer: a later question may be
fully resolved by an earlier one — drop it rather than ask.

### Step 4 — Grill loop (one at a time)
Use `AskUserQuestion`, **one focused decision per turn** (bundle only tightly-coupled
sub-parts). Every question MUST carry a **recommended answer as the first option, labeled
`(Recommended)`**, with 2–4 real alternatives. State the recommendation's *why* in one line.

```
Decision N/<total>: <the decision, concretely>
Code says: <what scope.md/the code already constrains, or "nothing — undecided">
Recommend: <your pick> — <1-line rationale>
[ options via AskUserQuestion: (Recommended) … · alt · alt ]
```

After each answer:
- **Listen for "should-want" signals** — buzzwords ("scalable", "robust") or "I should probably…"
  offered without a concrete need. Push back once: "what breaks if we *don't*?" Decide on the
  real need, not the best-practice reflex.
- Update working state; re-check the remaining list (Step 3) and drop now-answered decisions.

**Stop condition:** when you can confidently predict the user's answer to the next three
questions you'd ask, you're done — don't drain the list for completeness.

### Step 5 — Write back (only what changed)
Fold each decision into the plan **surgically** — touch only what the decisions changed:
- Resolve **Open Questions** (move to a decided line / delete) and sharpen affected **ACs**.
- Update **Risks**, **Out of Scope**, **Feature Flag**, **Testing Strategy**, and any phase
  whose steps/checkpoint the decision altered.
- Don't rewrite the plan or re-slice phases — that's `android-plan`'s job. If grilling reveals the
  plan is structurally wrong, **stop** and recommend re-running `android-plan` with the findings.

Write the decisions log: `{plan-dir}/grill.md`
```markdown
# Grill: <plan title>
**Plan**: {plan-dir}/plan.md   **Ticket**: <id|n/a>   **Date**: <YYYY-MM-DD>   **Branch**: <current>
## Decisions
| # | Decision | Recommended | Chosen | Plan change |
|---|---|---|---|---|
## Answered by the codebase (not asked)
- <decision> — <file:line / scope.md / ticket that settled it>
## Assumptions surfaced
- <implicit assumption the plan was making, now explicit>
## Still open (owner)
- <decision deferred to PM / BE / Design — who, and why it can wait>
```
(`<YYYY-MM-DD>` / any timestamp from `bash -c 'date +%F'`.)

### Step 6 — Handoff
- **Driven by `android-resolve`:** return control; resolve runs its approval gate on the hardened plan.
- **Standalone:** print the menu (do NOT auto-invoke):
```
Plan hardened → {plan-dir}/plan.md   (decisions: {plan-dir}/grill.md)
- Implement now            → android-execute {plan-dir}/plan.md
- Hostile review first     → android-plan red-team {plan-dir}
- Re-plan (structural gaps) → android-plan <ticket-or-topic>
```

---

Provenance: `SOURCES.yaml` (audit-only — `android-skill-consolidate` flags staleness; adaptation is
hand-curated, never auto-merged).
