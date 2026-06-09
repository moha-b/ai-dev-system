# Setup

Do **Part 1** once on your computer. Do **Part 2 + 3** once per project. That's it.

Before you start, make sure you have: `git`, the GitHub CLI (`gh`), and graphify
(you already installed it).

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
Then **close and reopen** the terminal so the variable takes effect.

### Step 3. Turn on context-mode (the token saver)

Inside **Claude Code**, type:
```
/plugin marketplace add mksglu/context-mode
/plugin install context-mode@context-mode
```
For **OpenCode**, open `opencode.json` and add this line:
```json
{ "$schema": "https://opencode.ai/config.json", "plugin": ["context-mode"] }
```

You're done with the one-time install.

---

## Part 2 — Start a project (per project)

Run these from inside your project folder (example: `clima`).

### Step 1. Copy in the rules and folders

**Mac**
```bash
cp -R "$AIDEV_HOME/project-template/.claude" .
mkdir -p .github/workflows
cp "$AIDEV_HOME/workflows/claude-review.yml" .github/workflows/
```

**Windows (PowerShell)**
```powershell
Copy-Item "$env:AIDEV_HOME\project-template\.claude" -Destination . -Recurse
New-Item -ItemType Directory -Force .github\workflows
Copy-Item "$env:AIDEV_HOME\workflows\claude-review.yml" .github\workflows\
```

### Step 2. Pick your stack
Open `.claude/conventions.md` and set the line:
```
Active agent: agents/flutter.md
```
Change `flutter` to your stack if it's not Flutter.

### Step 3. Turn on graphify for this project

**Mac & Windows (same)**
```bash
graphify opencode install
graphify hook install
```
(On Windows, always write `graphify .` — never `/graphify .`)

### Step 4. Make the dev branch

**Mac & Windows (same)**
```bash
git checkout -b dev
git push -u origin dev
```
In GitHub -> repo Settings -> Branches: protect `main`, and require pull requests
into `dev`.

---

## Part 3 — Turn on auto-review (per project)

1. Open `.github/workflows/claude-review.yml`.
2. Replace `<your-org>/ai-dev-system` with your real repo path.
3. In GitHub -> repo **Settings -> Secrets and variables -> Actions -> New secret**,
   add **one** of these:
   - `CLAUDE_CODE_OAUTH_TOKEN` — from your Claude subscription (no extra cost), **or**
   - `ANTHROPIC_API_KEY` — pay per review.

Now every pull request into `dev` gets reviewed automatically.

---

## How you use it day to day

1. **Plan** — in Claude Code, describe a feature. It makes GitHub issues.
2. **Build** — in OpenCode, work one issue on its own branch, commit, open a PR into `dev`.
3. **Review** — Claude reviews the PR by itself and comments.
4. **Repeat.** When `dev` is solid, open one PR from `dev` into `main`.

The system learns as you go: repeated mistakes get written into the rules so they
don't happen again.
