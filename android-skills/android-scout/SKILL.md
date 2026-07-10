---
name: android-scout
description: "Codebase scouting for an Android project via parallel Explore subagents. Use to locate where a feature/symbol lives before edits, or to gather codebase context for a task."
argument-hint: "[search-target] [quick|full]"
---

# Scout — Android

Find things fast using parallel `Explore` subagents — they do the reading; never pull
full files into main context. Output is a **map**, not analysis.

## Step 0 — Load profile

Read `.claude/android-profile.md` for `source_roots`, `test_roots`, `generated_paths`,
`ui_framework`, `navigation`, `networking`. If absent, read the main checkout's copy —
`$(git rev-parse --path-format=absolute --git-common-dir)/../.claude/android-profile.md`
(the profile is usually gitignored, so worktrees don't inherit it). Still absent → tell the
user to run `android-project-init` first (greenfield repos have nothing to scout).

## When to Use

- A feature spans multiple modules / `source_roots`
- Before cross-module edits (shared state, DI registrations, network operations)
- DRY check — confirm a pattern exists before adding a new one

## When NOT to Use

| Case | Use instead |
|---|---|
| Single file, known path | `Read` |
| One symbol grep | `Grep` |
| Trade-off discussion | `android-brainstorm` |
| Full implementation plan | `android-plan` |
| Pattern outside the repo | `android-research` |

---

## Argument Parsing

- **TARGET** (required) — type name, feature area, function, network operation, or NL description.
- **Depth** — omitted → Step 1's estimate decides the agent count · `quick` → force 1 agent ·
  `full` → force 2-3.

If TARGET missing, ask via `AskUserQuestion`.

---

## Workflow

### 1. Estimate scale (cheap probes)

Use profile paths:
```
Glob {source_roots}/**/<term>*.kt
Glob {test_roots}/**/<term>*.kt
Grep <term> across the narrowed dirs
```

Agent count:
- **1** — < 50 matched files, single module
- **2** — feature touches app module + a library module
- **3** — split: app module / library modules / tests

Don't spawn agents for trivial single-file lookups (overhead > benefit).

### 2. Spawn parallel `Explore` agents (single message → concurrent)

Each gets a tight scope. Prompt template:

```
Scope: {absolute path under a source_root}
Search target: {TARGET}

Find:
1. Primary implementation files (Composables/screens, ViewModels/Reducers, Navigators/NavHost, Repositories/Services)
2. Tests covering them ({test_roots})
3. Direct usages: callers, navigation wiring, DI registrations
4. Related models / DTOs / network operations ({networking})
5. testTag usages (if a UI surface and test_tags != none)

Return brief markdown: `file_path:line — one-line description`.
Do NOT paste file contents. Cap at 25 most relevant matches.
```

**Never** scout into `generated_paths` from the profile.

### 3. Aggregate

Combine, dedupe, sort by relevance. **Verify, don't guess** — drop any returned path
that doesn't exist. Note any agent that timed out and continue.

---

## Report Format

```markdown
# Scout Report: <TARGET>

**Branch**: <current>   **Agents**: <N>

## Relevant Files
### App module
- `<path>:<line>` — <desc>
### Library modules
- `<path>:<line>` — <desc>
### Tests
- `<path>:<line>` — <desc>

## Patterns Observed
- <state pattern, DI mechanism, navigation wiring actually seen>

## Unresolved Questions
- <gap or ambiguity>
```

If invoked inline (no report needed), output to the user directly.
