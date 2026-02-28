# thanx-qa-claude-plugins

Claude Code plugin marketplace for the Thanx QA Chapter.

This is the shared repo for QA-specific commands, agents, and skills. Build and iterate on your local machine. When something is ready to share, contribute it here.

## Quick Start

```bash
# Install the QA plugin
claude plugin add github:thanx/thanx-qa-claude-plugins

# Use a command
/qa:test-suite-prd <notion_prd_url> <jira_url>
```

This plugin is designed to run alongside `thanx-claude-plugins`. Commands that already exist in the main plugin (like `/investigate`, `/review`, `/commit`) are not duplicated here.

## What's Included

### Commands

| Command | Description |
|---------|-------------|
| `/qa:test-suite-prd` | Generate a [DRAFT] Test Suite from a Notion PRD |

### Agents

_Add shared QA agents here as they mature._

### Skills

_Add shared QA knowledge bases here as they mature._

## Contributing

### Adding a Command

1. Create a new `.md` file in `plugins/qa/commands/`
2. Add YAML frontmatter at the top:

```yaml
---
description: Brief description of what the command does
---
```

3. Write the command body with numbered steps
4. Run `npm test` to validate structure
5. Open a PR

### Adding a Skill

1. Create a new directory in `plugins/qa/skills/`
2. Add a `SKILL.md` file inside it
3. Skills are knowledge bases that agents and commands reference

### Adding an Agent

1. Create a new `.md` file in `plugins/qa/agents/`
2. Add YAML frontmatter with `description` and `capabilities`
3. Agents are launched via `Task()` calls from commands

## Development

```bash
# Install dependencies
npm ci

# Run all tests
npm test

# Lint markdown files
npm run lint:md

# Check links
npm run test:links

# Validate plugin structure
npm run validate
```

## CI

CircleCI runs on every push:

- Jest tests (schema validation, markdown linting)
- Markdown lint check
- Link validation
- Plugin structure validation (via Claude CLI)

## Relationship with thanx-claude-plugins

This repo is for **QA-specific** workflows. The main [thanx-claude-plugins](https://github.com/thanx/thanx-claude-plugins) repo contains engineering-wide commands like `/investigate`, `/review`, `/commit`, etc. Both plugins can be installed simultaneously without conflict.
