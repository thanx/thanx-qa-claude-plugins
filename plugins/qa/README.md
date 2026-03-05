# QA Plugin

Claude Code plugin for Thanx QA Chapter workflows.

## Commands

| Command | Description |
|---------|-------------|
| `/qa:check-new-prds` | Check for new PRDs in Technical Discovery and offer to kick them off |
| `/qa:recheck-prds` | Re-run adoption review and suite reviewer for PRDs that moved to In Progress |
| `/qa:qa-kickoff` | Run the full QA kickoff pipeline for a Notion PRD |
| `/qa:qa-brief` | Generate a QA Brief from a Notion PRD |
| `/qa:adoption-review` | Review a Notion PRD for adoption criteria completeness |
| `/qa:suite-reviewer` | Review an existing Test Suite Notion page |
| `/qa:test-suite-prd` | Generate a [DRAFT] Test Suite from a Notion PRD |
| `/qa:qa-status-update` | Post an updated QA status to one or all active project channels |
| `/qa:release-readiness` | Full release readiness check for a project — sweeps Notion, Jira, and Slack |
| `/qa:escaped-defect` | Analyze production bugs from a given period and classify their root cause |

## Agents

Internal agents launched via `Task()` from commands. Not user-invocable directly.

| Agent | Description |
|-------|-------------|
| `adoption-review` | Headless adoption criteria analysis — returns JSON |
| `test-suite-generator` | Headless test suite generation — returns JSON |
| `suite-reviewer` | Reviews test suite coverage and BDD quality, removes [DRAFT] from Notion title — returns JSON |
| `scorecard-updater` | Headless scorecard field resolver — returns Notion property updates |

## Adding New Commands

1. Create a `.md` file in `commands/`
2. Add YAML frontmatter with a `description` field
3. Follow the numbered step pattern (see existing commands)
4. Run `npm test` to validate

## Adding New Skills

1. Create a directory in `skills/` with a `SKILL.md` file
2. Skills are knowledge bases that agents and commands can reference

## Adding New Agents

1. Create a `.md` file in `agents/`
2. Add YAML frontmatter with `description` and `capabilities`
3. Agents are launched via `Task()` calls from commands
