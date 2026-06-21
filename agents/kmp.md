# agents/cmp.md — Kotlin Multiplatform with shared Compose UI + logic

> Inherits everything in `GLOBAL_RULES.md` and never loosens it. A project's
> `.claude/conventions.md` may tighten these. Class-C lessons live in
> `lessons/cmp.md`; recurring/explicitly-corrected patterns are promoted into
> **§ Patterns we've settled** at the bottom of this file by `/fix` (human-merged PR).
>
> Scope of this stack: **Android + iOS (desktop optional), shared UI** via
> **Compose Multiplatform** (one Compose UI in `commonMain`), **all logic shared**,
> **local-only** persistence. This is the shared-UI sibling of `kmp.md` (which keeps
> native SwiftUI/Compose per platform). Pick this when the UI itself is shared; pick
> `kmp.md` when each platform owns its own UI.

---

## Project shape (non-negotiable)

1. Structure: a shared logic library grouped as `:shared:domain`, `:shared:data`, `:shared:presentation`, plus a shared **UI** module `:composeApp` whose `commonMain` holds the Compose UI, plus thin platform shells — `androidApp` (a single Activity that hosts the root composable) and `iosApp` (an Xcode project that presents a `ComposeUIViewController`). Shells contain entry points and platform wiring only — **never business logic, never reimplemented screens**.
2. **The UI lives in `:composeApp/commonMain` as `@Composable` functions.** That is the point of this stack. The native shells host the shared UI; they do not rebuild it. (This is the exact inverse of `kmp.md` rule 2.)
3. Dependency direction is one-way: `:composeApp(UI) → :presentation → :domain ← :data`. `:domain` depends on nothing platform- or UI-specific. `:presentation` and `:data` never depend on each other, and the UI never depends on `:data`.
4. `:domain` must contain no `android.*`, no `platform.*`, no Compose, no Room, no Koin — plain Kotlin + coroutines. `:presentation` must contain **no Compose imports** either: it is framework-agnostic state (ViewModels + `StateFlow`) so it stays testable without the UI toolkit. Compose appears only in `:composeApp`.
5. The iOS app builds **only on macOS via Xcode**; the Compose iOS framework is produced on Mac. Shared + Android build anywhere. Never assume the iOS framework can be produced on a Windows dev box.

## Source sets & expect/actual

6. Prefer `commonMain` for everything — logic **and** UI. Drop to `androidMain`/`iosMain` only for genuine platform APIs (SQLite driver, secure storage, clock/uuid) and for **platform-specific UI** (e.g. a native map or camera view) expressed as `@Composable expect`/`actual`.
7. Use `expect`/`actual` sparingly, only at the platform edge. A platform composable is `@Composable expect fun PlatformX(...)` in `commonMain` with `actual` bodies in the platform source sets — don't scatter expect/actual through ordinary UI. An `expect` in `:domain` is a smell; inject the platform thing instead (see DI).
8. Keep `actual` implementations minimal and free of any dependency on higher layers; they wire a platform primitive, nothing more.

## Domain layer (`:shared:domain`)

9. Domain models are immutable `data class`es with no persistence/serialization annotations and no nullable-everything: model the real invariants.
10. Repository **interfaces** are declared here; implementations live in `:data`. Domain never sees a concrete repository.
11. Each use case is a single-responsibility class with the `UseCase` suffix, exposing one `operator fun invoke(...)`. Use cases orchestrate; they hold no platform state.
12. **Error handling = sealed `DomainResult<T>`.** Define one `sealed interface DomainResult<out T> { data class Success<T>(val data: T); data class Failure(val error: DomainError) }` with a sealed `DomainError` hierarchy. This is the KMP realization of GLOBAL rule 13 — **do not throw across layer boundaries**, do not use `kotlin.Result`, do not surface exceptions to `:presentation`/UI.
13. Suspend functions are allowed in domain; flows returned from domain are **cold** `Flow<T>`. Hot state belongs to `:presentation`.

## Data layer (`:shared:data`)

14. Persistence is **androidx Room (KMP)** (`androidx.room`) with **KSP** codegen. `@Entity`/`@Dao`/`@Database` define schema + typed queries; migrations are versioned Room migrations — never mutate a shipped schema in place. _(Stack default since L6; not SQLDelight.)_
15. The RoomDatabase is built per platform via a `RoomDatabaseConstructor` `expect`/`actual` using `BundledSQLiteDriver`, and **provided through Koin**, never constructed inside a repository.
16. **Room entity types never leave `:data`.** Map entities → domain models with explicit mapper functions (a `…Mappers.kt` per feature). An entity type appearing in `:domain`, `:presentation`, or the UI is a hard stop.
17. Repository implementations catch Room/driver/IO exceptions at the boundary and convert them to `DomainResult.Failure(DomainError.…)`. No Room/SQL exception escapes `:data`.
18. Reactive reads use Room `Flow` queries (a `@Query` returning `Flow<…>`) on a background dispatcher; never block.

## Presentation layer (`:shared:presentation`)

19. State holders are **multiplatform `ViewModel`s** (`org.jetbrains.androidx.lifecycle` viewmodel artifact), one per screen, named `<Screen>ViewModel`, **with no Compose imports**.
20. Each ViewModel exposes a single immutable `StateFlow<XUiState>` (one screen = one state object) and **discrete intent functions** (`addNote()`, `onQueryChange()`, `deleteNote(id)`), not a single MVI event funnel.
21. `UiState` is an immutable `data class` with an **explicit, always-handled** loading / empty / error / content representation — never a bare nullable list. Map `DomainResult.Failure` into a user-facing error field on the state, never a thrown error.
22. Build state with `stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), initial)`. ViewModels depend on use cases (or repository interfaces) only — never on `:data`, drivers, Compose, or platform types.

## Shared UI layer (`:composeApp`)

23. **Composables are stateless and hoisted.** A screen composable receives a `UiState` value and intent lambdas, renders, and calls the lambdas. **No business logic, IO, or data shaping in composition** — this is the Compose form of "no logic in `build()`". Derive and format in the ViewModel, not in the composable.
24. Collect state with **`collectAsStateWithLifecycle()`** (multiplatform `lifecycle-runtime-compose`), never bare `collectAsState()`, so collection respects lifecycle uniformly on Android and iOS.
25. Obtain ViewModels through the multiplatform `viewModel { }` (lifecycle-viewmodel-compose) wired via Koin. **Never** construct a ViewModel inside a composable and never hold one in `remember`.
26. `remember` / `rememberSaveable` is only for transient, throwaway UI state (focus, expanded toggles, scroll). Anything that must survive process death or feeds logic lives in the ViewModel/state.
27. **All shared assets go through Compose Resources** (`org.jetbrains.compose.components:components-resources`): strings, drawables, and fonts in `composeApp/commonMain/composeResources/`, accessed via the generated `Res` (`stringResource(Res.string.x)`, `painterResource(Res.drawable.y)`, `Font(Res.font.z)`). No Android `R.`, no duplicated per-platform assets. Ship extended-glyph fonts (ē, ā, ō…) here as a Compose Resource rather than relying on a platform default.
28. One shared theme (`MaterialTheme` or a custom design system) defined once in `commonMain`; no per-platform theme forks. Platform look-and-feel differences flow through theme tokens, not duplicated UI trees.
29. **Navigation is shared**: a single `NavHost` with type-safe routes via JetBrains Navigation Compose (`org.jetbrains.androidx.navigation:navigation-compose`), defined in `:composeApp/commonMain`. (Inverse of `kmp.md`'s "no shared navigation".) Don't add a second navigation library without a promoted lesson; **Decompose** (`com.arkivanov.decompose`) is the sanctioned alternative when component-scoped lifecycles are genuinely needed.
30. Platform-only UI uses `@Composable expect`/`actual` (rule 7), kept to the genuine minimum.
31. Every screen has a `@Preview` (`org.jetbrains.compose.ui.tooling.preview`) driven by a fake `UiState`. A preview must not need a ViewModel, Koin, or a driver.

## iOS hosting (this stack does **not** use SKIE)

32. iOS hosts the shared UI through a single entry exposed from `:composeApp/iosMain`: `fun MainViewController() = ComposeUIViewController { App() }`. The Xcode `iosApp` is a thin shell that presents it — **no SwiftUI screens, no per-screen Swift**.
33. **SKIE is not part of this stack.** iOS consumes Compose UI, not Kotlin `Flow`s from Swift, so there is no Swift↔Kotlin flow bridging. Keep the exported framework surface to the single UI entry (`MainViewController`) plus `initKoin()`. Don't export domain/data/Koin internals.
34. The Compose runtime owns recomposition and UI lifecycle on both platforms. There is **no manual SwiftUI teardown of ViewModels** (the divergence from `kmp.md` rule 23) — lifecycle is handled by the multiplatform lifecycle/navigation stack; don't hand-roll `dispose()` paths.
35. Collect UI state on main via `collectAsStateWithLifecycle`; keep heavy work off-main through injected dispatchers (see Concurrency). A background-emitting flow driving recomposition is a defect.

## Dependency injection (Koin)

36. **Koin** is the only DI. Each layer contributes a module (`domainModule`, `dataModule`, `presentationModule`, and a `uiModule` only if UI needs DI-provided values); shells call `startKoin { modules(...) }` at entry.
37. Platform primitives (drivers, secure storage, dispatchers) are provided via Koin from `androidMain`/`iosMain` — not via `expect`/`actual` singletons. iOS exposes an `initKoin()` the SwiftUI/UIKit shell calls once at launch; Android calls it from `Application`.
38. No service-locator calls (`getKoin().get()`) inside domain/data/presentation classes — constructor-inject everything; Koin only assembles graphs. In composables, use `koinViewModel()`/`koinInject()` at the screen boundary only.
39. **Dispatchers are injected**, never hardcoded. Provide a `DispatcherProvider` so tests swap in a test dispatcher.

## Concurrency

40. All blocking/IO work runs off the main thread via injected dispatchers. `runBlocking` is forbidden outside tests.
41. Structured concurrency only: no `GlobalScope`. Background work is scoped to `viewModelScope` or an injected scope that something owns and cancels.

## Testing (full TDD — all layers incl. UI)

42. **Test first.** New behavior starts as a failing test in `commonTest`, then implementation (tightens GLOBAL rule 16).
43. Layer coverage: `:domain` use cases unit-tested in `commonTest`; `:data` repositories tested against an **in-memory Room database**; `:presentation` ViewModels tested by asserting `StateFlow` transitions.
44. **Shared UI is tested once, in `commonTest`, with Compose Multiplatform UI tests** (`runComposeUiTest`): assert a screen renders its loading/empty/error/content states from a fake `UiState` and that tapping fires the right intent lambda. Don't snapshot per platform — the UI is shared, so test it shared.
45. `Flow`/`StateFlow` assertions use **Turbine**; coroutines use `kotlinx-coroutines-test` with an injected `StandardTestDispatcher`; Koin graphs verified with `koin-test` `checkModules()`.
46. Tests assert on the **`DomainResult` value** (success/failure), never on thrown exceptions. A test expecting a thrown domain error is wrong by construction.
47. Each platform keeps a smoke test that the app/DI graph initializes.

## Build, tooling & quality gates

48. Compose is enabled via the **`org.jetbrains.compose`** plugin **and** the Compose compiler plugin **`org.jetbrains.kotlin.plugin.compose`** (required on Kotlin 2.x), applied through `build-logic` convention plugins — never copy-pasted per module.
49. Single source of dependency truth: **`gradle/libs.versions.toml`** — no inline version strings in any `build.gradle.kts`. Cross-module config lives in a **`build-logic` composite build** (convention plugins).
50. **ktlint + detekt** run on every module and are wired into the OpenCode review + CI; add a Compose-aware lint ruleset (stable params, no state created in composables). A formatting/detekt violation fails the gate; no suppressions without an inline justified `@Suppress`.
51. Canonical libraries for this stack (coordinates fixed, versions pinned in the catalog and verified against latest stable at adoption): coroutines (`kotlinx-coroutines-core`), `kotlinx-datetime`, `kotlinx-serialization-json` (export/backup), stdlib `Uuid` for IDs, Kermit (`co.touchlab:kermit`), **Room KMP** (`androidx.room`) + **KSP** (`com.google.devtools.ksp`) + sqlite-bundled (`BundledSQLiteDriver`), Koin + `koin-compose` (`io.insert-koin`), Turbine (`app.cash.turbine`), **Compose Multiplatform** (`org.jetbrains.compose`), **Navigation Compose** (`org.jetbrains.androidx.navigation`), **lifecycle-viewmodel-compose + lifecycle-runtime-compose** (`org.jetbrains.androidx.lifecycle`), **Compose Resources** (`org.jetbrains.compose.components:components-resources`). Don't introduce an alternative to one of these without a promoted lesson.
52. Secrets/config: nothing baked into source (GLOBAL rule 8). The local-only app has no network keys, but signing config, store credentials, and any future endpoint config are injected at build/run time and git-ignored.

## Naming & packages

53. **Package-by-feature inside every module** (`…domain.note`, `…data.note`, `…presentation.note`, `…ui.note`, then `…settings`, etc.), not package-by-type.
54. Conventions: repository interface `XRepository` in domain / `XRepositoryImpl` in data; use cases `VerbNounUseCase`; state `XUiState`; ViewModel `XViewModel`; Koin module `xModule`; screen composable `XScreen` (stateful entry that wires the ViewModel) delegating to a stateless `XContent(state, onIntent…)`; Room types per aggregate (`NoteEntity`/`NoteDao`, one `AppDatabase`).
55. One public class per file in shared code; file name matches the type.

---

## Patterns we've settled

> Seeded with known Compose-Multiplatform gotchas for this stack. `/fix` appends a
> positive rule + short example here on recurrence (`Seen: 2`) or explicit user
> correction.

- **Composables stay stateless.** Screen composable takes `UiState` + intent lambdas and renders; no logic, IO, or formatting in composition — derive it in the ViewModel. _(Realizes rule 23.)_
- **Collect with lifecycle.** Use `collectAsStateWithLifecycle()`, never bare `collectAsState()`, so flows pause when the UI is away — uniformly on Android and iOS. _(Realizes rule 24.)_
- **Assets via Compose Resources.** Strings, drawables, and fonts go in `composeResources/` and load through the generated `Res`; never Android `R.` or duplicated platform assets. Ship extended-glyph fonts here. _(Realizes rule 27.)_
- **iOS is a host, not a UI.** Expose one `MainViewController()` returning `ComposeUIViewController { App() }`; the Xcode shell only presents it. No SwiftUI screens, no SKIE flow bridging. _(Realizes rules 32–33.)_
- **One shared NavHost.** Type-safe routes in `commonMain`; don't fork navigation per platform or add a second nav library without a promoted lesson. _(Realizes rule 29.)_

### Persistence — settled 2026-06-21
Do: use androidx Room (KMP) — `@Entity`/`@Dao`/`@Database` + KSP, `RoomDatabaseConstructor` `expect`/`actual`, `BundledSQLiteDriver`, RoomDatabase provided via Koin. Entities stay in `:data`; map to domain models. Reactive reads = Room `Flow` queries; IO failure → `DomainResult.Failure`.
Avoid: SQLDelight; building the DB inside a repo; returning Room entities from `:data`.
Example:
```kotlin
@Dao interface NoteDao { @Query("SELECT * FROM NoteEntity") fun observeAll(): Flow<List<NoteEntity>> }
// repo maps NoteEntity -> Note; catches IO -> DomainResult.Failure
```
_(Realizes rules 14–18; from L6.)_

### Foundation build gate — settled 2026-06-21
Do: before opening a foundation/scaffolding PR, run a real two-target build, commit only when green:
```
./gradlew assembleDebug compileKotlinIosArm64 compileKotlinIosSimulatorArm64
```
Avoid: committing a module/Gradle skeleton that was never built. One green build catches the whole class below.
Watch (all caught by the build gate):
- Catalog plugin alias is kebab, accessor is dotted: `kotlin-multiplatform` → `libs.plugins.kotlin.multiplatform` (a camelCase alias gives a single-segment accessor and won't resolve).
- KSP version must match Kotlin exactly; the newest Kotlin often has no KSP yet — pin Kotlin to the latest version that does.
- With `includeBuild("build-logic")`, declare every plugin in the root `plugins {}` as `alias(...) apply false`; a versioned alias applied in a subproject for a plugin build-logic also leaks fails with "already on the classpath with an unknown version."
- AGP 9 has built-in Kotlin: drop `org.jetbrains.kotlin.android` from `com.android.application` modules.
- `room-ktx` is Android-only — never in `commonMain` (it drags `kotlinx-coroutines-android` into iOS). `room-runtime` covers KMP.
- Koin 4.x moved the `viewModel { }` module DSL to `io.insert-koin:koin-core-viewmodel`, not `koin-core`.
- A module must depend on what it uses; `implementation` is not transitive — use it directly or expose via `api`. _(Realizes GLOBAL_RULES rule 20.)_
_(From the issue-3 foundation that merged without ever building.)_