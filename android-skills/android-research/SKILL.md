---
name: android-research
description: "Strategic technical research for an Android project (Kotlin / Jetpack Compose / Views / networking / coroutines / security / accessibility). Use for library evaluation, framework pattern questions, migration questions, CVE checks. Outputs a sourced report."
argument-hint: "[topic | TICKET-ID]"
---

# Research — Android

Strategic research producing an actionable, sourced report. **Honest, brutal, concise.**
**Bias toward what already works in the repo.** Don't recommend a library the project
doesn't use unless you can show the existing pattern is genuinely insufficient.

## Step 0 — Load profile

Read `.claude/android-profile.md`: `min_sdk`, `architecture`, `networking`, `ui_framework`,
`reports_dir`, `ticket_system`/`ticket_pattern`/`ticket_fetch`. If missing, read the main
checkout's copy — `$(git rev-parse --path-format=absolute --git-common-dir)/../.claude/android-profile.md`
(the profile is usually gitignored, so worktrees don't inherit it). Still missing → run
`android-project-init`.

## When NOT to Use

| Case | Use instead |
|---|---|
| Pattern already exists in repo | `android-scout` |
| API doc for a known library | `WebFetch` against the official URL |
| Trade-off discussion of approaches | `android-brainstorm` |
| Step-by-step debug reasoning | `android-sequential-thinking` |

---

## Argument Parsing

- **TICKET-ID** (matches `ticket_pattern`) → fetch via `ticket_fetch` for context.
- **Free-form topic** → research as-is. If missing, ask via `AskUserQuestion`.

---

## Process

### Phase 1 — Scope
- Key terms; recency window (< 12 months unless historical); in/out of scope.

### Phase 2 — Ground in the codebase (FIRST)
Before any web search:
- Is there already a working pattern? → `android-scout` / `Grep`.
- What's pinned? → `gradle/libs.versions.toml`, `build.gradle(.kts)`, `gradle.lockfile`.
- What's documented? → `rules_file` + `docs_root`.
- **If the codebase already solves it → say so and stop.** That's a successful outcome.

### Phase 3 — External research (≤ 5 search calls)
Preferred order: Android official docs (developer.android.com) → Google I/O · Android Dev
Summit (current + last 2y) → library official docs (e.g. networking lib for `networking`) →
Kotlin docs / KEEP → reputable Android eng blogs (Now in Android, Chris Banes, Jake Wharton,
Zach Klippenstein) → recent GitHub Discussions/SO.

Query patterns: `<topic> jetpack compose android <min_sdk>`, `<topic> kotlin coroutines flow`,
`<topic> <networking-lib> <version>`, `<topic> CVE <year>`.

### Phase 4 — Cross-reference & validate
- 2+ independent sources per claim; flag conflicts.
- Reject APIs above `min_sdk` unless gated (`Build.VERSION.SDK_INT`/`@RequiresApi`) or the user
  agrees to bump `minSdk`.
- Discard anything older than the framework major version in use.

### Phase 5 — Report
Save to `{reports_dir}/research-{YYMMDD-HHMM}-{TICKET|slug}.md`.
`{YYMMDD-HHMM}` MUST come from `bash -c 'date +%y%m%d-%H%M'`, not model memory.
Create `{reports_dir}` if absent.

---

## Report Template

```markdown
# Research: <topic>

**Ticket**: <id | n/a>   **Date**: <YYYY-MM-DD>   **Branch**: <current>
**Versions**: minSdk <min_sdk>, Kotlin <Y>, <networking-lib <Z> if any>  ← from repo

## Executive Summary
<3-5 bullets — findings + recommendation>

## Codebase Context
- Existing pattern: <what's there> (path:line)
- Why current approach falls short: <reason | "n/a — greenfield">

## Findings
### Best Practices (current consensus)
### Security / Privacy  (Android Keystore-backed encryption (Tink/DataStore), Network Security Config, biometrics, permissions — flag affected pinned versions)
### Performance Insights  (numbers > vibes)
### Comparative Analysis (if multiple options)
| Option | Effort | Risk | Maintenance | Fit |

## Recommendation
**Chosen**: …   **Rationale**: …

## Implementation Sketch
- Affected modules / state shape / DI / networking / localization impact

## Sources
1. [Title](url) — checked YYYY-MM-DD

## Open Questions
- <unresolved — owner: PM / BE / Design>
```

## Constraints

- **DO NOT** implement — research only.
- **MUST** cite sources with check dates.
- **Sacrifice grammar for concision** — bullet > paragraph.
