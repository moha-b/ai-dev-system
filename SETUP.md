# Setup

Do **Part 1** once on your computer. Do **Parts 2–4** once per project.

Before you start, make sure you have: `git`, the GitHub CLI (`gh`, logged in via
`gh auth login`), and graphify.

---

## Part 1 — Install once (per computer)

### Step 1. Download this repo
**Mac**
```bash
git clone https://github.com/<your-org>/ai-dev-system.git ~/ai-dev-system
```
**Windows (PowerShell)**
```powershell
git clone https://github.com/<your-org>/ai-dev-system.git $HOME\ai-dev-system
```

### Step 2. Remember where it lives (set AIDEV_HOME)
**Mac**
```bash
echo 'export AIDEV_HOME="$HOME/ai-dev-system"' >> ~/.zshrc
source ~/.zshrc
```
**Windows (PowerShell)**
```powershell
setx AIDEV_HOME "$env:USERPROFILE\ai-dev-system"
```
`setx` only affects **new** terminals — close and reopen it. To use it in the
*current* window too, also run: `$env:AIDEV_HOME = "$env:USERPROFILE\ai-dev-system"`.
Verify with `echo $env:AIDEV_HOME` (should print the path).

### Step 3. Turn on context-mode (token saver)
Inside **Claude Code**:
```
/plugin marketplace add mksglu/context-mode
/plugin install context-mode@context-mode
```
For **OpenCode**, in `opencode.json`:
```json
{ "$schema": "https://opencode.ai/config.json", "plugin": ["context-mode"] }
```

Done with the one-time install.

---

## Part 2 — Start a project (per project)

Run from inside your project folder.

### Step 1. Copy in the rules folders + memory files
**Mac**
```bash
cp -R "$AIDEV_HOME/project-template/.claude" .
cp "$AIDEV_HOME/project-template/CLAUDE.md" .
cp "$AIDEV_HOME/project-template/AGENTS.md" .
mkdir -p .github/workflows
cp "$AIDEV_HOME/workflows/claude-review.yml" .github/workflows/
```
**Windows (PowerShell)**
```powershell
Copy-Item "$env:AIDEV_HOME\project-template\.claude" -Destination . -Recurse
Copy-Item "$env:AIDEV_HOME\project-template\CLAUDE.md" .
Copy-Item "$env:AIDEV_HOME\project-template\AGENTS.md" .
New-Item -ItemType Directory -Force .github\workflows
Copy-Item "$env:AIDEV_HOME\workflows\claude-review.yml" .github\workflows\
```
> If you already have a graphify-generated `CLAUDE.md`/`AGENTS.md`, keep its
> graphify block and paste the "ai-dev-system workflow" section under it instead
> of overwriting.

### Step 2. Link the global rules into the project
This is what lets the tools (and you, locally) read the rules as `.ai-dev-system/…`.

**Mac**
```bash
ln -s "$AIDEV_HOME" .ai-dev-system
echo ".ai-dev-system" >> .gitignore
```
**Windows (PowerShell)** — use `New-Item`, NOT `cmd /c mklink` (PowerShell mangles the quotes):
```powershell
New-Item -ItemType Junction -Path .ai-dev-system -Target $env:AIDEV_HOME
Add-Content .gitignore ".ai-dev-system"
```
Verify (should print **True**): `Test-Path .ai-dev-system/GLOBAL_RULES.md`

### Step 3. Pick your stack
In `.claude/conventions.md`, set:
```
Active agent: agents/flutter.md
```
Change `flutter` to your stack if needed.

### Step 4. Turn on graphify
```bash
graphify opencode install
graphify hook install
```
(On Windows, write `graphify .` — never `/graphify .`)

### Step 5. Make the dev branch
```bash
git checkout -b dev
git push -u origin dev
```
In GitHub → Settings → Branches: protect `main`, require PRs into `dev`.

---

## Part 3 — Turn on auto-review (per project)

1. Open `.github/workflows/claude-review.yml`.
2. Set `repository:` to your rules repo in **owner/repo** form — e.g. `moha-b/ai-dev-system`. **Not a URL.**
3. Easiest: make that `ai-dev-system` repo **public** (it's only rules) so no token is needed.
4. Add ONE secret in GitHub → Settings → Secrets and variables → Actions:
   - `CLAUDE_CODE_OAUTH_TOKEN` (from your Claude subscription), **or** `ANTHROPIC_API_KEY`.
   Make sure the matching line in the workflow's `with:` block is the uncommented one.

Now every PR into `dev` is reviewed automatically.

---

## Part 4 — Use it day to day
1. **Plan** — Claude Code, `/plan`, describe a feature. It makes GitHub issues.
2. **Build** — OpenCode, "work issue #N". One issue, one branch, one PR into `dev`.
3. **Review** — fires automatically on the PR.
4. **Repeat.** When `dev` is solid, open one PR `dev → main`.

---

## Troubleshooting (the errors we already hit, and the fix)

**`gh issue view 1` → 404 / "issue doesn't exist" but it's on GitHub.**
`gh` reads the repo from your git remote. Run `gh issue list`, `gh repo view`,
`git remote -v`. If the remote is wrong: `git remote set-url origin
https://github.com/<you>/<repo>.git`, or `gh repo set-default <you>/<repo>`.
Also note issues and PRs **share numbering** — your first issue may be #2.

**Action: `Invalid repository '...'. Expected format {owner}/{repo}'`.**
The `repository:` field has a URL. Use `owner/repo` only (no `https://`, no `.git`).

**Action: `Retrieving the default branch name … Not Found`.**
Add `ref: main` to the rules checkout step (it's already in the template now), and
make the rules repo public OR remove any `token:` line that points at a secret you
didn't create.

**Action: `Could not fetch an OIDC token … add id-token: write`.**
The workflow `permissions:` block needs `id-token: write` (now in the template).

**Windows: `The syntax of the command is incorrect` when linking.**
Don't use `cmd /c mklink "$env:AIDEV_HOME"`. Use
`New-Item -ItemType Junction -Path .ai-dev-system -Target $env:AIDEV_HOME`, and
confirm `echo $env:AIDEV_HOME` isn't empty (reopen the terminal after `setx`).
