---
# android-profile.md — single source of project-specific facts for the android-* skill suite.
# Every android-* skill reads this FIRST. Copy this file to `.claude/android-profile.md`
# at the project root and fill it in (or let `android-project-init` generate it).
#
# Rule of thumb: if a value differs between two Android apps, it belongs HERE, not in a skill body.

app_name:                 # e.g. Acme
state: existing           # existing | greenfield

# ── Repo layout ────────────────────────────────────────────────
build_system:             # Gradle-KTS | Gradle-Groovy   (build.gradle.kts vs build.gradle)
modules: []               # Gradle module paths, e.g. [:app, :core:*, :feature:*]; [:app] if single-module
source_roots: []          # e.g. [app/src/main/java, core/*/src/main/kotlin]
test_roots: []            # e.g. [app/src/test (unit/JVM), app/src/androidTest (instrumented)]
generated_paths: []       # NEVER edit — e.g. [**/build/, *_Generated.kt, hilt_aggregated_deps/, *.g.kt]
docs_root:                # docs/ | none
plans_dir:                # docs/Plans | .claude/plans
reports_dir:              # docs/Reports | .claude/reports
rules_file:               # CLAUDE.md | docs/Architecture.md | none  (always/never rules)

# ── Architecture ───────────────────────────────────────────────
architecture:             # MVVM | MVI | Clean-MultiModule | MVVM+UseCase | other
state_type:               # e.g. UiState sealed interface | MVI Reducer | StateFlow<UiState> | none
di:                       # Hilt | Koin | Dagger | manual
navigation:               # Navigation-Compose | Jetpack-Navigation-XML | Fragment | custom/manual
ui_framework:             # Compose | Views-XML | mixed
min_sdk:                  # e.g. 24
kotlin_version:           # Kotlin version, e.g. 2.0 — gates version-specific specialist guidance (K2, Compose compiler)

# ── Integrations (set to `none` if unused — skills skip the related checks) ──
networking:               # Retrofit-OkHttp | Ktor | Apollo-GraphQL | none
async:                    # Coroutines-Flow | RxJava | mixed   (primary async stack — gates android-coroutines-expert)
serialization:            # kotlinx.serialization | Moshi | Gson | none
localization:             # strings.xml (stringResource) | Lyricist/Compose | none
test_tags:                # testTag convention (Compose) / resource-id module for UI tests | none
feature_flags:            # Firebase-RemoteConfig | LaunchDarkly | none
crash_reporting:          # Crashlytics | Sentry | none
ticket_system:            # Jira | GitHub | Linear | none
ticket_pattern:           # regex, e.g. ABC-\d+  (used to detect a ticket id in args/branch)
ticket_fetch:             # MCP tool or CLI to fetch a ticket | none

# ── Workflow / PR (used by android-resolve + android-execute --team) ───
default_base_branch: main # branch-from point + PR target; overridable with --base
pr_tool:                  # gh | none   (none → android-resolve prints manual PR steps)

# ── Verification ───────────────────────────────────────────────
# Whatever this project actually runs to prove a change is correct.
# Free-form: ./gradlew testDebugUnitTest, ./gradlew :app:test, ./gradlew check,
#            ./gradlew connectedDebugAndroidTest, ./scripts/verify.sh …
# Empty/unset → skills fall back to BUILD-ONLY (./gradlew assembleDebug) and say so explicitly.
verify_command: |

# ── Rigor ──────────────────────────────────────────────────────
# Domains that force adversarial review + correctness audit every time.
high_rigor_domains: [checkout, payment, auth, PII, money]

# ── Specialist skills (optional add-ons) ───────────────────────
# ON BY DEFAULT: any installed android-*-expert is auto-used — android-code-review routes to it per change
# type, android-execute applies it while writing. This field is an OPTIONAL OVERRIDE.
specialists:              # unset = auto-use all installed · [android-compose-expert, …] = restrict · none = off
---

# Project Notes (free text)

Anything a skill should know that doesn't fit a field above — naming conventions,
forbidden patterns, "always do X / never do Y" rules specific to this app.
