---
description: Full release readiness check for a project in Coming Soon status. Sweeps Notion artifacts, the DATA Jira initiative, and the project Slack channel to produce a Go / Conditional Go / Not Ready verdict. Posts the report to the project channel.
---

# Release Readiness Check

Run a complete release readiness sweep for a project that is about to go to production
(`(1.5) Coming soon`). Checks every QA artifact, open bugs in the DATA Jira initiative,
and the project Slack channel — including test execution results reported by devs that
may never have been updated in Notion or Jira.

## Usage

```bash
/qa:release-readiness <project_name_or_notion_url>
```

**Required:**

- `project_name_or_notion_url` — the project name (partial match OK) or the Notion PRD URL

---

You are executing the `/qa:release-readiness` command.

## Step 1: Parse Input and Find the Project

Arguments: $ARGUMENTS

If no argument was provided, stop and report:
> Error: project name or Notion URL is required. Usage: `/qa:release-readiness <project_name_or_notion_url>`

Parse the first argument:

- If it looks like a Notion URL (`https://notion.so/...`) — use it directly as `prd_url`
- Otherwise — treat it as a project name and search the Notion Projects database

**If searching by name:**

Query the Notion Projects database (data source ID: `collection://0d7ef002-875f-453b-bb05-7789a3436086`) for
entries where `Name` contains the argument (case-insensitive).

> **Config note:** This database ID is documented in CLAUDE.md > Configuration.

If zero matches: stop and report:
> Error: No project found matching "{argument}". Try the full project name or paste the Notion PRD URL directly.

If multiple matches: list them and ask the user to confirm which one.

If exactly one match: proceed with that project's Notion URL as `prd_url`.

> This command is intended for projects in `(1.5) Coming soon` status — projects that are
> about to go to production and need a final QA sign-off. For earlier phases, use
> `/qa:qa-status-update` instead.

---

## Step 2: Read the PRD and Extract Project Metadata

Fetch the PRD page using the Notion MCP. Extract:

- `prd_title` — page title
- `prd_page_id` — page ID
- `pm_name` — Product Manager field
- `eng_lead_name` — Eng Lead field
- `qe_name` — QE field
- `release_date` — any release date or target date field (empty string if not set)
- `project_status` — current Status field value

If `project_status` is not `(1.5) Coming soon`, stop and report:
> Error: `/qa:release-readiness` is only for projects in `(1.5) Coming soon`. Current status: `{project_status}`.
> Use `/qa:qa-status-update` for earlier phases.

Fetch all child pages from the PRD page ID and consolidate into `prd_text`.

---

## Step 3: Check QA Artifacts in Notion

From the PRD child pages, locate and check each artifact:

### QA Brief

- Found: `qa_brief_exists = true`, store `qa_brief_url`
- Not found: `qa_brief_exists = false`

### Adoption Review

- Found: read the verdict from the page content
  - `adoption_verdict` = `ready`, `needs_clarification`, `incomplete`, or `unknown`
  - Store `adoption_review_url`
- Not found: `adoption_verdict = "missing"`

### Test Suite

- Found: check title for `[DRAFT]`
  - `[DRAFT]` present: `suite_status = "draft"`
  - No `[DRAFT]`: `suite_status = "approved"`
- Read content and extract:
  - `suite_verdict` — `ready` or `review_first`
  - `risk_coverage_score` — coverage percentage if present
  - `bdd_quality_score` — BDD quality percentage if present
  - `open_questions_count` — rows with Status = `Open` in the open questions table
  - `uncovered_requirements_count` — `UNCOVERED` entries in the coverage matrix
- Store `test_suite_notion_url`
- Not found: `suite_status = "missing"`, `open_questions_count = 0`,
  `uncovered_requirements_count = 0`, `test_suite_notion_url = ""`

### QA Scorecard

Read the PRD Notion page properties and extract:

- `scorecard_suite_created` — `Tests Suite Created` checkbox value
- `scorecard_prd_read` — `QA - Read PRD` checkbox value
- `scorecard_slack_channel` — `Channel in slack with QA` checkbox value

---

## Step 4: Check Open Bugs in the DATA Jira Initiative

All projects at Thanx live in the DATA board. The QA kickoff creates an initiative in the DATA project — this is
the source of truth for open issues on this project.

Find the DATA initiative for this project:

- Look for `jira_key` from the PRD metadata or any `[QA Kickoff]` note in the PRD content
- If not found in the PRD, search Jira: `project = DATA AND summary ~ "{prd_title}" AND issuetype = Initiative
  ORDER BY created DESC`
- Store as `data_initiative_key` (e.g., `DATA-123`) and `data_initiative_url`
- If still not found: `data_initiative_key = ""`, note the gap and skip sub-steps below

> **Config note:** DATA project key documented in CLAUDE.md > Configuration.

**Search for open issues under the initiative:**

Query: `parent = {data_initiative_key} AND resolution = Unresolved ORDER BY priority ASC`

Count by priority:

- `bugs_critical` — Critical priority, unresolved
- `bugs_high` — High priority, unresolved
- `bugs_medium` — Medium priority, unresolved
- `open_issues_total` — all unresolved

For each Critical or High issue, store a brief summary: `{issue_key}: {summary}`.

---

## Step 5: Read the Project Slack Channel

Find the project Slack channel:

1. Search for channels whose Topic contains `prd_url`
2. Fallback: search for channels whose name matches `proj-{slugified prd_title}`

If no channel found: set `slack_channel_found = false`, `slack_channel_id = ""`, note the gap.

If found: set `slack_channel_found = true`, `slack_channel_id = <found_channel>.id`.
Read the last 7 days of messages from the channel.

Extract:

**Test execution signals** — messages from devs or QEs mentioning test results:

- Keywords: "tested", "test passed", "test failed", "bug", "looks good", "LGTM", "verified", "confirmed",
  "done testing", "ready for QA", "QA done"
- Store as `test_signals` — list of `{author, date, summary}` (max 5 most recent)

**Last QE message** — most recent message from Beatriz (`U08UEQ22H7W`) or Giovani (`U08UEQ5MJDA`):

- Store `last_qe_message_date` and `last_qe_message_summary`
- If none found: `last_qe_message_date = "never"`

**Blockers mentioned in chat** — messages about blockers, unresolved issues, or critical problems:

- Store as `slack_blockers` — list of `{author, date, summary}` (max 3)

> **Config note:** QE Slack User IDs are documented in CLAUDE.md > Configuration.

---

## Step 6: Calculate Verdict

Evaluate all signals:

**NOT READY** if any of the following:

- `qa_brief_exists = false`
- `suite_status` is `missing` or `draft`
- `suite_verdict = "review_first"`
- `uncovered_requirements_count > 0`
- `bugs_critical > 0`
- `open_questions_count >= 3`
- `adoption_verdict` is `incomplete`, `missing`, or `unknown`

**CONDITIONAL GO** if any of the following (and none of the NOT READY conditions apply):

- `open_questions_count > 0`
- `bugs_high > 0`
- `adoption_verdict = "needs_clarification"`
- `last_qe_message_date = "never"`
- `data_initiative_key = ""`
- `risk_coverage_score < 80` (when available)
- Any `slack_blockers` found

**GO** if none of the above apply.

Store `verdict`.

---

## Step 7: Build the Report

```text
🚦 Release Readiness — {prd_title}
{release_date_line}
📋 Status: {project_status}

VERDICT: {verdict_emoji} {verdict_label}

--- QA Artifacts ---
📋 QA Brief: {qa_brief_emoji} {qa_brief_label}
📊 Adoption Review: {adoption_emoji} {adoption_verdict}
🧪 Test Suite: {suite_emoji} {suite_label}
{suite_scores_line}
❓ Open questions: {open_questions_count}
{uncovered_line}

--- Jira (DATA) ---
📌 Initiative: {data_initiative_line}
🐛 Open issues: {open_issues_total} {bugs_breakdown}
{critical_high_list}

--- Slack Channel ---
💬 Last QE update: {last_qe_message_date} — {last_qe_message_summary}
{test_signals_section}
{slack_blockers_section}

--- Blocking Issues ---
{blocking_list}

--- Recommendation ---
{recommendation}
```

Where:

- `verdict_emoji` — `✅` for `go`, `⚠️` for `conditional_go`, `🔴` for `not_ready`
- `verdict_label` — `GO`, `CONDITIONAL GO`, `NOT READY`
- `release_date_line` — `📅 Release date: {release_date} ({N} days away)` if set; `(⚠️ overdue by N days)` if
  past; `📅 Release date: not set` if empty
- `qa_brief_emoji` — `✅` if exists, `🔴` if not
- `qa_brief_label` — `exists — {qa_brief_url}` or `MISSING`
- `adoption_emoji` — `🟢` ready, `🟡` needs_clarification, `🔴` incomplete/missing, `⚪` unknown
- `suite_emoji` — `✅` approved, `⚠️` draft, `🔴` missing
- `suite_label` — `approved`, `draft — pending review`, `MISSING`
- `suite_scores_line` — `Coverage: {risk_coverage_score}% | BDD quality: {bdd_quality_score}%` if available, empty otherwise
- `uncovered_line` — `⚠️ {N} uncovered requirement(s)` if > 0, empty otherwise
- `data_initiative_line` — `{data_initiative_key} ({data_initiative_url})` if found, `NOT FOUND — initiative missing in DATA`
- `bugs_breakdown` — `(Critical: {bugs_critical}, High: {bugs_high}, Medium: {bugs_medium})` if > 0, empty if 0
- `critical_high_list` — list of Critical/High issue keys and summaries if any, empty otherwise
- `test_signals_section` — list of recent test execution signals, or `No test execution signals in the last 7 days.`
- `slack_blockers_section` — list of blockers from chat, or empty
- `blocking_list` — numbered list of every NOT READY or CONDITIONAL GO trigger, or `None — project appears ready.`
- `recommendation` — 1-2 sentences on what needs to happen before release

---

## Step 8: Post Report to Project Channel

If `slack_channel_found = true`: post the full report to `slack_channel_id`.

If `slack_channel_found = false`:
> Warning: No project Slack channel found. Report not posted — verify the channel exists and has the PRD URL in its Topic.

---

## Step 9: Output to Claude

Always print the full report to Claude output, regardless of Slack result.

If Slack posting failed, append:
> Slack posting failed — copy and paste the report above manually if needed.
