# Development Guidelines

## Plugin Structure

```
plugins/qa/
  .claude-plugin/plugin.json   # Plugin manifest
  commands/*.md                # Slash commands
  agents/*.md                  # Agents (launched via Task() from commands)
  skills/*/SKILL.md            # Knowledge bases
```

## Writing Commands

- Every command file must start with YAML frontmatter containing a `description` field
- Use numbered steps (## Step 1, ## Step 2, etc.)
- Reference `$ARGUMENTS` to access user input
- Keep each step deterministic and measurable

## Writing Agents

- Frontmatter must include `description` and `capabilities`
- Define clear input/output contracts
- Include "When to Use This Agent" and "Process" sections

## Writing Skills

- One directory per skill, with a `SKILL.md` inside
- Skills are reference material, not executable workflows
- Keep content factual and up to date

## Testing

Run `npm test` before every PR. Tests validate:

- JSON schema correctness (`.mcp.json`, `plugin.json`, `marketplace.json`)
- YAML frontmatter presence on commands and agents
- Markdown structure (title heading, no trailing whitespace, no consecutive blank lines)
- Link validity (internal and external)

## Versioning

Update the version in `plugins/qa/.claude-plugin/plugin.json` when merging changes.
Follow semver: patch for fixes, minor for new commands/agents/skills, major for breaking changes.
