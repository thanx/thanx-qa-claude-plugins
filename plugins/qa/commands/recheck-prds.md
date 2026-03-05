---
description: Handle PRDs that moved to In Progress — runs the full kickoff for projects that skipped Technical Discovery, and re-runs adoption review + suite reviewer for projects that were already kicked off.
---

# Recheck PRDs

Scan the Notion Projects database for PRDs in `(2) In Progress` status and handle each one appropriately:

- **Never kicked off** (skipped Technical Discovery): run the full kickoff pipeline.
- **Already kicked off**: re-run adoption review and suite reviewer to catch PRD changes.

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

If the file does not exist, treat it as `{ "kicked_off": [], "rechecked": [] }`.
If `kicked_off` or `rechecked` are absent, treat each as an empty list.

Store:

- `kicked_off_ids` — list of page IDs already in `kicked_off`
- `rechecked_ids` — list of page IDs already in `rechecked`

---

## Step 2: Query the Projects Database

Query the Notion Projects database (data source ID: `collection://0d7ef002-875f-453b-bb05-7789a3436086`) for
**all** entries where `Status = "(2) In Progress"`.

> **Config note:** This database ID is documented in CLAUDE.md > Configuration. Update both locations if the
> Notion database changes.

Use the `notion-query-database-view` MCP tool with the following SQL:

```sql
SELECT url, "Name", "Status", "Product Manager", "Eng Lead", "QE",
       "Tests Suite Created", "QA - Read PRD", "Channel in slack with QA"
FROM "collection://0d7ef002-875f-453b-bb05-7789a3436086"
WHERE "Status" = '(2) In Progress'
```

If the query fails, stop and report:
> Error: Could not query the Projects database. Check your Notion MCP connection.

Store the results as `inprogress_prds`.

---

## Step 3: Classify PRDs into Two Groups

For each entry in `inprogress_prds`, derive a canonical page ID from the `url` field:

1. Remove any query string and fragment from the URL.
2. Take the last path segment.
3. Extract the trailing 32-character hex UUID (with or without dashes).
4. Normalize to lowercase dashed UUID form (e.g., `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`).

Apply the same canonicalization to all IDs in `kicked_off_ids` and `rechecked_ids` before comparing.

Split into two groups:

- **`needs_kickoff`** — page ID is **not** in `kicked_off_ids`. These projects skipped Technical Discovery and
  never had a kickoff. They need the full pipeline.
- **`needs_recheck`** — page ID is in `kicked_off_ids` but **not** in `rechecked_ids`. These were kicked off
  before and now need adoption review + suite reviewer re-run.

Entries already in both `kicked_off_ids` and `rechecked_ids` are fully processed — ignore them.

---

## Step 4: Report Results

If both groups are empty:

```text
✅ Nothing to do.

{N} project(s) in In Progress — all already handled.
```

Stop here.

---

If either group has entries, print a summary:

```text
📋 In Progress — action needed:

🆕 Never kicked off ({N}):
{For each PRD in needs_kickoff:}
  {index}. {Name}
     PM: {Product Manager}  |  Eng Lead: {Eng Lead}  |  QE: {QE}
     Link: {url}

🔄 Already kicked off — needs recheck ({N}):
{For each PRD in needs_recheck:}
  {index}. {Name}
     PM: {Product Manager}  |  Eng Lead: {Eng Lead}  |  QE: {QE}
     Link: {url}
```

Omit a section header if the group is empty.

---

## Step 5a: Full Kickoff for Never-Kicked-Off PRDs

For each PRD in `needs_kickoff`, ask:

> "{Name}" was never kicked off (skipped Technical Discovery). Run the full kickoff pipeline now?
> (yes / skip / stop)

- **yes** — invoke `/qa:qa-kickoff` with the PRD Notion URL using the Task tool. Wait for the pipeline to
  complete. On success, mark as kicked off in the log (Step 10 will handle this). On failure, note the error
  and do not mark as processed.
- **skip** — continue to the next PRD without running anything. Do not add to the log (it will appear again
  next time).
- **stop** — exit without processing more PRDs.

---

## Step 5b: Offer Recheck for Already-Kicked-Off PRDs

For each PRD in `needs_recheck`, ask:

> Recheck adoption review and test suite for "{Name}"? (yes / skip / stop)

- **yes** — re-run both agents for this PRD (Steps 6–9)
- **skip** — mark as rechecked without running
- **stop** — exit without processing more PRDs

If the user answers **yes**, execute Steps 6–9 for that PRD.

---

## Step 6: Read Current PRD and Existing Subpages

Fetch the PRD main page and all its child pages from Notion. Consolidate content into `prd_text`. Store the PRD
page ID as `prd_page_id`.

From the PRD's child pages, find:

- The `📋 QA Brief` subpage → store its content as `qa_brief_content`
- The `Test Suite` page (title contains "Test Suite", does not need to start with `[DRAFT]`) → store its URL as
  `test_suite_notion_url` and content as `test_suite_content`
- The `[ADOPTION REVIEW]` subpage(s) - if exactly one is found, store its Notion page ID as
  `adoption_review_page_id`. If two or more are found, store all page IDs as `adoption_review_page_ids` (list)
  and set `adoption_review_duplicate = true`. If none found, set `adoption_review_page_id = ""`.

If the QA Brief is not found, set `qa_brief_content = ""` and note the gap.
If the Test Suite is not found, skip the suite reviewer step and note it.

---

## Step 7: Re-run Adoption Review

Use the Task tool to invoke the `adoption-review` agent with `prd_text`.

The agent returns a JSON object with the updated verdict and criteria analysis.

**If exactly one `[ADOPTION REVIEW]` page was found in Step 6:** update the existing subpage with the new
results. Use the same format as `/qa:adoption-review` Step 4. Set `adoption_page_updated = true`.

**If no `[ADOPTION REVIEW]` page was found:** create a new `[ADOPTION REVIEW]` subpage under the PRD (same
format as `/qa:adoption-review` Step 4), store its page ID as `adoption_review_page_id`,
set `adoption_page_updated = true`, and set `adoption_changed = false` (no prior verdict to compare against).
Note in the output that the page was missing and was recreated.

**If two or more `[ADOPTION REVIEW]` pages were found (`adoption_review_duplicate = true`):** skip the page
update, set `adoption_page_updated = false`, and report:
> Warning: Found {N} existing [ADOPTION REVIEW] pages under this PRD. Skipping page update — please consolidate
> them manually.

Continue to Step 8 with the adoption verdict from the agent (the agent output is still valid even if the Notion
page update is skipped).

If `adoption_page_updated = true`, add a note at the top of the page:

```text
> ♻️ Re-checked on {current date} — project moved to In Progress.
```

Store:

- `adoption_verdict` — updated verdict
- `adoption_changed` — `true` if the verdict differs from what was previously in the page, `false` if the page
  was new or the verdict is unchanged

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

Find the project Slack channel by searching for channels whose Topic contains the PRD URL, or whose name matches
`proj-{slugified title}`.

If a channel is found, post:

```text
♻️ QA Recheck — {Name}

Project moved to In Progress. Here's the updated status:

{adoption_verdict_emoji} Adoption Review: {adoption_verdict}{adoption_changed_note}
{suite_status_line}

PRD: {prd_url}
{test_suite_line}
```

Where:

- `adoption_changed_note` is `⚠️ (changed since kickoff)` if `adoption_changed = true`, empty otherwise
- `adoption_verdict_emoji` is `🟢` for `ready`, `🟡` for `needs_clarification`, `🔴` for `incomplete`
- `suite_status_line` — if `test_suite_notion_url` exists:
  `{suite_verdict_emoji} Suite verdict: {suite_verdict}` where `suite_verdict_emoji` is `✅` for `ready`
  or `⚠️` for `review_first`; otherwise: `⚠️ Suite verdict: skipped (Test Suite page not found)`
- `test_suite_line` — if `test_suite_notion_url` exists: `Test Suite: {test_suite_notion_url}`;
  otherwise: `Test Suite: not found`

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

```text
{For each PRD that ran full kickoff:}
🆕 {Name} — full kickoff completed

{For each rechecked PRD:}
♻️ {Name}
   {adoption_verdict_emoji} Adoption: {adoption_verdict}{adoption_changed_note}
   {suite_verdict_emoji} Suite: {suite_verdict}

Done. Kicked off: {N}  |  Rechecked: {N}  |  Skipped: {N}
```
