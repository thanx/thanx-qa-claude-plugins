---
description: Check for new PRDs in Technical Discovery status that haven't been kicked off yet. Lists pending projects and offers to run /qa:qa-kickoff for each one.
---

# Check New PRDs

Scan the Notion Projects database for PRDs in `(2.5) Technical discovery` status that haven't been kicked off yet. For each new one, offer to run the full QA kickoff pipeline.

Run this at the start of your day or whenever a Notion automation notifies you that a project entered Technical Discovery.

## Usage

```bash
/qa:check-new-prds
```

No arguments required.

---

You are executing the `/qa:check-new-prds` command.

## Step 1: Read the Kickoff Log

Read the kickoff log file at `~/.qa-kickoff-log.json`.

This file tracks which PRD Notion page IDs have already been kicked off. It has the following structure:

```json
{
  "kicked_off": [
    {
      "page_id": "...",
      "title": "...",
      "kicked_off_at": "2026-03-01T10:00:00Z"
    }
  ]
}
```

If the file does not exist, treat it as `{ "kicked_off": [] }`.

Store the list of already-processed page IDs as `processed_ids`.

---

## Step 2: Query the Projects Database

Query the Notion Projects database (data source ID: `collection://0d7ef002-875f-453b-bb05-7789a3436086`) for all entries where `Status = "(2.5) Technical discovery"`.

Use the `notion-query-database-view` MCP tool with the following SQL:

```sql
SELECT url, "Name", "Status", "Product Manager", "Eng Lead", "QE",
       "date:Date PRD ready:start", "Channel in slack with QA",
       "Tests Suite Created", "QA - Read PRD"
FROM "collection://0d7ef002-875f-453b-bb05-7789a3436086"
WHERE "Status" = '(2.5) Technical discovery'
```

If the query fails, stop and report:
> Error: Could not query the Projects database. Check your Notion MCP connection.

Store the results as `technical_discovery_prds`.

---

## Step 3: Filter New PRDs

From `technical_discovery_prds`, exclude any entries whose Notion page ID is in `processed_ids`.

A page ID can be extracted from the `url` field (last segment of the Notion URL).

Store the remaining entries as `new_prds`.

---

## Step 4: Report Results

If `new_prds` is empty:

```
✅ No new PRDs to kick off.

{N} project(s) in Technical Discovery — all already processed.
```

Stop here.

---

If `new_prds` is not empty, print a summary:

```
🔍 {N} new PRD(s) in Technical Discovery:

{For each PRD:}
  {index}. {Name}
     PM: {Product Manager}  |  Eng Lead: {Eng Lead}  |  QE: {QE}
     PRD ready: {Date PRD ready}
     Suite created: {Tests Suite Created}  |  QA Brief: {QA - Read PRD}
     Link: {url}
```

Where `Tests Suite Created` and `QA - Read PRD` are shown as `✅ yes` or `—`.

---

## Step 5: Offer Kickoff for Each New PRD

For each PRD in `new_prds`, ask:

> Run /qa:qa-kickoff for "{Name}"? (yes / skip / stop)

- **yes** — proceed with kickoff for this PRD
- **skip** — mark it as processed without running the pipeline
- **stop** — exit without processing any more PRDs

If the user answers **yes**:

1. Use the Task tool to invoke the `qa-kickoff` command with the PRD Notion URL.
2. Wait for the pipeline to complete.
3. On success: add the page ID to the kickoff log.
4. On failure: note the error, do not add to log (so it will show up again next time).

If the user answers **skip**:

1. Add the page ID to the kickoff log with a note: `"skipped": true`.
2. Continue to the next PRD.

---

## Step 6: Update the Kickoff Log

After processing all PRDs, write the updated `~/.qa-kickoff-log.json`.

Add a new entry for each PRD that was kicked off or skipped:

```json
{
  "page_id": "...",
  "title": "...",
  "kicked_off_at": "{ISO timestamp}",
  "skipped": false
}
```

---

## Step 7: Output Final Summary

```
Done.

Kicked off:  {N}
Skipped:     {N}
Remaining:   {N} (will appear again next time)
```
