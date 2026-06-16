# agents/kmp.md — Kotlin Multiplatform (shared logic + native UI) rules

> Inherits everything in `GLOBAL_RULES.md` and never loosens it. A project's
> `.claude/conventions.md` may tighten these. Class-C lessons live in
> `lessons/kmp.md`; recurring/explicitly-corrected patterns are promoted into
> **§ Patterns we've settled** at the bottom of this file by `/fix` (human-merged PR).
>
> Scope of this stack: **Android + iOS, native UI per platform** (Jetpack Compose +
> SwiftUI), **all logic shared** in a Kotlin Multiplatform library, **local-only**
> persistence. No Compose Multiplatform UI, no shared navigation.

---

## Project shape (non-negotiable)

1. Use the **current KMP default structure**: a shared library plus separate runnable app modules. The shared library is grouped as `:shared:domain`, `:shared:data`, `:shared:presentation`; the apps are `androidApp` (Compose) and `iosApp` (Xcode/SwiftUI). The app modules contain entry points and platform wiring only — **never business logic**.
2. **No UI in any `commonMain`.** `commonMain` holds Kotlin logic only. Composable functions live in `androidApp`; SwiftUI lives in `iosApp`. A UI import in shared code is a hard stop.
3. Dependency direction is one-way: `:presentation → :domain ← :data`. `:domain` depends on **nothing platform-specific** — pure Kotlin + coroutines only. `:presentation` and `:data` may never depend on each other.
4. `:domain` must contain no `android.*`, no `platform.*`, no SQLDelight, no Koin, no framework types. If it won't compile as plain Kotlin, it's in the wrong module.
5. The iOS app builds **only on macOS via Xcode**. Never assume the iOS framework can be produced on the Windows dev box; shared + Android build anywhere, iOS framework + SwiftUI are Mac-only.

## Source sets & expect/actual

6. Prefer `commonMain` for everything. Drop to `androidMain`/`iosMain` **only** for genuine platform APIs (SQLite driver, secure storage, platform clock/uuid if needed).
7. Use `expect`/`actual` sparingly and only at the data/platform edge. An `expect` declaration in `:domain` is a smell — inject the platform thing instead (see DI rules).
8. Keep `actual` implementations minimal and dependency-free of higher layers; they wire a platform primitive, nothing more.

## Domain layer (`:shared:domain`)

9. Domain models are immutable `data class`es with no persistence/serialization annotations and no nullable-everything: model the real invariants.
10. Repository **interfaces** are declared here; implementations live in `:data`. Domain never sees a concrete repository.
11. Every use case is a single-responsibility class named with the `UseCase` suffix, exposing one `operator fun invoke(...)`. Use cases orchestrate; they hold no platform state.
12. **Error handling = sealed `Result<T>`.** Define one project `sealed interface DomainResult<out T> { data class Success; data class Failure(val error: DomainError) }` with a sealed `DomainError` hierarchy. This is the KMP realization of GLOBAL rule 13 (failures as values across boundaries) — **do not throw across layer boundaries**, do not use `kotlin.Result`, do not surface exceptions to `:presentation`.
13. Suspend functions are allowed in domain; flows returned from domain are **cold** `Flow<T>`. Hot state belongs to `:presentation`.

## Data layer (`:shared:data`)

14. Persistence is **SQLDelight 2.x** (`app.cash.sqldelight`) with the coroutines-extensions. `.sq` files define schema + typed queries; migrations are versioned `.sqm` files — never mutate a shipped schema in place.
15. The `SqlDriver` is created per platform (`AndroidSqliteDriver` in `androidMain`, `NativeSqliteDriver` in `iosMain`) and **provided through Koin**, never constructed inside a repository.
16. SQLDelight-generated types **never leave `:data`.** Map generated rows → domain models with explicit mapper functions. A SQLDelight type appearing in `:domain` or `:presentation` is a hard stop.
17. Repository implementations catch driver/IO exceptions at the boundary and convert them to `DomainResult.Failure(DomainError.…)`. No SQL exception escapes `:data`.
18. Reactive reads use SQLDelight `asFlow().mapToList(...)` on a background dispatcher; never block.

## Presentation layer (`:shared:presentation`)

19. State holders are **KMP `ViewModel`s** (the multiplatform `lifecycle-viewmodel` artifact), one per screen, named `<Screen>ViewModel`.
20. Each ViewModel exposes a single immutable `StateFlow<…UiState>` (one screen = one state object) and **discrete intent functions** (`addNote()`, `onQueryChange()`, `deleteNote(id)`), not a single MVI event funnel.
21. `UiState` is an immutable `data class` with an **explicit, always-handled** representation of loading / empty / error / content — never a bare nullable list. Map `DomainResult.Failure` into a user-facing error field on the state, never a thrown error.
22. Build state with `stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), initial)`. ViewModels depend on use cases (or repository interfaces) only — never on `:data`, drivers, or platform types.
23. **iOS owns the ViewModel lifecycle.** The SwiftUI view creates the ViewModel and must cancel/clear it on disappear — Android's framework clears it automatically, iOS does not. Encode a `dispose()`/clear path and call it from SwiftUI `onDisappear`. A leaked `viewModelScope` on iOS is a defect.

## iOS interop (SKIE)

24. **SKIE** (`co.touchlab.skie` Gradle plugin) is mandatory on the framework. It bridges `StateFlow`/`Flow` into Swift `AsyncSequence` and `suspend` into Swift `async` — design public shared APIs to be SKIE-friendly (sealed classes for exhaustiveness, no `KotlinInt`-style boxing in hot paths).
25. Everything the iOS app touches must be `public` in shared code and expressed in SKIE-translatable shapes. Internal helpers stay `internal`.
26. **All state reaching iOS is emitted on the main dispatcher.** Collecting a background-emitting `StateFlow` into SwiftUI is a defect; switch to `Dispatchers.Main` at the ViewModel's exposed boundary.
27. Keep the exported framework surface small and intentional — every public shared symbol is iOS API. Don't export data-layer or Koin internals.

## Dependency injection (Koin)

28. **Koin** is the only DI. Each module contributes a Koin module (`domainModule`, `dataModule`, `presentationModule`); the app modules call `startKoin { modules(...) }` at entry.
29. Platform primitives (drivers, secure storage, dispatchers) are provided via Koin from `androidMain`/`iosMain` — not via `expect`/`actual` singletons. iOS exposes a `KoinHelper`/`initKoin()` entry the SwiftUI app calls once at launch.
30. No service-locator calls (`getKoin().get()`) inside domain/data classes — constructor-inject everything; Koin only assembles graphs.
31. **Dispatchers are injected**, never hardcoded. Provide a `DispatcherProvider` so tests swap in a test dispatcher.

## Concurrency

32. All blocking/IO work runs off the main thread via injected dispatchers. `runBlocking` is forbidden outside tests.
33. Structured concurrency only: no `GlobalScope`. Background work is scoped to `viewModelScope` or an injected scope that something owns and cancels.

## Testing (full TDD — all layers)

34. **Test first.** New behavior starts as a failing test in `commonTest`, then implementation. This is the default for every issue on this stack (tightens GLOBAL rule 16).
35. Layer coverage: `:domain` use cases unit-tested in `commonTest`; `:data` repositories tested against the SQLDelight **in-memory/JVM driver**; `:presentation` ViewModels tested by asserting `StateFlow` transitions.
36. `Flow`/`StateFlow` assertions use **Turbine**. Coroutines use `kotlinx-coroutines-test` with an injected test dispatcher (`StandardTestDispatcher`). Koin graphs verified with `koin-test` `checkModules()`.
37. Tests assert on the **`DomainResult` value** (success/failure variants), never on thrown exceptions. A test that expects a thrown domain error is wrong by construction.
38. Each platform keeps at least a smoke test that the driver/DI graph initializes (`androidUnitTest` / `iosTest`).

## Build, tooling & quality gates

39. Single source of dependency truth: **`gradle/libs.versions.toml`**. No inline version strings in any `build.gradle.kts`. Versions are pinned in the catalog and verified against latest stable at adoption time.
40. Cross-module Gradle config lives in a **`build-logic` composite build** (convention plugins). The three shared modules and the app modules apply convention plugins, not copy-pasted config.
41. **ktlint + detekt** run on every module and are wired into the OpenCode review + CI. A formatting or detekt violation fails the gate; no warnings suppressed without an inline justified `@Suppress`.
42. Canonical shared libraries for this stack (coordinates fixed, versions in catalog): coroutines (`kotlinx-coroutines-core`), `kotlinx-datetime`, `kotlinx-serialization-json` (export/backup), Kotlin stdlib `Uuid` for IDs, Kermit (`co.touchlab:kermit`) for logging, SQLDelight (`app.cash.sqldelight`), Koin (`io.insert-koin`), SKIE (`co.touchlab.skie`), Turbine (`app.cash.turbine`). Don't introduce an alternative to one of these without a promoted lesson.
43. Secrets/config: nothing baked into source (GLOBAL rule 8). Local-only app has no network keys, but signing config, store credentials, and any future endpoint config are injected at build/run time and git-ignored.

## Naming & packages

44. **Package-by-feature inside every module** (`…domain.note`, `…data.note`, `…presentation.note`, then `…settings`, etc.), not package-by-type. A feature's domain/data/presentation pieces sit under the same feature package in their respective modules.
45. Conventions: repository interface `XRepository` in domain / `XRepositoryImpl` in data; use cases `VerbNounUseCase`; state `XUiState`; ViewModel `XViewModel`; Koin module `xModule`. SQLDelight files named per aggregate (`Note.sq`).
46. One public class per file in shared code; file name matches the type.

---

## Patterns we've settled

> Seeded with known KMP gotchas for this stack. `/fix` appends a positive rule +
> short example here on recurrence (`Seen: 2`) or explicit user correction.

- **StateFlow → SwiftUI must be main-dispatched.** Expose ViewModel state on `Dispatchers.Main`; never let a background-emitting flow reach SKIE's Swift `AsyncSequence`, or SwiftUI updates land off-main and glitch. _(Realizes rule 26.)_
- **iOS ViewModel teardown is manual.** Pair every shared `ViewModel` with a `dispose()` and call it from SwiftUI `.onDisappear`; Android relies on the framework, iOS does not. _(Realizes rule 23.)_
- **Map at the `:data` boundary, always.** Generated SQLDelight rows convert to domain models inside the repository impl — keep a `…Mappers.kt` per feature so the rule is mechanical, not ad-hoc. _(Realizes rule 16.)_
