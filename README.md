# skill-suggest

A [Claude Code](https://claude.ai/code) skill that scans your project and suggests the most relevant skills — pulling from both your **locally installed** skill library and the **official online registry**.

No more guessing which skills to activate. Point it at your project and it tells you exactly what to install or invoke.

---

## What It Does

1. **Scans your project** — detects file types, frameworks, and path context (e.g., `/api` vs `/ui`)
2. **Smart Orchestration** — uses an "Enough" heuristic to stop early if it finds clear matches, saving time and tokens
3. **Reads your installed skills** — checks `~/.claude/skills/` and project-local `.claude/skills/`
4. **Conditional Registry Fetch** — only hits the online registry if local results are sparse or if you use `online` mode
5. **Labels results clearly** — `[installed]`, `[project-local]`, vs `[not installed]`
6. **Directory-Scoped Activation** — offers to write `## Active Skills` to the root OR a local `CLAUDE.md` for folder-specific skill sets

---

## Install

```bash
npx skills add el-musleh/skill-suggest
```

Or manually:

```bash
mkdir -p ~/.agents/skills/skill-suggest
curl -o ~/.agents/skills/skill-suggest/SKILL.md \
  https://raw.githubusercontent.com/el-musleh/skill-suggest/main/SKILL.md
ln -s ~/.agents/skills/skill-suggest ~/.claude/skills/skill-suggest
```

Then add to your `~/.claude/settings.json`:

```json
{
  "skillOverrides": {
    "skill-suggest": "on"
  }
}
```

---

## Usage

```bash
/skill-suggest           # Fast Mode: Scans project + local skills (instant)
/skill-suggest online    # Online Mode: Also searches the official registry
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

## Example Output

```
## Suggested Skills for This Project

Based on: WordPress (wp-config.php, functions.php), Playwright (playwright.config.js)
+ task: "add Stripe payments and write Playwright tests"

| # | Skill            | Status          | Why                              | Action                              |
|---|------------------|-----------------|----------------------------------|-------------------------------------|
| 1 | wordpress-pro    | [installed]     | wp-config.php detected           | /wordpress-pro                      |
| 2 | playwright-pro   | [installed]     | playwright.config.js found       | /playwright-pro                     |
| 3 | stripe-integration-expert | [not installed] | task mentions Stripe    | npx skills add stripe-integration-expert |
| 4 | security-reviewer| [installed]     | payment integration = auth risk  | /security-reviewer                  |
| 5 | php-pro          | [installed]     | WordPress backend                | /php-pro                            |
| 6 | tdd-guide        | [not installed] | task mentions "tests"            | npx skills add tdd-guide            |

Want me to add these to your CLAUDE.md as an ## Active Skills table? (yes/no)
```

---

## The `## Active Skills` Pattern

This skill can write a persistent skill table into your project's `CLAUDE.md`. Claude reads `CLAUDE.md` automatically on every session start.

**New:** You can now save these to a subdirectory `CLAUDE.md` (e.g., `src/api/CLAUDE.md`). This ensures that specific skills (like `sql-pro`) only activate when you are working in that specific part of the codebase.

---

## Update

To get the latest improvements (Smart Orchestration, directory-scoped activation, etc.), run:

```bash
npx skills update skill-suggest
```

If you installed manually via `curl`, simply re-run the `curl` command from the [Install](#install) section.

---

## Scoring Signals

| Detected signal | Skills suggested |
|---|---|
| path contains `api`/`server` | `senior-backend`, `sql-pro`, `api-designer` |
| path contains `ui`/`frontend` | `senior-frontend`, `react-expert`, `vue-expert` |
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

---

## Contributing

PRs welcome — especially for new scoring signals or additional stack detection patterns.

---

## License

MIT
