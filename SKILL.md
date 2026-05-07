---
name: "skill-suggest"
description: "Scan the current project's files and a task description to suggest relevant skills — from both installed skills and the online registry. Run manually when starting a new project or task."
user-invocable: true
---

# /skill-suggest — Suggest Relevant Skills for This Project

Analyzes the current directory and your task description, then recommends skills from:
- **Installed skills** on this machine (`~/.claude/skills/`)
- **Online registry** (anthropics/claude-code-skills on GitHub)

## Usage

```
/skill-suggest           # Fast Mode: Scans project + installed skills (instant)
/skill-suggest online    # Online Mode: Also searches the official registry
/skill-suggest "task"    # Task Mode: Scans project + weights toward your task
```

## ⚠️ Critical Rules — Read Before Anything Else

1. **There is NO skills CLI.** No `claude skills`, no `npx skills install`, no `gemini skills`, no install command of any kind. Never run or suggest such commands. Skills are markdown files — managing them means reading/writing JSON files directly.
2. **"Local" means `.claude/settings.local.json`**, not copying files. Never copy skill directories.
3. **Step 4b is mandatory.** Always call `AskUserQuestion` with the exact questions defined in Step 4b before showing output. Never substitute your own questions.
4. **Never ask about "where to install"** — there is no install. There is only activating (writing `"on"` to `settings.local.json`) or trimming (writing `"name-only"`).

---

## Smart Orchestration & Early Exit

To optimize for speed and cost, the agent must follow this "Smart Flow" logic:

### 1. The "Enough" Heuristic
Before each step, ask: *"Do I already have 3+ high-confidence suggestions?"*
- **If Yes:** Skip further discovery (Step 3) and proceed directly to **Step 4b** (interactive review). Never skip Step 4b — it is mandatory.
- **If No:** Proceed to the next discovery step.

### 2. Complexity Budget
- **Tool Call Limit:** Maximum 3 discovery-related tool calls total.
- **Data Volume Limit:** If `find` or `grep` returns >100 lines, stop processing that stream and work with the first 50 results.
- **Token Saver:** Do not read more than the first 2KB of any config file.

### 3. Skip Conditions
- Skip **Step 3 (Online Fetch)** if the local stack is extremely common (e.g., standard React/Next.js) and `installed` skills already cover it.
- Skip **Deep Dependency Scan** in Step 1 if the project root is clearly labeled (e.g., `README.md` says "This is a simple Python script").

---

## What It Does

### Step 1: Scan the project

Run these commands to understand the codebase and current context:

```bash
# Context and Path detection
echo "CWD | $(pwd)"
echo "GIT_ROOT | $(git rev-parse --show-toplevel 2>/dev/null || pwd)"

# File type fingerprint
find . -maxdepth 3 -name "*.json" -o -name "*.php" -o -name "*.ts" -o -name "*.tsx" \
  -o -name "*.py" -o -name "*.go" -o -name "*.rs" -o -name "*.java" \
  -o -name "Dockerfile" -o -name "docker-compose.yml" \
  -o -name "*.tf" -o -name "playwright.config.*" \
  -o -name "wp-config.php" -o -name "functions.php" 2>/dev/null | head -60

# Key config files & Deep dependency check
ls package.json composer.json Pipfile pyproject.toml go.mod Cargo.toml 2>/dev/null
grep -E "\"dependencies\"|\"devDependencies\"" package.json -A 50 2>/dev/null | head -100

# Check for CLAUDE.md Active Skills section (local and root)
grep -h -A 30 "## Active Skills" CLAUDE.md .claude/CLAUDE.md "$(git rev-parse --show-toplevel 2>/dev/null)/CLAUDE.md" 2>/dev/null | sort -u || true
```

### Step 2: Read installed and local skills

```bash
# Read skill states from settings — works with both string and object forms.
# Falls back gracefully if python3 is absent (jq alternative) or file is missing.
_read_skill_states() {
  local file="$1" prefix="$2"
  [ -f "$file" ] || return
  if command -v python3 >/dev/null 2>&1; then
    python3 -c "
import json, sys
try:
    d = json.load(open('$file'))
    skills = d.get('skillOverrides', {})
    if not isinstance(skills, dict):
        raise ValueError('skillOverrides is not a dict')
    for name, cfg in skills.items():
        if isinstance(cfg, str):
            state = cfg
        elif isinstance(cfg, dict):
            state = cfg.get('state', 'on')
        else:
            state = 'unknown'
        print(f'$prefix | {name} | {state}')
except Exception as e:
    print(f'# WARN: could not parse $file: {e}', file=sys.stderr)
" 2>/dev/null
  elif command -v jq >/dev/null 2>&1; then
    jq -r '.skillOverrides // {} | to_entries[] | "$prefix | \(.key) | \(if (.value|type)=="string" then .value else (.value.state // "on") end)"' "$file" 2>/dev/null
  else
    echo "# WARN: neither python3 nor jq found — skill states unavailable" >&2
  fi
}

_read_skill_states ~/.claude/settings.json         SKILL_STATE
_read_skill_states .claude/settings.local.json     SKILL_STATE_LOCAL

# Build set of actually-installed skill names (global + project-local)
_installed_skills() {
  for d in ~/.claude/skills/*/; do [ -d "$d" ] && basename "$d"; done
  for d in .claude/skills/*/;   do [ -d "$d" ] && basename "$d"; done
}
INSTALLED_SET=$(_installed_skills)

# Reconcile: flag state entries whose skill directory no longer exists
# Output: ORPHAN | <name> | <state> | <source-file>
# These are excluded from scoring and surfaced as a cleanup notice.
_check_orphans() {
  local file="$1" label="$2"
  [ -f "$file" ] || return
  command -v python3 >/dev/null 2>&1 || return
  python3 -c "
import json, os, sys
installed = set('''$INSTALLED_SET'''.split())
try:
    d = json.load(open('$file'))
    for name in d.get('skillOverrides', {}):
        if name not in installed:
            print(f'ORPHAN | {name} | $label')
except Exception:
    pass
" 2>/dev/null
}

_check_orphans ~/.claude/settings.json         "~/.claude/settings.json"
_check_orphans .claude/settings.local.json     ".claude/settings.local.json"

# Global skills (skip skill-suggest itself)
for skill_dir in ~/.claude/skills/*/; do
  [ -d "$skill_dir" ] || continue
  skill=$(basename "$skill_dir")
  [ "$skill" = "skill-suggest" ] && continue
  desc=$(grep -m1 "^description:" "$skill_dir/SKILL.md" 2>/dev/null | sed 's/^description:[[:space:]]*//' | tr -d '"')
  echo "INSTALLED | $skill | $desc"
done

# Project-local skills (skip skill-suggest itself)
for skill_dir in .claude/skills/*/; do
  [ -d "$skill_dir" ] || continue
  skill=$(basename "$skill_dir")
  [ "$skill" = "skill-suggest" ] && continue
  desc=$(grep -m1 "^description:" "$skill_dir/SKILL.md" 2>/dev/null | sed 's/^description:[[:space:]]*//' | tr -d '"')
  echo "PROJECT-LOCAL | $skill | $desc"
done
```

### Step 3: Fetch online registry (Conditional)

**Skip this step entirely** — the official skills registry is not publicly available. Do not attempt any WebFetch for skill discovery. Work only from the installed skills found in Step 2.

### Step 4: Score and suggest

Before scoring, collect any `ORPHAN` lines from Step 2 into a separate list. **Exclude orphans from scoring entirely** — a deleted skill cannot be suggested or trimmed.

Then merge the `SKILL_STATE` and `SKILL_STATE_LOCAL` lines into a state map (local overrides global). Apply these rules:

- **`off`** — Do NOT exclude silently. Show in the table with a grayed note and action `enable with /skills`, so the user knows it exists but is disabled.
- **`on`** / **`user-only`** / **`name-only`** — Include normally; reflect the exact state in the State column.
- **Registry-only skills** (from Step 3, not installed) — State column shows `—`.
- If `SKILL_STATE` data is absent entirely (python3/jq unavailable), State column shows `?` for all installed skills with a footer note.

Score skills from all sources against the project fingerprint and **current directory path**. Never include `skill-suggest` itself as a suggestion.

**Scoring signals:**
| Signal | Skills to consider |
|---|---|
| path contains `api` / `backend` / `server` | `senior-backend`, `sql-pro`, `api-designer` |
| path contains `ui` / `frontend` / `web` | `senior-frontend`, `react-expert`, `vue-expert` |
| path contains `test` / `spec` | `playwright-pro`, `senior-qa`, `tdd-guide` |
| `package.json` contains `stripe` | `stripe-integration-expert` |
| `package.json` contains `tailwind` | `ui-design-system` |
| `wp-config.php` / `functions.php` | `wordpress-pro`, `php-pro`, `security-reviewer` |
| `playwright.config.*` | `playwright-pro` |
| `docker-compose.yml` | `docker-development`, `devops-engineer` |
| `*.tf` files | `terraform-engineer` |
| `*.ts` / `*.tsx` | `typescript-pro`, `react-expert`, `nextjs-developer` |
| `*.py` | `python-pro`, `fastapi-expert`, `django-expert` |
| task mentions "security" / "auth" | `security-reviewer`, `secure-code-guardian` |
| task mentions "database" / "SQL" | `postgres-pro`, `sql-pro`, `database-optimizer` |

### Step 4b: AskUserQuestion — MANDATORY

**Call `AskUserQuestion` now. Do not output text first. Use this exact call structure. Maximum 2 questions — never add a third question about CLAUDE.md, scope, or anything else:**

```
questions: [
  {
    header: "Activate for project",
    question: "Which skills do you want configured as active for this project?",
    multiSelect: true,
    options: [
      // TOP 4 scored skills only — pick highest 4 by score
      { label: "<skill-name>", description: "<why it fits this project>" },
      { label: "<skill-name>", description: "<why it fits this project>" },
      { label: "<skill-name>", description: "<why it fits this project>" },
      { label: "<skill-name>", description: "<why it fits this project>" }
    ]
  },
  // Include Question 2 ONLY if there are globally-on skills that scored below threshold.
  // Omit this entire block if none exist.
  {
    header: "Trim to name-only",
    question: "These globally-active skills don't match this project. Select any to demote to name-only here:",
    multiSelect: true,
    options: [
      // UP TO 4 globally-on skills that scored below threshold
      { label: "<skill-name>", description: "<why it doesn't fit, e.g. No PHP files found>" },
      ...
    ]
  }
]
```

**Do not ask about "where to install", "global vs local scope", or any other questions.** Those concepts do not apply here.

**After the user submits:**

1. Write Q1 selections to `.claude/settings.local.json` as `"on"`:
```bash
python3 - <<'PYEOF'
import json, os
path = ".claude/settings.local.json"
os.makedirs(".claude", exist_ok=True)
try:
    d = json.load(open(path))
except (FileNotFoundError, json.JSONDecodeError):
    d = {}
overrides = d.setdefault("skillOverrides", {})
confirmed = ["<name1>", "<name2>"]  # replace with actual Q1 selections
for name in confirmed:
    if name not in overrides:
        overrides[name] = "on"
json.dump(d, open(path, "w"), indent=2)
print(f"Saved {len(confirmed)} skill(s) to {path}")
PYEOF
```

**The write is REQUIRED even if the skills are already globally installed.** The purpose is to create an explicit project-level record in `settings.local.json` — not to install anything. "Already globally installed" is NOT a reason to skip this step.

2. Store Q2 selections as `TRIM_LIST` for Step 7. If Q2 was omitted or user selected nothing, `TRIM_LIST` is empty — skip Step 7.
3. Proceed directly to Step 5 output. No more questions.

### Step 5: Output

Present a ranked table for the skills confirmed in Step 4b Q1 — top 8 max. Clearly label `[project-local]` skills.

```
## Suggested Skills for This Project

Based on: [detected stack] in [current path] + "[user task]"

| # | Skill | Status | State | Why | Action |
|---|---|---|---|---|---|
| 1 | `typescript-pro` | [installed] | on → local | TS/TSX files found | confirmed in Step 4b — written to settings.local.json |
| 2 | `custom-tool` | [project-local] | user-only | Found in .claude/skills | `/custom-tool` |
| 3 | `docker-development` | [not installed] | — | docker-compose.yml found | `npx skills add docker-development` (installs globally) |
| 4 | `senior-backend` | [installed] | off | Strong backend signals | ⚠️ disabled — enable with `/skills` |
...

> **Note:** State `?` means skill state could not be read (python3 and jq both unavailable).

## Stale Entries (if any orphans found)

If the `ORPHAN` list from Step 2 is non-empty, append this section after the table:

```
## ⚠️ Stale skill entries detected

These skills have entries in settings but their directories no longer exist:
- `some-deleted-skill` (in ~/.claude/settings.json)
- `another-deleted-skill` (in .claude/settings.local.json)

Run this to clean them up:
  python3 -c "
import json
for path in ['$HOME/.claude/settings.json', '.claude/settings.local.json']:
    try:
        d = json.load(open(path))
        overrides = d.get('skillOverrides', {})
        stale = [k for k in overrides if not __import__('os').path.isdir(f\"{path.rsplit('/',2)[0]}/skills/{k}\")]
        [overrides.pop(k) for k in stale]
        json.dump(d, open(path, 'w'), indent=2)
        print(f'Cleaned {len(stale)} stale entries from {path}: {stale}')
    except FileNotFoundError:
        pass
  "
```
```

## Add to CLAUDE.md?

To make these permanent for this project or directory, add this to the relevant CLAUDE.md:

### Choice A: Root Project (./CLAUDE.md)
*Best for general skills used everywhere.*

### Choice B: This Directory (./subdir/CLAUDE.md)
*Best for scoped skills that only activate when working here.*

## Active Skills
| Skill | When to use |
|---|---|
| `api-designer` | Designing REST/GraphQL endpoints |
```

### Step 6: Offer to update CLAUDE.md

Ask: "Where should I add these `## Active Skills` suggestions?"
- [ ] Root CLAUDE.md (apply project-wide)
- [ ] Local CLAUDE.md (only for the current directory)
- [ ] No thanks

If a choice is made — append the table to the selected `CLAUDE.md` file (or update the existing `## Active Skills` section if one already exists). If the file doesn't exist, create it.

### Step 7: Apply scope trim

This step runs **only if** the user selected skills to trim in Step 4b Q2. Do **not** ask again — the user already confirmed their selection.

Show a brief summary of what will be written:
```
Setting name-only in .claude/settings.local.json:
- wordpress-pro    (no PHP/WP files found)
- playwright-pro   (no test config found)
- senior-devops    (no Docker/k8s/CI files found)
```

Then immediately apply the change using this exact shell snippet — **do not use any `claude` or `npx skills` CLI commands**, they do not support this operation:
   ```bash
   python3 - <<'EOF'
   import json, os
   path = ".claude/settings.local.json"
   os.makedirs(".claude", exist_ok=True)
   try:
       d = json.load(open(path))
   except (FileNotFoundError, json.JSONDecodeError):
       d = {}
   overrides = d.setdefault("skillOverrides", {})
   to_trim = {
       # populated at runtime from step above
       # "wordpress-pro": "name-only",
   }
   for name, state in to_trim.items():
       if name not in overrides:          # never overwrite existing entries
           overrides[name] = state
   json.dump(d, open(path, "w"), indent=2)
   print("Done.")
   EOF
   ```
   Substitute `to_trim` with the confirmed skill list before running.

**Safety rules — must all be respected:**
- **Never touch** skills already present in `.claude/settings.local.json` — existing project overrides are authoritative.
- **Only set `name-only`**, never `off` — skills remain discoverable by name.
- **Skip** any skill whose global state is already `name-only`, `off`, or `user-only` — nothing to do.
- **Always show the list** and wait for confirmation before writing — no silent modifications.
- **No invented CLI commands** — the only correct mechanism is reading/writing `.claude/settings.local.json` directly as shown above.
