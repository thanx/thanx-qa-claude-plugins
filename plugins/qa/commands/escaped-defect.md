---
description: Analyze bugs that reached production in a given period. Sweeps the BUGS Jira project, #triage-critical-bug, and #qa-bug-trends to identify escaped defects, classify their root cause, and log them to the bug impact record.
---

# Escaped Defect Analysis

Identify bugs that reached production in a given time window. For each one, determine
which project it belongs to, why it escaped (scope miss, coverage gap, or implementation
slip), and whether there was something QA could have caught earlier.

Run this at the end of your working day, or on Monday morning to cover the weekend.

## Usage

```bash
/qa:escaped-defect [days]
```

**Optional:**

- `days` — how many days back to look (default: 1; on Monday Claude auto-uses 3 to cover Friday, Saturday, Sunday)

---

You are executing the `/qa:escaped-defect` command.

## Step 1: Parse Input and Calculate Date Range

Arguments: $ARGUMENTS

Parse the first argument as `days` (integer). Default: `1`.

**Monday auto-detect:** If today is Monday, set `days = 3` unless the user explicitly provided a value.

Calculate:

- `from_date` = today minus `days` days, formatted as `YYYY-MM-DD`
- `to_date` = today, formatted as `YYYY-MM-DD`

---

## Step 2: Fetch Bugs from the BUGS Jira Project

Query the BUGS Jira project for issues created in the period:

- JQL: `project = BUGS AND created >= "{from_date}" AND created <= "{to_date}" ORDER BY priority ASC, created DESC`
- Jira Cloud ID: `7d5d6532-069d-419b-bd1c-d8321b134435`

> **Config note:** BUGS project key and Jira Cloud ID are documented in CLAUDE.md > Configuration.

For each issue extract:

- `issue_key`, `summary`, `priority`, `status`, `description`, `created_at`, `reporter`

Store as `production_bugs`. Record total count as `total_bugs`.

If zero bugs found: skip to Step 5 with an empty list.

---

## Step 3: Read #triage-critical-bug and #qa-bug-trends

Search for the `#triage-critical-bug` Slack channel by name and read its messages from the period.

Also read `#qa-bug-trends` for the same period.

From both channels, extract:

- Critical incidents or production issues not already in `production_bugs`
- Additional context on bugs already listed (root cause, impact, resolution timeline)

Store as `slack_bug_signals`. Add any new production issues found in Slack to `production_bugs` with `source = "slack"`.

---

## Step 4: Analyze Each Bug

For each bug in `production_bugs`, determine:

### 4a: Associated project

- Search Notion Projects database for a project matching the affected feature or bug description
- Search Jira: `project = DATA AND (summary ~ "{bug_summary}" OR text ~ "{bug_summary}") ORDER BY created DESC`
- Store `associated_project` (name or `"unknown"`) and `data_initiative_key` (e.g., `DATA-123`, or empty)

### 4b: Test suite existence

- If `associated_project` found: check if the project has a Test Suite subpage in Notion
- Store `test_suite_existed` = `true` or `false`, and `test_suite_url` if found

### 4c: PRD scope check

- If `associated_project` found: read the PRD and check whether the behavior that broke was mentioned anywhere — as a requirement, edge case, or assumption
- `in_prd_scope` = `true` if the behavior was described or implied in the PRD
- `in_prd_scope` = `false` if the behavior was not anticipated in the PRD at all
- `scope_notes` = brief description of what was or was not in scope

### 4d: Coverage check

- If `test_suite_existed = true`: check whether the test suite had a scenario covering the broken behavior
  - `was_covered` = `true` if a scenario matches the failure
  - `was_covered` = `false` if no scenario covered it
  - `coverage_notes` = what was missing or what scenario existed

### 4e: Classify the escape

Using the above, assign one of three escape types:

| Type | Condition | Meaning |
|---|---|---|
| `scope_miss` | `in_prd_scope = false` | The behavior was never anticipated — not in the PRD, not in scope. Discovery or requirements gap. |
| `coverage_gap` | `in_prd_scope = true` AND (`test_suite_existed = false` OR `was_covered = false`) | The behavior was known but no test covered it. QA coverage gap. |
| `implementation_slip` | `in_prd_scope = true` AND `was_covered = true` | A scenario existed and was executed, but the implementation introduced a bug that passed through. |
| `unknown` | Cannot determine from available data | Not enough context to classify. |

Store `escape_type` and `escape_description` — one sentence explaining why this bug escaped.

---

## Step 5: Build the Report

```text
🐛 Escaped Defect Analysis — {from_date} to {to_date}

{total_bugs} production bug(s) found.
🔍 Scope miss: {N} | ⚠️ Coverage gap: {N} | 🔁 Implementation slip: {N} | ❓ Unknown: {N}

---

{for each bug, sorted by priority:}

{priority_emoji} {issue_key} — {summary}
   Priority: {priority} | Status: {status} | Created: {created_at}
   Project: {associated_project}
   Test suite: {test_suite_label}
   Escape type: {escape_type_emoji} {escape_type_label}
   Why it escaped: {escape_description}
   Link: {jira_url}

---

By project:
{for each associated project:}
  {associated_project}: {N} bug(s) — {breakdown by escape type}

Patterns to address:
{for scope_miss bugs:} Discovery gap — "{prd_title}" did not anticipate: {scope_notes}
{for coverage_gap bugs:} Coverage gap — {coverage_notes}
{for implementation_slip bugs:} Consider adding regression scenario: {coverage_notes}
```

Where:

- `priority_emoji` — 🔴 Critical, 🟠 High, 🟡 Medium, ⚪ Low
- `test_suite_label` — `✅ existed` or `🔴 missing`
- `escape_type_emoji` — 🔍 scope_miss, ⚠️ coverage_gap, 🔁 implementation_slip, ❓ unknown
- `escape_type_label` — `Scope miss`, `Coverage gap`, `Implementation slip`, `Unknown`

If `total_bugs = 0`:

```text
✅ No production bugs found from {from_date} to {to_date}.
```

---

## Step 6: Log to Bug Impact Record

For each bug where `escape_type` is not `unknown`, append to `wins/bug-impact-log.md` (relative to the current workspace/project root).

If the file does not exist, create it with:

```markdown
# Bug Impact Log

Tracks bugs that reached production and escaped QA coverage.
Used for IDP reports and quarterly planning conversations.
```

Append for each bug:

```markdown
## {issue_key} — {summary}

| Field | Value |
|---|---|
| Date | {created_at} |
| Priority | {priority} |
| Project | {associated_project} |
| Escape type | {escape_type_label} |
| Why it escaped | {escape_description} |
| Test suite existed | {test_suite_existed} |
| Jira | {jira_url} |
```

> This file is local-only (gitignored in the Brain project) and is never pushed.

---

## Step 7: Output to Claude

Always print the full report from Step 5.

If the log was updated:
> Bug impact log updated at wins/bug-impact-log.md ({N} entries added).

If no bugs or all unknown:
> Nothing logged — no classifiable escaped defects found in this period.
