# lessons/ — stack-scoped, reused across projects

One file per stack (`flutter.md`, `kotlin.md`, `node-api.md`, …). Every project
of that stack reads its file at session start, so a new project inherits all
prior lessons for that stack for free. A Kotlin project never reads Flutter
lessons and vice versa.

- **Class C** lessons (true for any project on the stack) land here via a
  human-merged promotion PR.
- **Class B** (one repo only) stays in that repo's `.claude/lessons.local.md`.
- **Class D** (true everywhere) goes in `GLOBAL_RULES.md`, not here.

Entry format and the recurrence/agent-enhancement rules live in `LEARNING_LOOP.md`.
When a lesson here reaches `Seen: 2` (or the user explicitly corrects Claude), the
loop also writes the fix into `agents/<stack>.md` so it stops being a lesson and
becomes default behavior.
