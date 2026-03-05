---
description: End-of-session compounding improvement loop. Capture learnings, patch commands/agents/skills, and leave the QA plugin better for the next session.
---

# Session Improvement Loop

Capture what this session taught, patch what can be improved, and leave the
QA plugin measurably better for the next session.

**Philosophy:** Small bets, high frequency. Each /qa:improve run compounds. Do not batch improvements - apply them now.

## Step 1: Session Audit

Review the full conversation and identify:

1. **Commands used** - Which `/qa:` commands, agents, or workflows were invoked?
2. **What worked** - Fast paths, good outputs, useful patterns
3. **Friction points** - Where did the session slow down, require retries, or produce wrong output?
4. **Missing capabilities** - What did the user ask for that required manual work instead of a command?
5. **Incorrect assumptions** - What did the agent get wrong about PRDs, test suites, or QA workflows?
6. **Technical discoveries** - Notion PRD patterns, BDD edge cases, Jira field behaviors, agent pipeline quirks

Present findings as a structured table:

```text
| Category | Finding | Action |
|----------|---------|--------|
| Friction | Had to re-explain X context | Add to CLAUDE.md contextual rules |
| Missing  | No command for Y workflow   | Propose new /qa:Y command |
| Pattern  | Z technique worked well     | Capture in skill file |
```

## Step 2: Classify Improvements

Sort each proposed improvement by target:

| Target | File(s) | When to Apply |
|--------|---------|---------------|
| Plugin commands | `plugins/qa/commands/*.md` | New or improved `/qa:` commands |
| Plugin agents | `plugins/qa/agents/*.md` | New or improved agents |
| Plugin skills | `plugins/qa/skills/` | Knowledge bases for QA patterns, BDD conventions, automation tagging |
| CLAUDE.md rules | `CLAUDE.md` | New workflow patterns, guardrails, or contextual triggers |
| Plugin README | `plugins/qa/README.md` | Updated command table or documentation |
| Memory files | `~/.claude/projects/.../memory/` | Debugging insights, platform patterns, stable conventions |

## Step 3: Propose Changes

For each improvement, show the exact diff:

```text
### Target: <file_path>
**Why:** <one sentence explaining the improvement>
**Type:** New | Patch | Restructure

[Show the before/after or addition as a diff block]
```

**Rules:**

- Prefer small, targeted patches over large rewrites
- If the same file keeps getting patched, flag it for restructuring
- Every proposed change must include: what changes, why it helps future sessions
- New commands must follow the plugin structure (YAML frontmatter with description, numbered steps)
- New agents must include frontmatter with description and capabilities

## Step 4: Apply (With Approval)

Present all proposed changes as a numbered list. Wait for explicit approval before applying.

Format:

```text
## Proposed Improvements (N changes)

1. [plugins/qa/commands/test-suite-prd.md] Add handling for X edge case
2. [plugins/qa/skills/bdd-conventions/SKILL.md] New skill: BDD writing conventions
3. [CLAUDE.md] Add contextual rule for Y workflow
4. [plugins/qa/README.md] Update command table

Apply all? Or specify numbers to apply (e.g., "1,2,4"):
```

After approval, apply using Edit tool. Preserve existing content structure. Show confirmation for each applied change.

## Step 5: New Command Detection

Check if any repeated pattern from this session (or across recent sessions) should become a new `/qa:` command.

**Threshold test - all must be true:**

- [ ] The pattern is repeatable (not a one-time operation)
- [ ] It's non-trivial (saves >2 minutes per invocation)
- [ ] It's self-contained (can run without extensive setup context)
- [ ] It doesn't duplicate an existing command (`/qa:adoption-review`, `/qa:qa-kickoff`, `/qa:test-suite-prd`, `/qa:suite-reviewer`, `/qa:qa-brief`, `/qa:check-new-prds`, `/qa:recheck-prds`)
  or a main plugin command (`/investigate`, `/review`, etc.)

If a new command passes the threshold, draft the `plugins/qa/commands/<name>.md` file and include it in the proposed changes.

## Step 6: Knowledge Capture

Write durable knowledge to the appropriate location:

- **BDD conventions** - Gherkin patterns, scenario structures, common anti-patterns found during suite generation
- **PRD analysis patterns** - What makes a good PRD for test generation, common gaps in requirements
- **Automation tagging** - Effective tagging strategies, tag category evolution, cross-suite consistency
- **Notion PRD patterns** - Page structures, property names, status flows that affect QA commands
- **Agent pipeline insights** - Performance patterns, failure modes, input/output optimizations
- **Jira QA fields** - Custom field behaviors, transition rules, integration edge cases

Knowledge should be concise. One pattern per entry. Include the date and source.

## Step 7: Session Summary

Output a brief summary:

```text
## /qa:improve Summary

**Session focus:** [What this session accomplished]
**Changes proposed:** N
**Changes applied:** N
**New commands proposed:** N
**Knowledge captured:** N entries

**Compounding impact:** [One sentence on how these changes make future sessions better]
```

---

## Important Notes

- This command reads the FULL conversation history to extract learnings
- Changes to plugin files follow the existing structure (frontmatter, numbered steps, agent capabilities)
- New commands are created in `plugins/qa/commands/` directory
- New agents are created in `plugins/qa/agents/` directory
- All external communications remain gated (draft-only) per autonomy guardrails
- If past improvements aren't helping, flag them for revert - don't patch forever
- After applying changes, remember to bump the version in `plugins/qa/.claude-plugin/plugin.json`
