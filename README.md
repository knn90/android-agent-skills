# android-* skill suite

A **project-agnostic Android engineering workflow** for Claude Code. Point it at any Android app, fill in
**one** file (`.claude/android-profile.md`), and you get a full **ticket → PR** pipeline plus
specialist reviewers (used automatically once installed). The skill bodies hardcode **no** app names, paths, architectures, or commands —
every project-specific fact lives in the profile and is read at runtime.

```
android-project-init ──→ .claude/android-profile.md        # run ONCE per project

android-resolve  ── one command: ticket/context → wayfinder → scout → execute → review → PR
      │
      ├─ android-scout ────────┐
      ├─ android-research ──────┤
      ├─ android-brainstorm ────┼─→ wayfinder ─→ android-execute ─→ android-code-review
      └─ android-sequential-thinking  (reasoning aid; plugs in anywhere)
                                          │ apply           │ route
                                          └─── android-specialists ───┘
                          (Compose · Coroutines · Testing · TDD · SOLID — Google-primary)

Planning is delegated to mattpocock-skills:wayfinder (charts a spec via its own sub-skills),
then /to-tickets slices it when needed — install mattpocock-skills alongside this suite.
```

## Setup (once per project)

1. **Install the skills** — recommended: install as a Claude Code **plugin** (handles
   install, update, and uninstall natively, across all your projects):
   ```
   /plugin marketplace add knn90/android-agent-skills
   /plugin install android-skills@android-agent-skills
   ```
   Skills are then invocable as `android-skills:android-resolve`, `android-skills:android-scout`, … and are
   auto-selected by description as usual. Update with `/plugin marketplace update android-agent-skills`;
   remove with `/plugin uninstall android-skills@android-agent-skills`. (The maintenance-only
   `android-skill-consolidate/` is intentionally not part of the plugin.)

   <details><summary>Manual install (no plugin)</summary>

   Copy the skill folders into your app or globally:
   ```bash
   cp -R android-skills/*       <project>/.claude/skills/    # the core workflow
   cp -R android-specialists/*  <project>/.claude/skills/    # optional specialists you want
   ```
   (Or into `~/.claude/skills/`.) **Don't** ship `android-skill-consolidate/` — it's repo-maintenance.
   </details>

   Newly installed skills appear after the Claude Code session is restarted/reloaded.
2. **Create the profile** — run **`android-project-init`**. It *detects* an existing app's architecture/paths, or sets conventions for a greenfield app, and writes `.claude/android-profile.md`. (Or copy `android-skills/android-project-init/android-profile.template.md` and fill it in by hand.)
3. **Specialists work automatically** once installed (step 1) — `android-code-review` routes change-typed
   slices to them and `android-execute` applies them while writing. No extra opt-in. The profile's
   `specialists:` field is an **optional override**:
   ```yaml
   # specialists: [android-compose-expert, android-coroutines-expert]   # restrict to just these
   # specialists: none                                                    # turn specialist routing off
   ```
   Leave it unset to auto-use every `android-*-expert` you installed.

## How to use it

### The whole loop, one command

```
android-resolve ABC-123
android-resolve "add pull-to-refresh to the orders list"
android-resolve ABC-123 --devs 2          # parallel dev team
android-resolve ABC-123 --solo --no-pr    # single agent, stop before the PR
```

`android-resolve` is the front door. It runs:

```
fetch ticket/context → create branch → wayfinder (spec) ──(you approve)──► to-tickets (if needed)
    → android-scout → android-execute (TDD; solo or --team N) → android-code-review → /commit → open PR
```

`wayfinder` charts the work as a spec — naming the destination, then resolving its open decisions
one at a time (driving its own sub-skills — prototyping, grilling, research, … — as each demands)
so you approve a **hardened** spec, not one full of unstated assumptions. For a small, clear change
it declines the map and the flow falls straight through to scout → execute.
You approve the spec **before** any code is written, and nothing is pushed without you.

### Or à la carte

Every skill also stands alone:

```
android-scout OrderListViewModel            # where does this live?
android-research "offline sync options"     # sourced technical research
android-brainstorm "offline support"        # brutal trade-off analysis → decision report
wayfinder "offline support"                  # chart the spec (you approve); a mattpocock-skills skill
android-execute <plan-path> --team 2         # implement (TDD; parallel worktree team)
android-code-review #42                      # adversarial, multi-lens review of a PR
```

## The skills

**Core workflow — `android-skills/`**

| Skill | Role |
|---|---|
| `android-project-init` | One-time: writes `.claude/android-profile.md` (detect existing app / set greenfield conventions). |
| `android-resolve` | **The front door.** ticket/context → wayfinder → scout → execute → review → PR. Orchestrates; never implements directly. |
| `android-scout` | Fast, token-efficient parallel code discovery — returns a *map*, not analysis. |
| `android-research` | Sourced technical research grounded in the codebase. |
| `android-brainstorm` | Brutally honest trade-off analysis → decision report. |
| `android-sequential-thinking` | Step-by-step reasoning aid; plugs in anywhere. |
| `wayfinder` *(mattpocock-skills)* | Planning front-end: charts the work as a spec through decision tickets, driving its own sub-skills (prototyping, grilling, research, …); `/to-tickets` then slices the spec. Approval gate on the spec. Not part of this suite — install `mattpocock-skills` alongside it. |
| `android-execute` | **The only implementer.** spec → code → verify → review. **TDD always**; solo, or `--team N` (parallel worktree devs + peer review + a dedicated edge-case reviewer + merge). |
| `android-code-review` | Multi-lens adversarial review; **routes change-typed slices to the specialists**; precision-over-recall findings. |

**Specialists — `android-specialists/` (optional add-ons; auto-used once installed)**

Deep domain reviewers, consolidated from curated community sources with **Google / official Android docs as the source of truth**. `android-code-review` triggers them per change type; `android-execute` applies them while writing.

| Specialist | Domain |
|---|---|
| `android-compose-expert` | Jetpack Compose — composition, state, performance, navigation, accessibility, Material 3 |
| `android-coroutines-expert` | Kotlin Coroutines & Flow — suspend, dispatchers, structured concurrency, StateFlow/SharedFlow, cancellation |
| `android-testing-expert` | Testing — JUnit **and** MockK/Turbine, Compose UI test, Robolectric, flakiness |
| `android-tdd` | TDD discipline — red→green→refactor, prove-it bug repro, vertical slices (process, pairs with `android-testing-expert`) |
| `android-solid-expert` | SOLID + decoupling — SRP/OCP/LSP/ISP/DIP, composition-root DI, framework isolation |

**Maintenance — `android-skill-consolidate/` (repo-only; not shipped)**

Rebuilds any specialist (or a core skill like `android-code-review`) from its `SOURCES.yaml`: a staleness audit + **web discovery** for newer/better sources + fan-out extraction + synthesis. Run it with no argument to pick a skill from a menu (or `all`). Keeps skills from rotting; Google/official Android docs always win on conflict.

## The profile — the one file that matters

`.claude/android-profile.md` is the single source of every project-specific fact: `source_roots`,
`architecture`, `state_type`, `di`, `navigation`, `networking`, `verify_command`,
`high_rigor_domains`, `specialists`, `ticket_fetch`, `default_base_branch`, `pr_tool`, … Every skill
reads it first, so the skill bodies stay generic. Start from
`android-skills/android-project-init/android-profile.template.md`.

The profile is usually **gitignored**, so git worktrees don't inherit it. Every skill falls
back to the main checkout's copy via
`$(git rev-parse --path-format=absolute --git-common-dir)/../.claude/android-profile.md`.

## Layout

```
android-agent-skills/
├── .claude-plugin/              # plugin + marketplace manifests (the one-command install)
│   ├── plugin.json              #   bundles android-skills/ + android-specialists/ as the plugin
│   └── marketplace.json         #   self-marketplace listing the plugin
├── README.md
├── android-skills/              # core workflow (8 skills)        — shipped by the plugin
│   └── android-project-init/android-profile.template.md   # copy to <project>/.claude/android-profile.md
├── android-specialists/         # optional specialist reviewers (5)   — shipped by the plugin
├── android-skill-consolidate/   # repo-maintenance (rebuilds skills from sources) — NOT shipped
└── docs/                        # design notes
```

## Core principles (every skill)

- **Profile-driven** — read `.claude/android-profile.md` first; never hardcode project facts.
- **Plan-first** — no implementation before an approved spec/plan (hard gate in `android-execute`).
- **TDD always** — `android-execute` writes a failing test before the code.
- **Verify before claiming** — run `verify_command` (Gradle), read the output, *then* claim done (empty → build-only, stated).
- **Google / official Android docs = source of truth** — specialists and review defer to developer.android.com over community sources.
- **HIGH-RIGOR escalation** — diffs touching the profile's `high_rigor_domains` get a mandatory adversarial pass + correctness audit.
- **Precision over recall** — reviews surface a focused list of real findings, not a wall of nits.
