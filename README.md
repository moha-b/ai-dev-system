# ai-dev-system

A **global, stack-agnostic** AI development system. One repo you own, adopted by
every project. It turns a plain-language idea into GitHub issues, implements each
on its own branch under a stack-specific rule set, runs an automated PR review as a
safety net, and **feeds what you correct back into the rules and the agent itself**
— so the same mistake never recurs and the same request comes out right the first
time on the next project.

## One idea

The rules live on disk. The roles take turns reading them:

- **Claude Code plans** — you chat, agree the features, it opens one GitHub issue per feature. That's all it does: no code, no review.
- **OpenCode implements** — say "implement next issue"; it grabs the next open issue, codes to the stack agent's rules, `/caveman-commit … (#N)`, PR into `dev`.
- **You review** — the human is the real reviewer. **OpenCode auto-reviews** each PR on open as a backstop (`/opencode` re-triggers it).
- **`/fix` learns** — when you spot a problem, "fix X" in Claude plan mode captures it as a durable lesson (PR to this repo) so it never recurs on this stack.

Both tools read the same files, which is *why* swapping the implementer can't drift the structure — nothing lives in one tool's head.

## Adopt in a new project (quick steps)

Once-per-computer setup (clone this repo, set `AIDEV_HOME`, install context-mode)
is in `SETUP.md` Part 1. After that, each new project is just:

1. Copy the templates in: `.claude/` folder, `CLAUDE.md`, `AGENTS.md`, and both workflows (`.github/workflows/opencode-review.yml`, `.github/workflows/close-issue-on-merge.yml`).
2. Link the rules: create a `.ai-dev-system` junction/symlink to `AIDEV_HOME`, and add it to `.gitignore`.
3. Pick your stack in `.claude/conventions.md` (e.g. `Active agent: agents/flutter.md`).
4. Turn on graphify: `graphify opencode install && graphify hook install`.
5. Make a `dev` branch and protect `main` in GitHub settings.
6. Install the OpenCode GitHub app (`opencode github install`), set `repository:` in `opencode-review.yml` to your rules repo as `owner/repo`, and add an `ANTHROPIC_API_KEY` secret.

Then: **plan** in Claude Code (`/plan`), **build** in OpenCode ("implement next
issue"), **you review** the PR (OpenCode auto-reviews as a backstop), and merging
into `dev` auto-closes the issue. Full commands (Mac + Windows) and a
troubleshooting list are in `SETUP.md`.

## What's in this repo (global)

| File / dir | Purpose |
|---|---|
| `GLOBAL_RULES.md` | Cross-stack non-negotiables every agent obeys |
| `agents/<stack>.md` | Full, self-enhancing per-stack rule set (e.g. `flutter.md`) |
| `lessons/<stack>.md` | **Stack-scoped lessons, reused by every project of that stack** |
| `LEARNING_LOOP.md` | How findings are classified, lessons written, and the agent enhanced (via `/fix`) |
| `workflows/opencode-review.yml` | Drop-in GitHub Action: OpenCode auto-review on PR open (+ `/opencode` re-trigger) |
| `workflows/close-issue-on-merge.yml` | Drop-in GitHub Action: closes `Closes #N` issues when a PR merges into `dev` |
| `project-template/.claude/` | Copied into each repo (conventions, local lessons, session archive) |

## Lessons are stack-scoped, not project-scoped

A Flutter lesson is useless to a Kotlin project and vice versa. So lessons live
**globally, keyed by stack**:

- `lessons/flutter.md` — every Flutter project reads this at session start. New Flutter projects inherit all prior Flutter lessons for free.
- `.claude/lessons.local.md` (in each repo) — only the quirks that are true for *that one repo*.

Cross-stack truths don't go in lessons at all — they go in `GLOBAL_RULES.md`.

## The learning loop (self-enhancing agent)

You drive the loop with **`/fix`** (in Claude plan mode) when you spot a problem;
OpenCode's auto-review also flags candidates in a `PROMOTE` block. Each finding is
classified and routed (full spec in `LEARNING_LOOP.md`):

| Class | Means | Lands in |
|---|---|---|
| A — defect | wrong only in this PR | fix the PR; no lesson |
| B — repo lesson | recurs in *this* repo | `.claude/lessons.local.md` |
| C — stack lesson | true for any project on this stack | `lessons/<stack>.md` (global) |
| D — cross-stack | true everywhere | `GLOBAL_RULES.md` |

**When the loop enhances the agent:** a single first-time defect is only logged —
agents are not touched, to avoid overfitting to a fluke. `/fix` edits
`agents/<stack>.md` when **either**:

1. a lesson **recurs** (recorded a second time, `Seen: 2`), or
2. you **explicitly correct** the output ("no, do it this way" — exactly what `/fix` is for).

On that trigger `/fix` opens a promotion PR that writes the corrected pattern into
the agent's "Patterns we've settled" section — a positive rule plus a short example
— so the next time that feature or bug comes up, the agent builds it right the
first time on this project *and any future project on the same stack*. **You merge
the PR** (human-gated), so one noisy repo can't rewrite global behavior.

## Session history is archived, read on demand

Each project keeps `.claude/wiki/sessions/`: one greppable line per session in
`index.md`, full detail in a per-session file. The rule both tools follow: **read
the index only; open a specific session file solely when you need its detail —
never scan them all.** This is the durable, committed, human-readable history.
context-mode handles the *live* in-flight memory separately (see below).

## Token tools (both active)

- **graphify** — query the code graph instead of reading/grepping files.
- **context-mode** — sandboxes tool output (raw data stays out of context) and restores working state across compaction. It's the live working memory; `wiki/sessions/` is the committed archive. They're complementary.
