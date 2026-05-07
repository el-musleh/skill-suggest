# skill-suggest

A [Claude Code](https://claude.ai/code) skill that scans your project and suggests the most relevant skills — pulling from both your **locally installed** skill library and online registries ([skills.sh](https://skills.sh/), [clawhub.ai](https://clawhub.ai/)).

No more guessing which skills to activate. Point it at your project and it tells you exactly which ones to promote — and automatically demotes the rest to `name-only` to keep your context clean.

---

## What It Does

1. **Triggers Project Initialization** — automatically runs `/init` if the `.claude/` folder is missing to ensure a consistent project base.
2. **Prioritizes Online Discovery** — always offers to search online registries (skills.sh, clawhub.ai) for the latest tools.
3. **Local-First Installation** — installs new skills directly into `.claude/skills/` for maximum portability (or references global ones to avoid duplication).
4. **Portability & Collaboration** — records all active skills in `settings.local.json`, making it easy for contributors to sync missing skills.
5. **Scans your project** — detects file types, frameworks, and path context.
6. **Smart Orchestration** — uses an "Enough" heuristic to stop early if it finds clear matches, saving time and tokens.
7. **Reads your installed skills** — checks `~/.claude/skills/` and project-local `.claude/skills/`.
8. **Classifies every skill into a bucket** — then asks targeted questions before writing anything.
9. **Auto demotes unused skills** — score-0 skills are silently set to `name-only` in the project's `settings.local.json`.

---

## Recommended Setup

**Install all skills you might ever use, then set them globally to `name-only`:**

```json
// ~/.claude/settings.json
{
  "skillOverrides": {
    "python-pro": "name-only",
    "typescript-pro": "name-only",
    "wordpress-pro": "name-only",
    "senior-backend": "name-only"
    // ... all your skills
  }
}
```

Then run `/skill-suggest` at the start of each project. It will:
- Promote the right skills to `"on"` in `.claude/settings.local.json`
- Set irrelevant ones to `name-only` at the project level
- Ask you about borderline cases

This keeps your global context minimal while giving each project exactly the skills it needs.

---

## Install

```bash
npx skills add el-musleh/skill-suggest
```

Or manually:

```bash
mkdir -p ~/.claude/skills/skill-suggest
curl -o ~/.claude/skills/skill-suggest/SKILL.md \
  https://raw.githubusercontent.com/el-musleh/skill-suggest/main/SKILL.md
```

Then enable it in `~/.claude/settings.json`:

```json
{
  "skillOverrides": {
    "skill-suggest": "on"
  }
}
```

## Update

```bash
npx skills update skill-suggest
```

---

## Usage

```bash
/skill-suggest           # Fast Mode: Scans project + local skills (instant)
/skill-suggest online    # Online Mode: Also searches skills.sh and clawhub.ai
/skill-suggest "task"    # Task Mode: Scans project + weights toward your task
```

### Examples

```
/skill-suggest
/skill-suggest "add Stripe payments and write Playwright tests"
/skill-suggest "fix the WooCommerce checkout bug"
/skill-suggest "set up CI/CD pipeline and Docker deployment"
/skill-suggest "build a REST API with JWT auth and rate limiting"
```

---

## How Skills Are Classified

Before asking any questions, every installed skill is sorted into one of four buckets:

| Bucket | Criteria | What happens |
|---|---|---|
| **ACTIVATE_CANDIDATES** | Top 4 by score | Q1 — user picks which to set `on` |
| **EXPLICIT_TRIM** | Below threshold, globally `on` | Q2 — user picks which to demote |
| **BORDERLINE** | Non-zero score, not top 4 | Q3 — user picks which to keep; rest become `name-only` |
| **AUTO_NAME_ONLY** | Score = 0, globally `on`, no project override | Silent — written to `settings.local.json` without prompting |

Skills already at `name-only`, `off`, or `user-only` globally are never touched.

---

## Example Output

```
## Suggested Skills for This Project

Based on: WordPress (wp-config.php, functions.php), Playwright (playwright.config.js)
+ task: "add Stripe payments and write Playwright tests"

| # | Skill            | Status      | State        | Why                             | Action                          |
|---|------------------|-------------|--------------|---------------------------------|---------------------------------|
| 1 | wordpress-pro    | [installed] | on → local   | wp-config.php detected          | confirmed — written to local    |
| 2 | playwright-pro   | [installed] | on → local   | playwright.config.js found      | confirmed — written to local    |
| 3 | security-reviewer| [installed] | on → local   | payment integration = auth risk | confirmed — written to local    |
| 4 | php-pro          | [installed] | name-only    | WordPress backend               | promoted by user in Q3          |

Setting name-only in .claude/settings.local.json:

  Auto name-only (no project signals):
  - nextjs-developer
  - react-expert
  - terraform-engineer
```

---

## The `## Active Skills` Pattern

This skill can write a persistent skill table into your project's `CLAUDE.md`. Claude reads `CLAUDE.md` automatically on every session start.

You can save these to a subdirectory `CLAUDE.md` (e.g., `src/api/CLAUDE.md`). This ensures that specific skills (like `sql-pro`) only activate when you are working in that part of the codebase.

---

## Scoring Signals

| Detected signal | Skills suggested |
|---|---|
| path contains `api`/`server` | `senior-backend`, `sql-pro`, `api-designer` |
| path contains `ui`/`frontend` | `senior-frontend`, `react-expert`, `vue-expert` |
| path contains `test`/`spec` | `playwright-pro`, `senior-qa`, `tdd-guide` |
| `package.json` has `stripe` | `stripe-integration-expert` |
| `package.json` has `tailwind` | `ui-design-system` |
| `wp-config.php` / `functions.php` | `wordpress-pro`, `php-pro`, `security-reviewer` |
| `playwright.config.*` | `playwright-pro` |
| `docker-compose.yml` | `docker-development`, `devops-engineer` |
| `*.tf` files | `terraform-engineer` |
| `*.ts` / `*.tsx` | `typescript-pro`, `react-expert`, `nextjs-developer` |
| `*.py` | `python-pro`, `fastapi-expert`, `django-expert` |
| `.de` domain / German project | `gdpr-dsgvo-expert` |
| task: "API" | `api-designer`, `api-design-reviewer` |
| task: "test" | `playwright-pro`, `tdd-guide`, `senior-qa` |
| task: "security" / "auth" | `security-reviewer`, `secure-code-guardian` |
| task: "database" / "SQL" | `postgres-pro`, `sql-pro` |
| task: "deploy" / "CI" | `ci-cd-pipeline-builder`, `devops-engineer` |
| task: "performance" | `performance-profiler` |
| task: "migrate" | `migration-architect` |

---

## Requirements

- [Claude Code](https://claude.ai/code) CLI installed
- Skills system enabled (comes with Claude Code by default)
- `python3` or `jq` available (for reading skill states — both are standard on Linux/macOS)

---

## Contributing

PRs welcome — especially for new scoring signals or additional stack detection patterns.

---

## License

MIT
