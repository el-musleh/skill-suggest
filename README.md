# skill-suggest

A [Claude Code](https://claude.ai/code) skill that scans your project and suggests the most relevant skills — pulling from both your **locally installed** skill library and the **official online registry**.

No more guessing which skills to activate. Point it at your project and it tells you exactly what to install or invoke.

---

## What It Does

1. **Scans your project** — detects file types, frameworks, and config files (`wp-config.php`, `docker-compose.yml`, `playwright.config.*`, `*.tf`, `tsconfig.json`, etc.)
2. **Reads your installed skills** — checks `~/.claude/skills/` for everything already available
3. **Fetches the online registry** — always queries the [anthropics/claude-code-skills](https://github.com/anthropics/claude-code-skills) registry for the full skill list
4. **Scores and ranks** — cross-references detected stack + optional task description to produce a ranked top-8 list
5. **Labels results clearly** — `[installed]` vs `[not installed]` with ready-to-run install commands
6. **Offers to update CLAUDE.md** — can write an `## Active Skills` table directly into your project's `CLAUDE.md`

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

```
/skill-suggest
```
Scans the current directory and suggests skills based on detected stack. Also checks the online registry.

```
/skill-suggest "your task description"
```
Same scan + registry fetch, but weights suggestions toward your specific task.

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

This skill can write a persistent skill table into your project's `CLAUDE.md`. Claude reads `CLAUDE.md` automatically on every session start, so it knows which skills are relevant without you having to invoke them manually each time.

Example table it generates:

```markdown
## Active Skills

| Skill | When to use |
|---|---|
| `wordpress-pro` | Theme/plugin edits, WP-CLI tasks |
| `playwright-pro` | E2E test writing and debugging |
| `security-reviewer` | Before any push to live server |
| `php-pro` | Custom PHP in themes or functions.php |
```

Once this is in your `CLAUDE.md`, Claude will proactively suggest these skills when the context matches — no manual invocation needed.

---

## Scoring Signals

| Detected signal | Skills suggested |
|---|---|
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
