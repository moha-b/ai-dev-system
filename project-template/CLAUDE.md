## graphify

This project has a knowledge graph at graphify-out/ with god nodes, community structure, and cross-file relationships.

Rules:
- For codebase questions, first run `graphify query "<question>"` when graphify-out/graph.json exists. Use `graphify path "<A>" "<B>"` for relationships and `graphify explain "<concept>"` for focused concepts. These return a scoped subgraph, usually much smaller than GRAPH_REPORT.md or raw grep output.
- If graphify-out/wiki/index.md exists, use it for broad navigation instead of raw source browsing.
- Read graphify-out/GRAPH_REPORT.md only for broad architecture review or when query/path/explain do not surface enough context.
- After modifying code, run `graphify update .` to keep the graph current (AST-only, no API cost).

## ai-dev-system workflow

You are the **planner** here (and the reviewer if I ask you to review locally).
The rules live at `.ai-dev-system/` (a link to the global ai-dev-system repo).
Read only what you need — don't reinvent.

### Before planning
1. Read `.claude/conventions.md` for this project's stack and any overrides.
2. Skim `.ai-dev-system/GLOBAL_RULES.md` for the non-negotiables.

### Planning a feature set
- Break the idea into small, single-purpose features. **Show me the list and wait for my OK before creating anything.**
- Then create **one GitHub issue per feature**. GitHub issues are the only backlog source of truth.
- If the `/caveman` command exists, use it to format each issue. Otherwise use:
  - Title: `[FEATURE] <name>`
  - Body: `## Description`, then `## Acceptance Criteria` (checkbox list), then `## Technical Notes`.
- Do **not** write feature code in this role. Plan, agree, file issues. Implementation happens in OpenCode.

### Branch & commit rules (and enforce these in any review you run)
- One branch per issue: `feature/issue-{N}-{slug}`. PRs target **dev**, never **main**.
- Commit titles end with `(#N)`. Use `/caveman-commit` if present, else `type(scope): summary (#N)` with a body line saying what changed; PR body ends with `Closes #N`.

### Reviewing locally (optional — CI does this automatically on PR open)
- Follow `.ai-dev-system/LEARNING_LOOP.md` exactly: load the issue + rules + lessons, review the diff, classify findings A/B/C/D, write lessons, and propose agent enhancements when a lesson recurs or I correct you.

### Session archive
- At session end, append ONE line to `.claude/wiki/sessions/index.md` and put detail in a per-session file.
- To recall past work, read the index only; open a single session file if needed — never scan them all.
