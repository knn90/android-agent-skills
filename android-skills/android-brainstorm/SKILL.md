---
name: android-brainstorm
description: "Brainstorm solutions for an Android app. Use for ideation, architecture decisions, or feasibility checks. Does NOT implement — hands off to android-plan."
argument-hint: "[topic | TICKET-ID | description]"
model: best
effort: xhigh
---

# Brainstorm — Android

Collaborate to find the best solution while being **brutally honest** about feasibility,
trade-offs, and over-engineering.

**YAGNI + KISS + DRY.** Prefer the boring, proven approach already used in the codebase.

## Step 0 — Load profile

Read `.claude/android-profile.md` — keys are referenced at point of use below.
If missing, read the main checkout's copy —
`$(git rev-parse --path-format=absolute --git-common-dir)/../.claude/android-profile.md`. Still missing →
run `android-project-init`.

<HARD-GATE>
Brainstorm and advise only. Do NOT invoke any implementation skill, write code, write
plans, or scaffold anything until you have presented a design AND the user has approved
it — regardless of perceived simplicity. If you think you already know the answer,
writing it down takes 30 seconds.
</HARD-GATE>

---

## Phase 1 — Context
- Parse args: matches `ticket_pattern` → fetch via `ticket_fetch`; extract title,
  description, ACs, links. Otherwise treat as a free-form topic.
- Load `rules_file` (honor it throughout), relevant `docs_root` files, branch name +
  `git log -5 --oneline`.
- Scout for non-trivial topics (`android-scout` if scope spans 3+ modules). **Never recommend
  something whose existence you haven't verified.**

## Phase 2 — Clarifying questions
`AskUserQuestion`, bundle 2-4. Typical: feature flag needed (`feature_flags`)? deadline?
`min_sdk` floor? backend/schema change? design sign-off? localization keys? success/error/
empty/loading states (`state_type` shape)? testTags needed for UI tests?
Skip anything already answered by the ticket or scout.

## Phase 3 — Scope assessment
| Signal | Action |
|---|---|
| Spans 3+ independent subsystems | **Decompose** — split into sub-topics |
| 1-line copy/colour change | **Skip brainstorm** — point at implementation |
| Touches `high_rigor_domains` | **HIGH-RIGOR** — extra scrutiny, security Qs, rollback discussion |
| Greenfield, no prior art | First feature **establishes** the pattern — relax DRY, focus on getting the foundation right |

## Phase 4 — Propose approaches (2-3)

```markdown
### Approach N: <name>
**Summary**: <1-2 sentences>
**How it works here**:
- UI ({ui_framework}):
- State ({state_type}):
- Navigation ({navigation}):
- Dependencies / DI ({di}):
- Networking ({networking}, if any):
- Localization + testTags touched:
- Tests (unit / screenshot / UI):
**Pros / Cons** · **Effort** S/M/L · **Risk** LOW/MED/HIGH · **Maintainability**
**Recommended? Yes/No — because <reason>**
```

### Brutal-honesty checklist
- [ ] Matches existing patterns, or introduces a new abstraction?
- [ ] Hidden costs (build time, app size, runtime, CI)?
- [ ] Migration burden on unrelated screens?
- [ ] Is the user solving the right problem?
- [ ] Violates any `rules_file` rule?

If the idea is wrong / over-engineered / regression-prone — **say so directly.**

## Phase 5 — Decision & report
Path: `{reports_dir}/brainstorm-{YYMMDD-HHMM}-{TICKET|slug}.md`
(`{YYMMDD-HHMM}` from `bash -c 'date +%y%m%d-%H%M'`). Sacrifice grammar for concision.

```markdown
# Brainstorm: <topic>
**Ticket**: <id|n/a>  **Date**: <YYYY-MM-DD>  **Branch**: <current>
## Problem Statement
## Constraints & Requirements
## Approaches Evaluated  (1 / 2 / 3 — summary, pros, cons, effort/risk)
## Recommendation  (chosen + rationale)
## Implementation Considerations
- state shape / DI wiring / navigation / networking / localization + testTags
- feature flag (if feature_flags): <yes/no, name, default-off?>
- migration / rollback / testing strategy
## Risks & Mitigations  | Risk | Likelihood | Mitigation |
## Success Metrics
## Out of Scope
## Open Questions  (owner: PM / BE / Design)
```

## Phase 6 — Hand-off (menu, do NOT auto-invoke)
```
Brainstorm complete → {reports_dir}/<file>.md
- Create plan      → android-plan <ticket-or-topic>
- More research    → android-research <sub-topic>
- Deep reasoning   → android-sequential-thinking <problem>
- Stop here        → keep report as decision record
```
