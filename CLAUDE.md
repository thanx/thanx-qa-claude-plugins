# Development Guidelines

## Plugin Structure

```text
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

## Verification Commands

Run before every PR, in this order:

```bash
# 1. Markdown lint
npm run lint:md

# 2. Link validation (internal and external)
npm run test:links

# 3. Schema + frontmatter validation
npm test

# 4. Full plugin validation
npm run validate
```

All four must pass. Fix before pushing.

### Common Failures

| Error | Cause | Fix |
|-------|-------|-----|
| Missing frontmatter | New .md file without YAML header | Add `---\ndescription: ...\n---` |
| Trailing whitespace | Markdown lint rule MD009 | Run `npm run lint:md:fix` |
| Broken link | Reference to moved/deleted file | Update the link target |
| Schema error | Invalid JSON in plugin.json | Check JSON syntax |

## Testing

Run `npm test` before every PR. Tests validate:

- JSON schema correctness (`.mcp.json`, `plugin.json`, `marketplace.json`)
- YAML frontmatter presence on commands and agents
- Markdown structure (title heading, no trailing whitespace, no consecutive blank lines)
- Link validity (internal and external)

## Configuration

Hardcoded identifiers used across commands.
When team members or infrastructure changes, update this section and the corresponding command files.

### Slack User IDs

| Name | Slack User ID | Used in |
|------|---------------|---------|
| Beatriz | `U08UEQ22H7W` | `qa-kickoff.md` (Step 9), `test-suite-prd.md` (Step 7) |
| Giovani | `U08UEQ5MJDA` | `test-suite-prd.md` (Step 7) |
| Lial | `U08HTSEURPH` | `qa-kickoff.md` (Step 9) |

### Notion Database IDs

| Database | ID | Used in |
|----------|-----|---------|
| Projects database | `collection://0d7ef002-875f-453b-bb05-7789a3436086` | `check-new-prds.md` (Step 2), `recheck-prds.md` (Step 2) |

### Jira IDs

| Identifier | Value | Used in |
|------------|-------|---------|
| Jira Cloud ID | `7d5d6532-069d-419b-bd1c-d8321b134435` | `test-suite-prd.md` (Step 2) |
| Test Suite link field | `customfield_12289` | `test-suite-prd.md` (Step 6), `qa-kickoff.md` (Step 8a) |
| TQA project key | `TQA` | `qa-kickoff.md` (Step 8b) |

### Notion PRD Field Names

| Field | Property Name | Format | Used in |
|-------|---------------|--------|---------|
| Jira initiative link | `"JIRA"` | URL string | `qa-kickoff.md` (Step 8a) |
| Slack channel URL | `"Slack #channel"` | `https://thanx.slack.com/archives/{channel_id}` | `qa-kickoff.md` (Step 9) |
| Slack channel checkbox | `"Channel in slack with QA"` | checkbox | `scorecard-updater` agent |

### Environment Variables

| Variable | Purpose | Used in |
|----------|---------|---------|
| `QA_SLACK_WEBHOOK_URL` | Incoming webhook for QA Slack channel notifications | `test-suite-prd.md` (Step 7) |

**Update protocol:** When any value changes, update this table first,
then update each file listed in the "Used in" column.
Run `grep` across `plugins/` for the old value to catch missed references.

## Error Correction Log

When Claude makes a repeated mistake in this repo, add it here.

| Date | Mistake | Correction |
|------|---------|------------|
| - | - | - |

## Versioning

Update the version in `plugins/qa/.claude-plugin/plugin.json` when merging changes.
Follow semver: patch for fixes, minor for new commands/agents/skills, major for breaking changes.
