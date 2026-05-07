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

## Smart Orchestration & Early Exit

To optimize for speed and cost, the agent must follow this "Smart Flow" logic:

### 1. The "Enough" Heuristic
Before each step, ask: *"Do I already have 3+ high-confidence suggestions?"*
- **If Yes:** Skip further discovery (Step 3) and proceed directly to Output (Step 5).
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
# Global skills
for skill_dir in ~/.claude/skills/*/; do
  [ -d "$skill_dir" ] || continue
  skill=$(basename "$skill_dir")
  desc=$(grep -m1 "^description:" "$skill_dir/SKILL.md" 2>/dev/null | sed 's/description: *//' | tr -d '"')
  echo "INSTALLED | $skill | $desc"
done

# Project-local skills
for skill_dir in .claude/skills/*/; do
  [ -d "$skill_dir" ] || continue
  skill=$(basename "$skill_dir")
  desc=$(grep -m1 "^description:" "$skill_dir/SKILL.md" 2>/dev/null | sed 's/description: *//' | tr -d '"')
  echo "PROJECT-LOCAL | $skill | $desc"
done
```

### Step 3: Fetch online registry (Conditional)

To save time and tokens, **only** fetch the official registry if:
1. The user mentions keywords like `online`, `discover`, `registry`, or `all`.
2. OR, fewer than 3 relevant skills were found locally (installed or project-local).

When fetching, use a targeted prompt to minimize the returned data:

```
WebFetch: https://raw.githubusercontent.com/anthropics/claude-code-skills/main/README.md
Prompt: "Extract only the skill names and descriptions related to [detected stack]. Format as a simple list."
```

If these conditions aren't met, skip this step and use local results only. Label results as `[not installed]` if found here but not in Step 2.

### Step 4: Score and suggest

Score skills from all sources against the project fingerprint and **current directory path**.

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

### Step 5: Output

Present a ranked table — top 8 max. Clearly label `[project-local]` skills.

```
## Suggested Skills for This Project

Based on: [detected stack] in [current path] + "[user task]"

| # | Skill | Status | Why | Action |
|---|---|---|---|---|
| 1 | `api-designer` | [installed] | In /api directory | `/api-designer` |
| 2 | `custom-tool` | [project-local] | Found in .claude/skills | `/custom-tool` |
...

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
