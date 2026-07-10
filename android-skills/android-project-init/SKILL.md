---
name: android-project-init
description: "One-time bootstrap for the android-* skill suite. Creates .claude/android-profile.md — the single source of project-specific facts every other android-* skill reads. Use before first running any android-* skill on a project, or whenever the profile is missing or stale."
argument-hint: "[--greenfield | --detect]"
---

# Project Init — android-* suite bootstrap

Produce `.claude/android-profile.md`: the one file that makes the generic `android-*` skills
concrete for THIS project. This skill only writes the profile (+ optional docs scaffold) —
never invoke implementation skills from it.

The profile holds **facts, not opinions**: if a value differs between two Android apps it
belongs in the profile; everything else stays in the skill bodies. Keep it minimal.

---

## Two modes

| Repo state | Mode | Profile is… |
|---|---|---|
| Empty / no source files yet | **greenfield** (prescriptive) | a record of decisions you make up front |
| Has `build.gradle(.kts)`/`settings.gradle(.kts)` + source | **detect** (descriptive) | a record of reality read from the code |

Auto-detect the mode:
```
Glob **/build.gradle.kts, **/build.gradle, **/settings.gradle.kts, **/settings.gradle
Glob **/src/main/**/*.kt, **/*.kt   (any real source?)
```
- Source present → `--detect`. Empty/scaffold only → `--greenfield`.
- `$ARGUMENTS` may force `--greenfield` / `--detect`.

If a profile already exists, show it and ask (via `AskUserQuestion`): keep / refresh / overwrite.

---

## Mode A — Detect (existing app)

Read the codebase, fill the profile from evidence. **Verify every value — never guess.**

| Profile field | How to detect |
|---|---|
| `build_system` | `build.gradle.kts` present → Gradle-KTS · `build.gradle` (Groovy) → Gradle-Groovy |
| `modules` | `include(...)` entries in `settings.gradle(.kts)`; `[:app]` if single-module |
| `source_roots` / `test_roots` | dirs containing `src/main/**/*.kt` / `src/test` (unit/JVM) + `src/androidTest` (instrumented) |
| `architecture` | grep: `@HiltViewModel`/`dagger.hilt`→Hilt DI stack · `androidx.compose`→Compose · `MVI`/`Reducer`/`intent(` patterns→MVI · `viewModelScope`+`StateFlow`→MVVM · multi-module `:core:`/`:feature:`→Clean-MultiModule |
| `state_type` | grep for a shared `sealed interface UiState`/`sealed class …State`, MVI `Reducer`, `StateFlow<UiState>`, else none |
| `di` | grep imports: `dagger.hilt`→Hilt · `org.koin`→Koin · `dagger.` (no hilt)→Dagger · else manual |
| `navigation` | `NavHost`/`composable(`→Navigation-Compose · `nav_graph.xml`/`<navigation>`→Jetpack-Navigation-XML · `FragmentTransaction`→Fragment · custom |
| `networking` | `retrofit2`/`okhttp3`→Retrofit-OkHttp · `io.ktor`→Ktor · `com.apollographql`/`*.graphql`→Apollo-GraphQL · none |
| `async` | `kotlinx.coroutines`/`Flow`→Coroutines-Flow · `io.reactivex`→RxJava · both→mixed |
| `serialization` | `kotlinx.serialization`/`@Serializable` · `com.squareup.moshi`→Moshi · `com.google.gson`→Gson · none |
| `localization` | `strings.xml` + `stringResource(`→strings.xml · Lyricist/Compose · none |
| `test_tags` | grep `testTag(` (Compose) / resource-id usage in Espresso, else none |
| `feature_flags` / `crash_reporting` | grep `RemoteConfig`/`LaunchDarkly`, `Crashlytics`/`Sentry` |
| `generated_paths` | `**/build/` dirs, generated Hilt/KSP/KAPT output (`hilt_aggregated_deps/`, `*.g.kt`), files with "generated" headers |
| `verify_command` | existing `scripts/*verify*`, a Gradle test task, else propose `./gradlew testDebugUnitTest` (or `./gradlew check`); build-only fallback `./gradlew assembleDebug` |
| `rules_file` | `CLAUDE.md` else `docs/Architecture.md` else none |
| `ticket_system`/`ticket_pattern` | branch names + `git log` (e.g. `ABC-123`), or ask |
| `high_rigor_domains` | grep for `Checkout`/`Payment`/`Auth`/`Profile`; default `[auth, PII]` if none |
| `default_base_branch` | `git symbolic-ref refs/remotes/origin/HEAD` (repo default), else `main` |
| `pr_tool` | `gh` if the GitHub CLI is installed + authed, else `none` |
| `min_sdk` | `minSdk` in the app `build.gradle(.kts)` (or `defaultConfig`) |
| `kotlin_version` | Kotlin version in `libs.versions.toml` / root `build.gradle(.kts)`, else `./gradlew -version` |
| everything else (`app_name`, `ui_framework`, `docs_root`/`plans_dir`/`reports_dir`, …) | build files / `applicationId` / obvious repo layout |

Use `Explore`/`Grep`/`Glob` (or invoke `android-scout` if the repo is large).
Done = every template field filled or explicitly `none` — no silent blanks. Confirm the
detected `architecture`, `verify_command`, and `high_rigor_domains` with the user before
writing — these three drive the most downstream behaviour.

---

## Mode B — Greenfield (new app)

Nothing to detect. **Decide** the conventions via `AskUserQuestion`, then record them.

Ask (bundle into 2-4 `AskUserQuestion` groups):

1. **Architecture** — MVVM · MVI · Clean-MultiModule · MVVM+UseCase · other.
   For each, state the trade-off honestly (testability, boilerplate, team familiarity,
   module-count fit). If the user is unsure, recommend the boring proven default for their
   stated team size and push back on novelty. *(This is `android-brainstorm` applied to
   foundations — borrow its brutal-honesty stance.)*
2. **UI framework + min SDK** — Compose (Compose-primary default) / Views-XML / mixed; `minSdk`.
3. **Networking** — Retrofit-OkHttp · Ktor · Apollo-GraphQL · none yet.
4. **DI (Hilt vs Koin) · async (Coroutines vs Rx) · navigation · localization** — pick or defer (`none` is valid early).
5. **Verify command** — default suggestion `./gradlew assembleDebug` (no tests yet) or
   `./gradlew testDebugUnitTest` once a test source set exists. May be left **empty** → build-only.
6. **Ticket system** + pattern, or `none`.
7. **HIGH-RIGOR domains** — which of checkout/payment/auth/PII this app will have.

Do **NOT** generate a `verify.sh` or any wrapper script. The verify mechanism is just the
`verify_command` string — keep it as configuration, not a committed artifact.

Optionally (ask first) create the docs scaffold the other skills expect:
`{docs_root}/Plans/` and `{docs_root}/Reports/`, and a starter `rules_file` (CLAUDE.md)
capturing the always/never rules implied by the chosen architecture.

---

## Output

Write `.claude/android-profile.md` from `android-profile.template.md` (sibling of this SKILL.md
in the skill's base directory), filled in. If the sibling is missing (untracked skill
files don't appear in git worktrees), read the main checkout's copy — same relative path
under `$(git rev-parse --path-format=absolute --git-common-dir)/..`. Then:

```
✓ Profile written: .claude/android-profile.md
- Mode: detect | greenfield
- Architecture: <…>   Networking: <…>   Verify: <command or "build-only (unset)">
- HIGH-RIGOR domains: <…>
- Specialists available: <installed android-*-expert skills, or none>

Next: the android-* skills are now live for this project.
- Find code            → android-scout
- Plan a feature       → android-plan
- Implement            → android-execute
- Resolve end-to-end   → android-resolve <ticket | "description">  (scout→plan→execute→review→PR)
```
