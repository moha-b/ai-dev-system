# ai-dev-system

A **global, stack-agnostic** AI development system. One repo you own, adopted by
every project. It turns a plain-language idea into GitHub issues, implements each
on its own branch under a stack-specific rule set, auto-reviews every PR against
those rules and the issue, and **feeds what review learns back into the rules and
the agent itself** — so the same mistake never recurs and the same request comes
out right the first time on the next project.

## One idea

The rules live on disk. Three roles take turns reading them:

- **Claude Code plans** — you chat, agree the features, it opens one GitHub issue per feature.
- **OpenCode implements** — one branch per issue, code to the stack agent's rules, `/caveman-commit … (#N)`, PR into `dev`.
- **Claude reviews** — the GitHub Action fires on PR open, checks the diff against the issue + rules + lessons, comments, and learns.

Both tools read the same files, which is *why* swapping the implementer can't drift the structure — nothing lives in one tool's head.

## Adopt in a new project (quick steps)

Once-per-computer setup (clone this repo, set `AIDEV_HOME`, install context-mode)
is in `SETUP.md` Part 1. After that, each new project is just:

1. Copy the templates in: `.claude/` folder, `CLAUDE.md`, `AGENTS.md`, and `.github/workflows/claude-review.yml`.
2. Link the rules: create a `.ai-dev-system` junction/symlink to `AIDEV_HOME`, and add it to `.gitignore`.
3. Pick your stack in `.claude/conventions.md` (e.g. `Active agent: agents/flutter.md`).
4. Turn on graphify: `graphify opencode install && graphify hook install`.
5. Make a `dev` branch and protect `main` in GitHub settings.
6. In `claude-review.yml`, set `repository:` to your rules repo as `owner/repo`, and add a `CLAUDE_CODE_OAUTH_TOKEN` (or `ANTHROPIC_API_KEY`) secret.

Then: **plan** in Claude Code (`/plan`), **build** in OpenCode ("work issue #N"),
and the **review** runs itself on the PR. Full commands (Mac + Windows) and a
troubleshooting list are in `SETUP.md`.

## What's in this repo (global)

| File / dir | Purpose |
|---|---|
| `GLOBAL_RULES.md` | Cross-stack non-negotiables every agent obeys |
| `agents/<stack>.md` | Full, self-enhancing per-stack rule set (e.g. `flutter.md`) |
| `lessons/<stack>.md` | **Stack-scoped lessons, reused by every project of that stack** |
| `LEARNING_LOOP.md` | How review classifies findings, writes lessons, and enhances the agent |
| `workflows/claude-review.yml` | Drop-in GitHub Action: auto-review on PR open |
| `project-template/.claude/` | Copied into each repo (conventions, local lessons, session archive) |

## Lessons are stack-scoped, not project-scoped

A Flutter lesson is useless to a Kotlin project and vice versa. So lessons live
**globally, keyed by stack**:

- `lessons/flutter.md` — every Flutter project reads this at session start. New Flutter projects inherit all prior Flutter lessons for free.
- `.claude/lessons.local.md` (in each repo) — only the quirks that are true for *that one repo*.

Cross-stack truths don't go in lessons at all — they go in `GLOBAL_RULES.md`.

## The learning loop (self-enhancing agent)

Review classifies each finding and routes it (full spec in `LEARNING_LOOP.md`):

| Class | Means | Lands in |
|---|---|---|
| A — defect | wrong only in this PR | PR comment, author fixes |
| B — repo lesson | recurs in *this* repo | `.claude/lessons.local.md` |
| C — stack lesson | true for any project on this stack | `lessons/<stack>.md` (global) |
| D — cross-stack | true everywhere | `GLOBAL_RULES.md` |

**When the loop enhances the agent (my decision):** a single first-time defect is
only logged — agents are not touched, to avoid overfitting to a fluke. The loop
edits `agents/<stack>.md` when **either** of these is true:

1. a lesson **recurs** (recorded a second time), or
2. the **user explicitly corrects or overrides** Claude's output ("no, do it this way", or a human bugfix commit on top of Claude's code).

On that trigger it opens a promotion PR that writes the corrected pattern into the
agent's "Patterns we've settled" section — a positive rule plus a short example —
so the next time that feature or bug comes up, the agent builds it right the first
time instead of repeating the miss. **You merge the PR** (human-gated), so one
noisy repo can't rewrite global behavior.

## Session history is archived, read on demand

Each project keeps `.claude/wiki/sessions/`: one greppable line per session in
`index.md`, full detail in a per-session file. The rule both tools follow: **read
the index only; open a specific session file solely when you need its detail —
never scan them all.** This is the durable, committed, human-readable history.
context-mode handles the *live* in-flight memory separately (see below).

## Token tools (both active)

- **graphify** — query the code graph instead of reading/grepping files.
- **context-mode** — sandboxes tool output (raw data stays out of context) and restores working state across compaction. It's the live working memory; `wiki/sessions/` is the committed archive. They're complementary.
