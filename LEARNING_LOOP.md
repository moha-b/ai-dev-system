# LEARNING_LOOP.md

The single spec for how a PR is reviewed and how the system learns from it. The
GitHub Action's prompt points here; a human running a manual review follows the
same steps. Tool-agnostic — identical under Claude Code or the Action runner.

---

## Phase 0 — Load context (cheap)
Never read the repo top to bottom.
1. `gh pr view {N}` and `gh pr diff {N}`.
2. `gh issue view {ISSUE}` — the acceptance criteria (from `Closes #N`).
3. `graphify query "<impacted area>"` — what the diff touches and its blast radius.
4. Read, in order: `GLOBAL_RULES.md`; the active `agents/<stack>.md` (stack from `.claude/conventions.md`); the global `lessons/<stack>.md`; the project `.claude/lessons.local.md`.

## Phase 1 — Review
Check the diff against, in priority order:
1. **Issue fit** — every acceptance criterion met, nothing extra (scope creep).
2. **GLOBAL_RULES.md** — secrets, branch target (`dev` not `main`), logging, error modeling, input validation.
3. **agents/<stack>.md** — the stack's rules and its "Patterns we've settled".
4. **lessons** — has a recorded lesson (stack or local) recurred?

Post findings as a PR review, grouped and specific (file:line). Never paste a
secret, PII, or stack trace into a comment.

## Phase 2 — Classify each finding (exactly one class)

| Class | Test | Lands in |
|---|---|---|
| **A. Defect** | wrong only in this PR; rules already cover it | PR comment only |
| **B. Repo lesson** | recurs in *this* repo, project-specific context | `.claude/lessons.local.md` |
| **C. Stack lesson** | any project on this stack benefits | `lessons/<stack>.md` (global) |
| **D. Cross-stack** | true regardless of stack | `GLOBAL_RULES.md` |

Decision aid: "remember this repo's table is named X" → B. "Flutter cubits must
not import widgets" → C. "Never log full response bodies" → D.

## Phase 3 — Write the lesson (always, same pass)
Append to the right file using this fixed, greppable, de-dupable block:

```
## [YYYY-MM-DD] {class B/C/D} | {one-line title}
- Trigger: what happened (PR #N, file)
- Rule: the corrected behavior, as an imperative
- Scope: repo | stack:<name> | global
- Seen: 1
- Agent: not-yet | PR #<id> to agents/<stack>.md
```

Before appending, grep the target file for a near-duplicate. If found, **do not
add a new entry** — increment its `Seen:` count. Crossing `Seen: 2` is the
recurrence trigger below.

Routing: B → `.claude/lessons.local.md`. C → promotion PR adding it to the global
`lessons/<stack>.md`. D → promotion PR adding it to `GLOBAL_RULES.md`. (C and D
are PRs against this repo; a human merges them.)

## Phase 4 — Enhance the agent (the learning loop)

**Activation rule — when the loop edits `agents/<stack>.md`:**
A first-time, single defect is logged only; the agent is left alone (no
overfitting to flukes). The loop proposes an agent edit when **either**:

1. a lesson reaches **`Seen: 2`** (it recurred), **or**
2. the **user explicitly corrected or overrode** Claude — a "do it this way"
   instruction, or a human commit that fixes Claude's code.

On activation, open a promotion PR that writes the corrected pattern into the
agent's **`## Patterns we've settled`** section as a positive, imperative rule
plus a ≤6-line example:

```
### <thing> — settled <YYYY-MM-DD> (from PR #N)
Do: <the right way, imperative>
Avoid: <the wrong way that was caught>
Example:
  <minimal snippet>
```

Set the lesson's `Agent:` field to the PR id. A human merges it. From then on,
the agent reads that section at session start (see the Session Protocol in
`agents/_TEMPLATE.md`), so re-requesting the same feature/bugfix produces the
right result the first time — modify or extend the settled pattern rather than
rebuild it wrong.

## Phase 5 — After merge to `dev`
`git checkout dev && git pull`, then `graphify` to refresh the graph. Don't batch.

---

## Session archive protocol (both roles)
At the **end** of every session, append ONE line to
`.claude/wiki/sessions/index.md`:

```
- [YYYY-MM-DD] {session-id} | {one-line summary} | issues: #N,#M | files: <top dirs>
```

Write the detail to its own file `.claude/wiki/sessions/{session-id}-{slug}.md`.

**Retrieval rule:** read `index.md` only. Open a specific session file solely when
you need that session's detail. **Never read all session files.** This is the
committed, greppable history; context-mode holds the live working memory.

## Hard rules
- Issue fit and GLOBAL_RULES outrank style nits.
- Lessons are written on the same pass as the review, never deferred.
- C/D promotions and every agent edit are PRs, never direct pushes. Human merges.
- The agent is touched only on recurrence or explicit user correction — never on a first-time fluke.
