---
description: Re-run adoption review and suite reviewer for PRDs that moved to In Progress. Detects PRD changes since the initial kickoff and updates the existing Notion subpages.
---

# Recheck PRDs

Scan the Notion Projects database for PRDs in `(2) In Progress` status that haven't been re-checked yet. For each one, re-run the adoption review and suite reviewer to catch any changes made to the PRD since the initial kickoff.

Run this when a Notion automation notifies you that a project entered In Progress.

## Usage

```bash
/qa:recheck-prds
```

No arguments required.

---

You are executing the `/qa:recheck-prds` command.

## Step 1: Read the Kickoff Log

Read the kickoff log file at `~/.qa-kickoff-log.json`.

It has the following structure:

```json
{
  "kicked_off": [...],
  "rechecked": [
    {
      "page_id": "...",
      "title": "...",
      "rechecked_at": "2026-03-01T10:00:00Z"
    }
  ]
}
```

If the file does not exist or `rechecked` is absent, treat it as an empty list.

Store the list of already-rechecked page IDs as `rechecked_ids`.

---

## Step 2: Query the Projects Database

Query the Notion Projects database (data source ID: `collection://0d7ef002-875f-453b-bb05-7789a3436086`) for all entries where `Status = "(2) In Progress"` and `Tests Suite Created = "__YES__"`.

Use the `notion-query-database-view` MCP tool with the following SQL:

```sql
SELECT url, "Name", "Status", "Product Manager", "Eng Lead", "QE",
       "Tests Suite Created", "QA - Read PRD", "Channel in slack with QA"
FROM "collection://0d7ef002-875f-453b-bb05-7789a3436086"
WHERE "Status" = '(2) In Progress'
AND "Tests Suite Created" = '__YES__'
```

If the query fails, stop and report:
> Error: Could not query the Projects database. Check your Notion MCP connection.

Store the results as `inprogress_prds`.

---

## Step 3: Filter Unrechecked PRDs

From `inprogress_prds`, exclude any entries whose Notion page ID is in `rechecked_ids`.

Store the remaining entries as `new_rechecks`.

---

## Step 4: Report Results

If `new_rechecks` is empty:

```
✅ No PRDs to recheck.

{N} project(s) in In Progress — all already rechecked.
```

Stop here.

---

If `new_rechecks` is not empty, print a summary:

```
🔄 {N} PRD(s) moved to In Progress — ready for recheck:

{For each PRD:}
  {index}. {Name}
     PM: {Product Manager}  |  Eng Lead: {Eng Lead}  |  QE: {QE}
     Link: {url}
```

---

## Step 5: Offer Recheck for Each PRD

For each PRD in `new_rechecks`, ask:

> Recheck adoption review and test suite for "{Name}"? (yes / skip / stop)

- **yes** — re-run both agents for this PRD
- **skip** — mark as rechecked without running
- **stop** — exit without processing more PRDs

If the user answers **yes**, execute Steps 6–8 for that PRD.

---

## Step 6: Read Current PRD and Existing Subpages

Fetch the PRD main page and all its child pages from Notion. Consolidate content into `prd_text`. Store the PRD page ID as `prd_page_id`.

From the PRD's child pages, find:

- The `📋 QA Brief` subpage → store its content as `qa_brief_content`
- The `Test Suite` page (title contains "Test Suite", does not need to start with `[DRAFT]`) → store its URL as `test_suite_notion_url` and content as `test_suite_content`
- The `[ADOPTION REVIEW]` subpage → store its Notion page ID as `adoption_review_page_id`

If the QA Brief is not found, set `qa_brief_content = ""` and note the gap.
If the Test Suite is not found, skip the suite reviewer step and note it.

---

## Step 7: Re-run Adoption Review

Use the Task tool to invoke the `adoption-review` agent with `prd_text`.

The agent returns a JSON object with the updated verdict and criteria analysis.

Update the existing `[ADOPTION REVIEW]` subpage in Notion (`adoption_review_page_id`) with the new results. Use the same format as `/qa:adoption-review` Step 4.

Add a note at the top of the page:

```
> ♻️ Re-checked on {current date} — project moved to In Progress.
```

Store:
- `adoption_verdict` — updated verdict
- `adoption_changed` — `true` if the verdict differs from what was previously in the page, `false` otherwise

---

## Step 8: Re-run Suite Reviewer

If a Test Suite page was found in Step 6:

Use the Task tool to invoke the `suite-reviewer` agent with the four sections:

1. `test_suite_notion_url`
2. `prd_text`
3. `qa_brief_content`
4. `test_suite_content`

The agent will:
- Re-evaluate coverage, BDD quality, and automation tagging
- Remove `[DRAFT]` from the test suite title if it still has it
- Return updated verdict JSON

Store:
- `suite_verdict` — updated verdict
- `suite_review_json` — full output

---

## Step 9: Post Update to Project Slack Channel

Find the project Slack channel by searching for channels whose Topic contains the PRD URL, or whose name matches `proj-{slugified title}`.

If a channel is found, post:

```
♻️ QA Recheck — {Name}

Project moved to In Progress. Here's the updated status:

{adoption_verdict_emoji} Adoption Review: {adoption_verdict}{adoption_changed_note}
{suite_verdict_emoji} Suite verdict: {suite_verdict}

PRD: {prd_url}
Test Suite: {test_suite_notion_url}
```

Where:
- `adoption_changed_note` is ` ⚠️ (changed since kickoff)` if `adoption_changed = true`, empty otherwise
- `suite_verdict_emoji` is `✅` for `ready` or `⚠️` for `review_first`
- `adoption_verdict_emoji` is `🟢` for `ready`, `🟡` for `needs_clarification`, `🔴` for `incomplete`

If no channel is found, skip the Slack post and note it in the output.

---

## Step 10: Update the Kickoff Log

Add an entry to the `rechecked` list in `~/.qa-kickoff-log.json`:

```json
{
  "page_id": "...",
  "title": "...",
  "rechecked_at": "{ISO timestamp}"
}
```

---

## Step 11: Output Summary

```
{For each rechecked PRD:}
♻️ {Name}
   {adoption_verdict_emoji} Adoption: {adoption_verdict}{adoption_changed_note}
   {suite_verdict_emoji} Suite: {suite_verdict}

Done. Rechecked: {N}  |  Skipped: {N}
```
