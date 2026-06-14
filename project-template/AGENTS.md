## graphify

This project has a knowledge graph at graphify-out/ with god nodes, community structure, and cross-file relationships.

When the user types `/graphify`, invoke the `skill` tool with `skill: "graphify"` before doing anything else.

Rules:
- For codebase questions, first run `graphify query "<question>"` when graphify-out/graph.json exists. Use `graphify path "<A>" "<B>"` for relationships and `graphify explain "<concept>"` for focused concepts. These return a scoped subgraph, usually much smaller than GRAPH_REPORT.md or raw grep output.
- Dirty graphify-out/ files are expected after hooks or incremental updates; dirty graph files are not a reason to skip graphify. Only skip graphify if the task is about stale or incorrect graph output, or the user explicitly says not to use it.
- If graphify-out/wiki/index.md exists, use it for broad navigation instead of raw source browsing.
- Read graphify-out/GRAPH_REPORT.md only for broad architecture review or when query/path/explain do not surface enough context.
- After modifying code, run `graphify update .` to keep the graph current (AST-only, no API cost).

## ai-dev-system workflow

You are the **implementer**. The rules live at `.ai-dev-system/`. You work one
GitHub issue at a time, on its own branch, strictly to the rules below.

### Picking the issue
When the user says **"implement next issue"**, select the next open feature issue:
```bash
gh issue list --state open --label feature --json number,title --jq 'sort_by(.number) | .[0]'
```
Take the lowest open issue number (or the explicit `#N` the user names), then
follow the flow below. Don't start another issue until the current PR is merged.

### Before writing code (every task)
1. Read `.claude/conventions.md` for the stack and project overrides.
2. Open the active stack agent named there — e.g. `.ai-dev-system/agents/flutter.md` — and obey it. **Read its `Patterns we've settled` section first** (that's where past lessons became defaults).
3. Read `.ai-dev-system/lessons/flutter.md` (global, your stack) and `.claude/lessons.local.md` (this repo only).
4. Skim `.ai-dev-system/GLOBAL_RULES.md` for the non-negotiables.
5. Use graphify (above) to understand existing code before changing it.

### While implementing
- Work only the issue you're assigned, on branch `feature/issue-{N}-{slug}`. Stay scoped to that issue.
- Core hard rules (full list in GLOBAL_RULES.md): no secrets in source; typed failures, never a raw exception or stack trace to the user; HTTPS only; validate all input.

### Commit & PR
- Commit with `/caveman-commit` if present, else `type(scope): summary (#N)` plus a body line of what changed.
- Open the PR with base **dev** and end the body with `Closes #N` (closed automatically when the PR merges into `dev`). Never PR into main.
- The human reviews the PR; your GitHub Action auto-review posts a backstop on PR open. Don't merge — wait for the human.

### After code
- Add any bug found/fixed to `.claude/wiki/known-issues.md`; any decision to `.claude/wiki/decisions.md`.
- Append ONE line to `.claude/wiki/sessions/index.md` and write detail to a per-session file.
- Run `graphify update .`.
