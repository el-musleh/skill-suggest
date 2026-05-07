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
/skill-suggest                                        # scan project files only
/skill-suggest "fix WooCommerce checkout + add tests" # scan + focus on specific task
```

Both modes always fetch the online registry. The optional argument narrows the suggestions toward your specific task.

## What It Does

### Step 1: Scan the project

Run these commands to understand the codebase:

```bash
# File type fingerprint
find . -maxdepth 3 -name "*.json" -o -name "*.php" -o -name "*.ts" -o -name "*.tsx" \
  -o -name "*.py" -o -name "*.go" -o -name "*.rs" -o -name "*.java" \
  -o -name "Dockerfile" -o -name "docker-compose.yml" \
  -o -name "*.tf" -o -name "playwright.config.*" \
  -o -name "wp-config.php" -o -name "functions.php" 2>/dev/null | head -60

# Key config files
ls package.json composer.json Pipfile pyproject.toml go.mod Cargo.toml 2>/dev/null

# Check for CLAUDE.md Active Skills section
grep -A 30 "## Active Skills" CLAUDE.md .claude/CLAUDE.md 2>/dev/null || true
```

### Step 2: Read installed skills

```bash
for skill_dir in ~/.claude/skills/*/; do
  skill=$(basename "$skill_dir")
  desc=$(grep -m1 "^description:" "$skill_dir/SKILL.md" 2>/dev/null | sed 's/description: *//' | tr -d '"')
  echo "INSTALLED | $skill | $desc"
done
```

### Step 3: Fetch online registry (always)

Always fetch the official registry regardless of whether a task description was provided:

```
WebFetch: https://raw.githubusercontent.com/anthropics/claude-code-skills/main/README.md
```

Parse skill names + descriptions from the registry. Cross-reference against installed skills.

Label each result:
- `[installed]` — already in `~/.claude/skills/`
- `[not installed]` — available online, install with `npx skills add <name>`

### Step 4: Score and suggest

Score skills from both sources (installed + online registry) against the project fingerprint. If a task description was provided, weight matches toward it — otherwise weight purely on detected stack.

**Scoring signals:**
| Signal | Skills to consider |
|---|---|
| `wp-config.php` / `functions.php` | `wordpress-pro`, `php-pro`, `security-reviewer` |
| `playwright.config.*` | `playwright-pro` |
| `docker-compose.yml` | `docker-development`, `devops-engineer` |
| `*.tf` files | `terraform-engineer` |
| `*.ts` / `*.tsx` | `typescript-pro`, `react-expert`, `nextjs-developer` |
| `*.py` | `python-pro`, `fastapi-expert`, `django-expert` |
| German project / `.de` domain | `gdpr-dsgvo-expert` |
| task mentions "API" | `api-designer`, `api-design-reviewer` |
| task mentions "test" | `playwright-pro`, `tdd-guide`, `senior-qa` |
| task mentions "security" / "auth" | `security-reviewer`, `secure-code-guardian` |
| task mentions "database" / "SQL" | `postgres-pro`, `sql-pro`, `database-optimizer` |
| task mentions "deploy" / "CI" | `ci-cd-pipeline-builder`, `devops-engineer` |
| task mentions "performance" | `performance-profiler` |
| task mentions "migrate" | `migration-architect` |

### Step 5: Output

Present a ranked table — top 8 max:

```
## Suggested Skills for This Project

Based on: [detected stack] + "[user task]"

| # | Skill | Status | Why | Action |
|---|---|---|---|---|
| 1 | `wordpress-pro` | [installed] | wp-config.php detected | `/wordpress-pro` |
| 2 | `playwright-pro` | [installed] | playwright.config.js found | `/playwright-pro` |
| 3 | `security-reviewer` | [not installed] | pushing to live server | `npx skills add security-reviewer` |
...

## Add to CLAUDE.md?

To make these permanent for this project, add this to your CLAUDE.md:

## Active Skills
| Skill | When to use |
|---|---|
| `wordpress-pro` | Theme/plugin work |
...
```

### Step 6: Offer to update CLAUDE.md

Ask: "Want me to add an `## Active Skills` section to this project's CLAUDE.md with these suggestions?"

If yes — append the table to CLAUDE.md (or update the existing `## Active Skills` section if one already exists).
