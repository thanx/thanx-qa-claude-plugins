---
description: Review a Notion PRD for adoption criteria completeness. Analyzes the 4 adoption questions and creates an [ADOPTION REVIEW] subpage under the PRD.
---

# Adoption Review

Analyze a Notion PRD and evaluate whether it defines adoption criteria clearly enough to measure feature success after launch.

This command is the standalone version of the `adoption-review` pipeline agent. It fetches the PRD from Notion,
runs the analysis, and creates an `[ADOPTION REVIEW]` subpage with the results.

## Usage

```bash
/qa:adoption-review <notion_prd_url>
```

**Required:**

- `notion_prd_url` — The Notion page URL of the PRD to analyze

---

You are executing the `/qa:adoption-review` command.

## Step 1: Parse and Validate Input

Arguments: $ARGUMENTS

Parse the arguments:

- **First argument:** Notion PRD URL (required)

If the Notion PRD URL is missing, stop:
> Error: A Notion PRD URL is required.
> Usage: /qa:adoption-review <notion_prd_url>

---

## Step 2: Read the PRD from Notion

Fetch the main PRD page using the Notion MCP. Extract:

- `prd_title` — the page title
- `prd_page_id` — the Notion page ID

Fetch child pages directly from the PRD page ID. Fetch each child page individually and consolidate all content into `prd_text`.

If the PRD is not accessible, stop:
> Error: Could not access the PRD at [url]. Verify the link and your Notion permissions.

---

## Step 3: Run the Adoption Review Agent

Use the Task tool to invoke the `adoption-review` agent.

Pass `prd_text` as the input. The agent returns a JSON object with the verdict and criteria analysis.

Parse and store the full JSON output as `adoption_json`.

---

## Step 4: Create or Update the [ADOPTION REVIEW] Subpage in Notion

Fetch child pages directly from the PRD page ID. Look for any with "[ADOPTION REVIEW]" in the title.

Apply this guardrail:

- **No existing [ADOPTION REVIEW] page:** create a new subpage
- **One [ADOPTION REVIEW] page found:** update it in place
- **Two or more found:** stop and report:
  > Found {N} existing [ADOPTION REVIEW] pages under this PRD: {titles and URLs}
  > Please delete or consolidate them manually before running this command again.

When creating (no existing page found), use `notion-create-pages` with:

- `parent: { "type": "page_id", "page_id": "{prd_page_id}" }` — this sets the parent correctly. Do NOT use `parent_id`.
- `pages[0].properties.title`: `[ADOPTION REVIEW] {prd_title}`
- `pages[0].content`: the formatted content below

When updating, use `notion-update-page` with `command: "replace_content"`.

Format the page content as follows:

---

### [ADOPTION REVIEW] Page Format

**Title:** `[ADOPTION REVIEW] {prd_title}`

**Content:**

```text
## Adoption Review

| Field | Value |
|---|---|
| Verdict | {verdict_emoji} {verdict} |
| Generated | {current timestamp} |
| Source PRD | {prd_url} |

> {summary}

---

## Criteria Analysis

| # | Question | Status | Finding |
|---|---|---|---|
| 1 | Expected customer behavior post-launch | {status_emoji} {status} | {finding} |
| 2 | Adoption signal | {status_emoji} {status} | {finding} |
| 3 | Value metric or evidence | {status_emoji} {status} | {finding} |
| 4 | Expected timeframe | {status_emoji} {status} | {finding} |

---

## Questions for PM
```

If `questions_for_pm` is not empty, add a numbered list after the "Questions for PM" heading:

```text
1. {question} _(Criterion {criterion})_
```

If `questions_for_pm` is empty, add: `None — all criteria are clearly defined in the PRD.`

---

Where status emojis are:

- `found` → `✅`
- `partial` → `🟡`
- `not_found` → `🔴`

---

Store the Notion subpage URL as `review_notion_url`.

---

## Step 5: Output Summary to User

```text
{verdict_emoji} Adoption Review — {prd_title}

Verdict: {verdict}
{summary}

Criteria:
  ✅ found: {count}  🟡 partial: {count}  🔴 not found: {count}

{questions_line}

Review page: {review_notion_url}
```

Where `questions_line` is:

- `Questions for PM: {N}` if there are open questions
- `No open questions.` if all criteria are found
