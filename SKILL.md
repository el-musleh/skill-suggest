---
name: "skill-suggest"
description: "Scan the current project's files and task description to suggest relevant skills, auto-demote unused ones to name-only, and offer to search online registries (skills.sh, clawhub.ai). Run at the start of each project."
user-invocable: true
---

# /skill-suggest — Suggest Relevant Skills for This Project

Analyzes the current directory and your task description, then recommends skills from:
- **Installed skills** on this machine (`~/.claude/skills/`)
- **Online registries** (skills.sh, clawhub.ai) — offered interactively each run

## Usage

```
/skill-suggest           # Scan project + suggest skills (always asks about online check)
/skill-suggest "task"    # Weight suggestions toward your specific task
```

## ⚠️ Critical Rules — Read Before Anything Else

1. **The skills CLI is `npx skills`.** The correct install command is `npx skills add <owner/repo>` (e.g. `npx skills add vercel-labs/agent-skills`). Commands that do NOT exist: `claude skills install`, `npx skills install`, `gemini skills`. Never invent subcommands — use only: `add`, `remove`, `list`, `find`, `update`, `init`.
2. **"Local" means `.claude/settings.local.json`**, not copying files. Never copy skill directories.
3. **Step 4b is mandatory.** Always call `AskUserQuestion` with the exact questions defined in Step 4b before showing output. Never substitute your own questions.
4. **Q0 covers the online check and install scope together** — never add a separate question for install location. The only install paths are `npx skills add` (global) or writing SKILL.md to `.claude/skills/` (local), triggered from Q0.

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
- Skip **Step 3 (Online Fetch)** if the user chose "No thanks" in Q0, or if the local stack is extremely common (e.g., standard React/Next.js) and `installed` skills already cover it.
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

Run this step only if the user selected "Yes — install globally" or "Yes — install locally" in Q0. If they chose "No thanks", skip entirely.

Two registries to search:
- **https://skills.sh/** — browse or search for community skills by tag/language
- **https://clawhub.ai/** — search for Claude Code skills by name or stack

Use `WebFetch` to retrieve results from these URLs. Extract skill names and descriptions relevant to the detected project stack. Add any matches as `registry` entries in the scoring pool — label them `[not installed]` in the output table with the action `npx skills add <owner/repo>` where `owner/repo` is taken **directly from the registry page result**. If the registry page does not provide a repo slug, show the registry URL instead and do not guess.

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

**Before calling AskUserQuestion, classify every installed skill (excluding `skill-suggest`) into one of four buckets:**

- **ACTIVATE_CANDIDATES** — top 4 highest-scoring skills (shown in Q1)
- **EXPLICIT_TRIM** — globally-on skills that scored below threshold (shown in Q2, up to 4)
- **BORDERLINE** — installed skills with a non-zero score but not in top 4; globally-on or `name-only`; not already in ACTIVATE_CANDIDATES or EXPLICIT_TRIM (shown in Q3, up to 4)
- **AUTO_NAME_ONLY** — all remaining installed skills with score = 0 that are currently globally `on` and have no entry in `.claude/settings.local.json`; these are applied silently in Step 7 without asking. Skills already globally `name-only`, `off`, or `user-only` are excluded — nothing to change.

**Before calling AskUserQuestion, check: does `.claude/` exist in the project root?**
- If **no**: create it automatically with `mkdir -p .claude` — do NOT ask the user to run `/init` first.

**Call `AskUserQuestion` now. Do not output text first. Use this exact call structure. Maximum 4 questions — never add a 5th:**

```
questions: [
  // Question 0 is ALWAYS shown — it asks about online check and, if yes, install scope in one shot.
  {
    header: "Online check",
    question: "Do you want to search online registries (skills.sh, clawhub.ai) for additional skills?",
    multiSelect: false,
    options: [
      {
        label: "Yes — install globally",
        description: "Fetch online registries, then install any picks with npx skills add (available in all projects)"
      },
      {
        label: "Yes — install locally",
        description: "Fetch online registries, then download SKILL.md into .claude/skills/ (this project only)"
      },
      {
        label: "No thanks",
        description: "Skip online check and work from locally installed skills only"
      }
    ]
  },
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
  },
  // Include Question 3 ONLY if BORDERLINE bucket is non-empty.
  // Omit this entire block if no borderline skills exist.
  {
    header: "Keep active?",
    question: "These skills have some relevance but aren't a strong match. Select any to keep active — unselected ones will be set to name-only:",
    multiSelect: true,
    options: [
      // UP TO 4 BORDERLINE skills — show partial-match reason
      { label: "<skill-name>", description: "<partial match signal, e.g. Dockerfile found but no k8s>" },
      ...
    ]
  }
]
```

**Do not add any questions beyond these four (Q0–Q3).** The maximum is strictly 4.

**After the user submits:**

0. Based on Q0 response, set state before continuing:
   - **"Yes — install globally"**: set `ONLINE_MODE=true`, `INSTALL_SCOPE=global`. Run Step 3 now, then for each skill the user picks from the online results: run `npx skills add <owner/repo>`. Re-read installed skills so newly installed ones appear in Q1.
   - **"Yes — install locally"**: set `ONLINE_MODE=true`, `INSTALL_SCOPE=local`. Run Step 3 now, then for each pick: use `WebFetch` to download the raw SKILL.md and write it to `.claude/skills/<name>/SKILL.md`. Re-read installed skills.
   - **"No thanks"**: set `ONLINE_MODE=false`. Skip Step 3 entirely and proceed.

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

2. Store Q2 selections as `TRIM_LIST` for Step 7.
3. From Q3: skills the user **did NOT select** are added to `TRIM_LIST`. Skills the user **did select** are written as `"on"` using the same python3 snippet in step 1 above (extend the `confirmed` list).
4. `AUTO_NAME_ONLY` bucket is always added to `TRIM_LIST` regardless of user interaction.
5. If `TRIM_LIST` is empty (Q2 omitted or nothing selected, Q3 omitted or all kept, and AUTO_NAME_ONLY empty), skip Step 7.
6. Proceed directly to Step 5 output. No more questions.

### Step 5: Output

Present a ranked table for the skills confirmed in Step 4b Q1 — top 8 max. Clearly label `[project-local]` skills.

```
## Suggested Skills for This Project

Based on: [detected stack] in [current path] + "[user task]"

| # | Skill | Status | State | Why | Action |
|---|---|---|---|---|---|
| 1 | `typescript-pro` | [installed] | on → local | TS/TSX files found | confirmed in Step 4b — written to settings.local.json |
| 2 | `custom-tool` | [project-local] | user-only | Found in .claude/skills | `/custom-tool` |
| 3 | `docker-development` | [not installed] | — | docker-compose.yml found | `npx skills add <owner/repo>` (from registry) |
| 4 | `senior-backend` | [installed] | off | Strong backend signals | ⚠️ disabled — enable with `/skills` |
...

> **Note:** State `?` means skill state could not be read (python3 and jq both unavailable).

## Stale Entries (if any orphans found)

If the `ORPHAN` list from Step 2 is non-empty, append this section after the table:

---
**⚠️ Stale skill entries detected**

These skills have entries in settings but their directories no longer exist:
- `some-deleted-skill` (in ~/.claude/settings.json)
- `another-deleted-skill` (in .claude/settings.local.json)

Run this to clean them up:
```bash
python3 - <<'PYEOF'
import json, os
for path in [os.path.expanduser('~/.claude/settings.json'), '.claude/settings.local.json']:
    try:
        d = json.load(open(path))
        overrides = d.get('skillOverrides', {})
        skills_root = os.path.join(os.path.dirname(path), 'skills')
        stale = [k for k in list(overrides) if not os.path.isdir(os.path.join(skills_root, k))]
        for k in stale:
            del overrides[k]
        json.dump(d, open(path, 'w'), indent=2)
        print(f'Cleaned {len(stale)} stale entries from {path}: {stale}')
    except FileNotFoundError:
        pass
PYEOF
```
---

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

This step runs **only if** `TRIM_LIST` is non-empty. `TRIM_LIST` is populated from three sources:
- **Q2 selections** — explicitly trimmed by the user
- **Q3 non-selections** — borderline skills the user chose not to keep active
- **AUTO_NAME_ONLY bucket** — score-0 skills with no existing project-level override (applied silently, no user confirmation needed)

Show a brief summary before writing, grouping by source:
```
Setting name-only in .claude/settings.local.json:

  Explicitly trimmed (Q2):
  - wordpress-pro    (no PHP/WP files found)
  - playwright-pro   (no test config found)

  Borderline — not kept (Q3):
  - senior-devops    (Dockerfile found but no k8s/CI signals)

  Auto name-only (no project signals):
  - php-pro
  - nextjs-developer
  - react-expert
```

Then immediately apply the change — **do not use any `claude` or `npx skills` CLI commands**, they do not support this operation:
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
   to_trim = [
       # populated at runtime — full TRIM_LIST from Q2 + Q3 non-selections + AUTO_NAME_ONLY
       # "wordpress-pro",
       # "playwright-pro",
   ]
   for name in to_trim:
       if name not in overrides:          # never overwrite existing project entries
           overrides[name] = "name-only"
   json.dump(d, open(path, "w"), indent=2)
   print(f"Set {len(to_trim)} skill(s) to name-only.")
   EOF
   ```
   Substitute `to_trim` with the full `TRIM_LIST` before running.

**Safety rules — must all be respected:**
- **Never touch** skills already present in `.claude/settings.local.json` — existing project overrides are authoritative. The python3 snippet's `if name not in overrides` guard enforces this.
- **Only write `name-only`**, never `off` — skills must remain discoverable by name.
- **AUTO_NAME_ONLY bucket already excludes** skills whose global state is `name-only`, `off`, or `user-only` (see bucket definition above) — the python3 guard is a secondary safety net, not the primary filter.
- **AUTO_NAME_ONLY writes are silent** — do not prompt again; zero score is sufficient signal.
- **Always show the grouped summary** before the python3 write runs — it serves as a receipt for all three trim sources.
- **No invented CLI commands** — the only correct mechanism is reading/writing `.claude/settings.local.json` directly as shown above.
