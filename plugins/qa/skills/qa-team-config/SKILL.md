---
description: QA team roster, channel IDs, Jira config, Notion database IDs, and custom field mappings. Reference this skill instead of hardcoding values in commands.
last_verified: "2026-03-04"
---

# QA Team Configuration

This is the **single source of truth** for QA team data. All commands and agents should reference this skill
instead of hardcoding values.

## Team Roster

| Name | Slack ID | Role | GitHub |
|------|----------|------|--------|
| Beatriz Souza | `U08UEQ22H7W` | QA Lead | `Beatriz-Thanx` |
| Giovani Silva | `U08UEQ5MJDA` | QA Lead | `giovani-thanx` |

**Engineering Manager:** Juliano Coimbra (`U06MDTUHKTP`)

**Team Slack IDs (for filtering):** `U08UEQ22H7W`, `U08UEQ5MJDA`

## Slack Channels

| Channel | ID | Type |
|---------|-----|------|
| `#rnd-qa-engs-internal` | `C08UMERQG5D` | Internal — QA team discussions |
| `#qa-bug-trends` | `C0AHA7768J0` | Bug signal alerts — used by `/analyze-app-reviews` and Cerebro |

**Webhook:** `QA_SLACK_WEBHOOK_URL` env var (used for bot notifications to QA channel)

## Notion Configuration

| Database | ID | Used in |
|----------|-----|---------|
| Projects database | `collection://0d7ef002-875f-453b-bb05-7789a3436086` | `check-new-prds.md`, `recheck-prds.md` |
| QA PRD Scorecard | `2efa84ed-4024-8015-87de-da0ca1cfa07b` | `scorecard-updater.md`, `qa-kickoff.md` |

> **Note:** QA PRD Scorecard is a view built on the Projects (PRD) database, not a standalone database.

## Jira Configuration

| Identifier | Value | Used in |
|------------|-------|---------|
| Jira Cloud ID | `7d5d6532-069d-419b-bd1c-d8321b134435` | `test-suite-prd.md` |
| TQA project key | `TQA` | `test-suite-prd.md`, `qa-kickoff.md` |
| Test Suite link field | `customfield_12289` | `test-suite-prd.md`, `qa-kickoff.md` |

## Key Repos

| Repository | Purpose |
|------------|---------|
| `thanx-qa-claude-plugins` | QA plugin commands, agents, and skills |
| `thanx-web-ordering-tests` | Web ordering E2E test suite |

## QA Kickoff Pipeline

The `/qa:qa-kickoff` command orchestrates these agents in sequence:

1. QA Brief generation (inline)
2. Adoption Review (`adoption-review` agent)
3. Test Suite generation (`test-suite-generator` agent)
4. Suite Review (`suite-reviewer` agent)
5. Scorecard Update (`scorecard-updater` agent)

Each agent is headless (JSON input/output) except `suite-reviewer` which has Notion MCP access.
