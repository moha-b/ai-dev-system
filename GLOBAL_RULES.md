# GLOBAL_RULES.md — Non-Negotiables (every stack, every project)

> Every `agents/<stack>.md` references this file. A project's
> `.claude/conventions.md` may tighten these, never loosen them. Class-D lessons
> (true everywhere) are promoted into this file by reviewed PR.

## Workflow & branches
1. GitHub issues are the backlog source of truth. No work without an issue.
2. Never push to `main`. Feature branches target **`dev`**; `dev` promotes to `main` only at a stable release.
3. One feature per branch, one branch per issue: `feature/issue-{N}-{slug}`.
4. Every commit title ends with `(#N)`. Every PR body ends with `Closes #N`.
5. Use `/caveman-commit`. Stay strictly scoped to the issue.
6. Run `graphify` after every merge to `dev` — don't batch.

## Roles (fixed)
7. Planning is done in **Claude Code** (`/plan` → issues only). Implementation **and** the automated PR review are done by **OpenCode** (review fires on PR open as a safety net). Final review is **the user**. Lessons are captured by the user via **`/fix`** in Claude plan mode. All read the same on-disk rules and lessons.

## Security (hard stops)
8. No hardcoded secrets — no keys, tokens, or credential-bearing URLs in source. Inject config at build/run time; keep env files out of version control.
9. Sensitive data uses platform-secure storage, never plaintext.
10. HTTPS only; treat TLS failures as hard errors.
11. Never log tokens, credentials, PII, or full bodies that may contain them. Strip/gate verbose logs in release.

## Error handling (hard stops)
12. No raw exception or stack trace ever reaches the user — map to a typed, human-readable result with a retry path.
13. Model failures as values across layer boundaries, not thrown exceptions bubbling through the app.
14. There is always an explicit error state, and the consumer always handles it.

## Hygiene
15. No leftover debug code, unlinked TODOs, or commented-out blocks in a PR.
16. New behavior ships with a test or a stated reason it can't.
17. Validate and sanitize all external input (user input, deep links, webhooks).

## Review & learning contract
18. Every PR is reviewed — by the user, and automatically by OpenCode on PR open — against the issue's acceptance criteria, this file, the active `agents/<stack>.md`, the global `lessons/<stack>.md`, and the project `lessons.local.md`.
19. Findings are classified and routed per `LEARNING_LOOP.md`. The user enhances the agent by running `/fix` (Claude plan mode), which opens a human-merged PR against this repo on recurrence or explicit correction.
