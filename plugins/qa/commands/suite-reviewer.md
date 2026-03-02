---
description: Review an existing Test Suite Notion page against its PRD and QA Brief. Evaluates coverage, BDD quality, and automation tagging. Removes [DRAFT] from the title when done.
---

# Suite Reviewer

Review a Test Suite Notion page: check coverage against the QA Brief risk areas, evaluate BDD scenario quality, and assess automation tagging. Removes the `[DRAFT]` prefix from the test suite title when the review is complete.

This command is the standalone version of the `suite-reviewer` pipeline agent. It fetches all context from Notion — PRD, QA Brief, and test suite content — and runs the review.

## Usage

```bash
/qa:suite-reviewer <test_suite_notion_url>
```

**Required:**

- `test_suite_notion_url` — The Notion page URL of the `[DRAFT] Test Suite` to review

---

You are executing the `/qa:suite-reviewer` command.

## Step 1: Parse and Validate Input

Arguments: $ARGUMENTS

Parse the arguments:

- **First argument:** Test Suite Notion URL (required)

If the URL is missing, stop:
> Error: A Test Suite Notion URL is required.
> Usage: /qa:suite-reviewer <test_suite_notion_url>

---

## Step 2: Fetch the Test Suite Page

Fetch the test suite page using the Notion MCP. Extract:

- `test_suite_title` — the page title (e.g., `[DRAFT] Test Suite — Loyalty Revamp`)
- `test_suite_page_id` — the Notion page ID
- `test_suite_content` — the full page content as text

If the page is not accessible, stop:
> Error: Could not access the test suite page at [url]. Verify the link and your Notion permissions.

---

## Step 3: Fetch the Parent PRD

Use the Notion MCP to get the parent page of the test suite.

Fetch the parent page and all its child pages. Consolidate content into `prd_text`. Store the PRD URL as `prd_url`.

If the parent page cannot be found or accessed, set `prd_text = ""` and note the gap — the review will proceed with limited context.

---

## Step 4: Fetch the QA Brief

From the parent PRD page, fetch its child pages and look for one with "QA Brief" in the title.

Fetch that page and store its full content as `qa_brief_content`.

If no QA Brief is found, set `qa_brief_content = ""` and note that risk area coverage analysis will be limited.

---

## Step 5: Run the Suite Reviewer Agent

Use the Task tool to invoke the `suite-reviewer` agent.

Pass the following four sections as the input, separated by `---SECTION---`:

1. `test_suite_notion_url` (the test suite Notion page URL from the input)
2. `prd_text` (the full PRD content, or empty string if unavailable)
3. `qa_brief_content` (the QA Brief content, or empty string if unavailable)
4. `test_suite_content` (the full test suite page content as formatted text)

Note: section 4 contains the test suite page content rather than raw JSON. The agent will parse the scenarios from the formatted page.

The agent will:
- Evaluate coverage, BDD quality, and automation tagging
- Remove `[DRAFT]` from the test suite Notion page title via MCP
- Return a JSON object with the verdict and review details

Parse the result and store:
- `suite_verdict` — `ready` or `review_first`
- `draft_removed` — `true` or `false`
- `suite_review_json` — the full JSON output

---

## Step 6: Output Summary to User

```
{verdict_emoji} Suite Review — {test_suite_title}

Verdict: {suite_verdict}
{summary from suite_review_json}

Scores:
  Coverage:         {risk_coverage}%
  BDD quality:      {bdd_quality}%
  Automation tags:  {automation_tagging}%

Risk Coverage:
  ✅ covered: {count}
  🟡 partially covered: {count}
  🔴 not covered: {count}

{bdd_issues_line}
{tagging_issues_line}
{missing_scenarios_line}

[DRAFT] removed: {draft_removed}
Test suite: {test_suite_notion_url}
```

Where:
- `verdict_emoji` is `✅` for `ready` or `⚠️` for `review_first`
- `bdd_issues_line` is `BDD issues found: {N} — see test suite for details.` if count > 0, empty otherwise
- `tagging_issues_line` is `Automation tagging issues: {N}` if count > 0, empty otherwise
- `missing_scenarios_line` is `Missing scenarios suggested: {N}` if count > 0, empty otherwise
