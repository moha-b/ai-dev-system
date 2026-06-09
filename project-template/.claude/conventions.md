# conventions.md — project stack & overrides (WINS over agents/<stack>.md)

## Stack
- Active agent: **agents/flutter.md**   <!-- swift-ios / kotlin-android / node-api / react-web / … -->
- Global rules repo (AIDEV_HOME): set by setup; review CI checks it out to `.ai-dev-system/`
- Backlog source of truth: **GitHub issues**

## Branch model
- PRs target **dev**; `main` receives `dev` only at stable releases
- Branch naming: `feature/issue-{N}-{slug}`

## Project-specific overrides
> Only where this project deviates from agents/<stack>.md or GLOBAL_RULES.md.
> May tighten, never loosen GLOBAL_RULES.
- (none yet)

## Project facts the assistant must not re-derive
- API base: <inject via build config — never hardcode>
- Key integrations: <…>
