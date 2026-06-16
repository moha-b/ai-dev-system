# lessons/kmp.md — Kotlin Multiplatform (stack-scoped)

> Read at session start by every project on the `kmp` stack. New KMP projects
> inherit all of these for free. Per-repo quirks go in that repo's
> `.claude/lessons.local.md`; cross-stack truths go in `GLOBAL_RULES.md`, not here.
> Format: one lesson per entry, with a `Seen:` counter. A lesson promoted into
> `agents/kmp.md` § Patterns we've settled stays here for provenance.

---

### L1 — Off-main StateFlow corrupts SwiftUI
**Seen: 1** · Class C
A `StateFlow` collected by SwiftUI through SKIE that emits on a background dispatcher causes off-main UI updates (flicker, stale frames, occasional crash). Always expose presentation state on `Dispatchers.Main`.
→ Promoted to `agents/kmp.md` rule 26 / Patterns.

### L2 — Leaked viewModelScope on iOS
**Seen: 1** · Class C
Kotlin `ViewModel` instances created from SwiftUI are not auto-cleared the way Android clears them; without an explicit teardown the `viewModelScope` and its collectors leak across screen pushes. Provide `dispose()` and call it from `.onDisappear`.
→ Promoted to `agents/kmp.md` rule 23 / Patterns.

### L3 — SQLDelight types leaking upward
**Seen: 1** · Class C
Returning generated SQLDelight row types from a repository couples `:domain`/`:presentation` to the database and breaks the Clean boundary. Map to domain models inside the repository impl every time.
→ Promoted to `agents/kmp.md` rule 16 / Patterns.

### L4 — `com.android.library` vs the Android-KMP plugin
**Seen: 1** · Class C
On AGP 9+, configuring a KMP module's Android target with the legacy `com.android.library` plugin relies on deprecated, opt-in APIs slated for removal in AGP 10. Use the official `com.android.kotlin.multiplatform.library` plugin for the shared module's Android target.

### L5 — `expect`/`actual` used where DI belongs
**Seen: 1** · Class C
Reaching for `expect`/`actual` singletons to supply platform services (drivers, storage, clock) scatters platform coupling and resists testing. Inject platform primitives through Koin from `androidMain`/`iosMain` instead; reserve `expect`/`actual` for thin platform-primitive wrappers only.
