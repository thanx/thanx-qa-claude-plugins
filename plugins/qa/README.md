# QA Plugin

Claude Code plugin for Thanx QA Chapter workflows.

## Commands

| Command | Description |
|---------|-------------|
| `/qa:test-suite-prd` | Generate a [DRAFT] Test Suite from a Notion PRD |

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
