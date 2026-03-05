---
description: Post an updated QA status message to one or all active project channels. Reads current artifact state from Notion and compares with the last QE message in each channel.
---

# QA Status Update

Post a fresh QA status update to the Slack channel of one or all active projects.
Reads the current state of every QA artifact from Notion and shows what has changed
since the last time a QE sent a message in that channel.

Run this when you want to give the team a current view of QA status without
having to check Notion manually for each project.

## Usage

```bash
/qa:qa-status-update [project_name_or_notion_url]
```

**Optional:**

- `project_name_or_notion_url` — if provided, update only that project.
  If omitted, update all active projects from the kickoff log.

---

You are executing the `/qa:qa-status-update` command.

## Step 1: Determine Scope

Arguments: $ARGUMENTS

**If an argument is provided:**

- If it looks like a Notion URL — use it as `prd_url`, set `scope = "single"`
- Otherwise — search the Notion Projects database for a project whose `Name` contains
  the argument. If found, use that project. If multiple matches, list them and ask the user
  to confirm. Set `scope = "single"`.

**If no argument is provided:**

- Set `scope = "all"`
- Read the kickoff log at `~/.qa-kickoff-log.json`
- Build `project_list` from all entries in `kicked_off` where `skipped != true`
- If the log is empty or missing:
  > No kicked-off projects found in ~/.qa-kickoff-log.json. Run /qa:check-new-prds first,
  > or provide a project name or Notion URL as an argument.

---

## Step 2: For Each Project — Read Current State from Notion

For each project in scope:

Fetch the PRD page and extract:

- `prd_title`, `prd_page_id`, `prd_url`
- `pm_name`, `eng_lead_name`, `qe_name`
- `project_status` — current Status field
- `release_date` — any release or target date field (empty string if not set)

From child pages, locate and read each artifact:

**QA Brief:**
- `qa_brief_exists` = true/false
- `qa_brief_url` if found

**Adoption Review:**
- `adoption_verdict` = `ready`, `needs_clarification`, `incomplete`, `missing`
- `adoption_review_url` if found

**Test Suite:**
- `suite_status` = `approved`, `draft`, `missing`
- `suite_verdict` = `ready` or `review_first` (from content)
- `risk_coverage_score` — if present in content
- `bdd_quality_score` — if present in content
- `open_questions_count` — open questions with no owner assigned
- `test_suite_notion_url` if found

---

## Step 3: For Each Project — Read the Project Slack Channel

Find the project Slack channel:

1. Search for channels whose Topic contains `prd_url`
2. Fallback: search for channels whose name matches `proj-{slugified prd_title}`

If no channel found: set `slack_channel_found = false`, skip to Step 4 for this project.

If found: read the last 30 days of messages.

Find the most recent message from Beatriz (`U08UEQ22H7W`) or Giovani (`U08UEQ5MJDA`):

- Store `last_qe_message_date` (ISO date) and `last_qe_message_summary` (brief description)
- Calculate `days_since_last_qe_update` = today minus `last_qe_message_date`
- If no QE message found in 30 days: `last_qe_message_date = "never"`, `days_since_last_qe_update = null`

> **Config note:** QE Slack User IDs are documented in CLAUDE.md > Configuration.

---

## Step 4: Decide Whether to Post

For each project, post the update if **any** of the following:

- `scope = "single"` (always post when a specific project is requested)
- `days_since_last_qe_update > 5` (more than 5 days with no QE message)
- `last_qe_message_date = "never"`
- `slack_channel_found = false` (note the gap but do not post)

If `scope = "all"` and none of the above apply, skip the project and note:
> {prd_title} — skipped (last QE update {N} days ago, no changes to report)

---

## Step 5: Build and Post the Status Message

For each project where posting was decided:

Build this message:

```text
📊 QA Status — {prd_title}
{release_date_line}
📋 Project status: {project_status}

📄 PRD: {prd_url}
📋 QA Brief: {qa_brief_emoji} {qa_brief_label}
📊 Adoption Review: {adoption_emoji} {adoption_verdict}
🧪 Test Suite: {suite_emoji} {suite_label}
{suite_scores_line}
❓ Open questions: {open_questions_count}
{data_initiative_line}

📅 Last QE update: {last_qe_update_line}
```

Where:

- `release_date_line` — `📅 Release: {release_date} ({N} days away)` if set and future;
  `📅 Release: {release_date} (⚠️ overdue by {N} days)` if past;
  omit line entirely if not set
- `qa_brief_emoji` — `✅` if exists, `🔴` if not
- `qa_brief_label` — `{qa_brief_url}` or `missing`
- `adoption_emoji` — `🟢` ready, `🟡` needs_clarification, `🔴` incomplete/missing
- `suite_emoji` — `✅` approved, `⚠️` draft, `🔴` missing
- `suite_label` — `approved — {test_suite_notion_url}` / `draft — {test_suite_notion_url}` / `missing`
- `suite_scores_line` — `   {suite_verdict_emoji} {suite_verdict} | 🎯 {risk_coverage_score}% coverage | ✍️ {bdd_quality_score}% BDD`
  if scores are available; omit if not
- `suite_verdict_emoji` — `✅` ready, `⚠️` review_first
- `data_initiative_line` — `🎯 Jira: {data_initiative_url}` if available (look up from Notion PRD metadata or TQA epic description); omit if not found
- `last_qe_update_line` — `{days_since_last_qe_update} days ago — {last_qe_message_summary}`
  or `never — no QE message found in this channel`

Post to `slack_channel_id`.

If posting fails, note the failure and continue to the next project.

---

## Step 6: Output Summary to Claude

Print a summary of what was done:

```text
QA Status Update — {scope_label}

{for each project:}
  ✅ {prd_title} — posted to #{slack_channel_name}
  ⏭️ {prd_title} — skipped (last update {N} days ago)
  ⚠️ {prd_title} — no Slack channel found

Done. Posted: {N} | Skipped: {N} | No channel: {N}
```

Where `scope_label` is `{prd_title}` for single, or `all active projects ({N})` for bulk.
