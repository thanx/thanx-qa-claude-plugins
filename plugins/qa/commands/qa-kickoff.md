---
description: Run the full QA kickoff pipeline for a Notion PRD. Generates QA Brief, Adoption Review, Test Suite, Suite Review, and Scorecard Update. Creates a TQA tracking Epic and a Slack channel when Eng Lead is set.
---

# QA Kickoff Pipeline

Orchestrate the full QA kickoff for a project: five agents, with QA Brief and Adoption Review running in
parallel, followed by Test Suite generation, Suite Review, and Scorecard Update.

## Usage

```bash
/qa:qa-kickoff <notion_prd_url>
```

**Required:**

- `notion_prd_url` — The Notion PRD page URL for the project to kick off

---

You are executing the `/qa:qa-kickoff` command.

## Step 1: Parse and Validate Input

Arguments: $ARGUMENTS

Parse the arguments:

- **First argument:** Notion PRD URL (required)

If the Notion PRD URL is missing, stop:
> Error: A Notion PRD URL is required.
> Usage: /qa:qa-kickoff <notion_prd_url>

---

## Step 2: Read the PRD from Notion

Fetch the main PRD page using the Notion MCP. Extract:

- `prd_title` — the page title
- `prd_page_id` — the Notion page ID

Fetch child pages directly from the PRD page ID. Fetch each child page individually and consolidate all content into `prd_text`.

Extract team fields from the PRD page metadata:

- **Product Manager** → `pm_name`
- **Eng Lead** → `eng_lead_name`
- **QE** → `qe_name`

These fields are optional — if absent, store as empty string and continue.

If the PRD is not accessible, stop:
> Error: Could not access the PRD at [url]. Verify the link and your Notion permissions.

---

## Step 3: Eng Lead Guardrail

If `eng_lead_name` is empty:

- Set `skip_external_actions = true`
- Note: Product Jira Initiative and Slack channel will be skipped

Otherwise set `skip_external_actions = false`.

> The TQA tracking Epic (Step 8b) is **not gated** by this guardrail — it is always created as internal QA
> tracking, regardless of whether Eng Lead is set.

---

## Step 4: Agents 1 & 2 — QA Brief + Adoption Review (parallel)

Use the Task tool to invoke **both agents simultaneously** in a single step.

**Agent 1 — QA Brief:** Run the `qa-brief` command, passing the PRD URL as the argument. The agent reads the PRD,
creates the `📋 QA Brief` subpage in Notion, and returns a summary including the subpage URL and full brief content.

**Agent 2 — Adoption Review:** Invoke the `adoption-review` agent, passing `prd_text` as the input. The agent
evaluates the PRD against the 4 adoption criteria and returns a JSON object.

Wait for both to complete, then store:

- `qa_brief_url` — the Notion subpage URL
- `qa_brief_content` — the full brief text
- `qa_brief_created = true` if successful, `false` if the agent failed
- `adoption_verdict` — `ready`, `needs_clarification`, or `incomplete`
- `adoption_json` — full JSON output

If the adoption review agent fails, set `adoption_verdict = "unknown"` and continue.

---

## Step 4b: Create [ADOPTION REVIEW] Notion Subpage

Create a Notion subpage under the PRD with the adoption review results from Agent 2.

Initialize defaults before branching:

- `adoption_review_url = ""`
- `adoption_page_updated = false`

First, fetch child pages directly from `prd_page_id`. Look for any with "[ADOPTION REVIEW]" in the title.

Apply this guardrail:

- **No existing [ADOPTION REVIEW] page:** create a new subpage
- **One [ADOPTION REVIEW] page found:** update it in place
- **Two or more found:** skip page creation and note in the output:
  > Warning: Found {N} existing [ADOPTION REVIEW] pages. Skipping update - please consolidate manually.

Create or update the subpage with the following format:

**Title:** `[ADOPTION REVIEW] {prd_title}`

**Content:**

```text
## Adoption Review

| Field | Value |
|---|---|
| Verdict | {verdict_emoji} {adoption_verdict} |
| Generated | {current timestamp} |
| Source PRD | {prd_url} |

> {summary from adoption_json}

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

If `questions_for_pm` from `adoption_json` is not empty, add a numbered list:

```text
1. {question} _(Criterion {criterion})_
```

If `questions_for_pm` is empty, add: `None - all criteria are clearly defined in the PRD.`

Where status emojis are: `found` = `✅`, `partial` = `🟡`, `not_found` = `🔴`

Store the Notion subpage URL as `adoption_review_url`.

If `adoption_verdict = "unknown"` (agent failed in Step 4), skip this step.

If page creation fails, set `adoption_review_url = ""` and note the failure in the output. Continue.

---

## Step 5: Agent 3 — Test Suite Generator

Use the Task tool to invoke the `test-suite-generator` agent.

Pass `prd_text` as the input. The agent returns a JSON object.

Parse and store:

- `test_suite_json` — the full JSON output
- `test_suite_title` — the `title` field (e.g., `[DRAFT] Test Suite - Loyalty Revamp`)
- `scenario_count` — `metadata.scenarios_count`
- `requirements` — the `requirements` array

If the agent fails, set `test_suite_created = false` and skip Steps 6 and 7.

---

## Step 6: Create Test Suite Notion Subpage

Fetch child pages from the PRD page ID. Look for pages with "Test Suite" in the title.

Apply this guardrail:

- **No existing Test Suite page:** create a new subpage
- **One `[DRAFT] Test Suite` page found:** update it in place
- **One approved Test Suite page found (title does not start with `[DRAFT]`):** create a new `[DRAFT] Test Suite`
  subpage
- **Multiple Test Suite pages found:** create a new subpage and note the conflict in the output

When creating a new subpage, use `notion-create-pages` with `parent: { "type": "page_id", "page_id": "{prd_page_id}" }`.
Do NOT use `parent_id` — it is not a valid parameter and will cause the page to be created at workspace root.

When updating, use `notion-update-page` with `command: "replace_content"`.

Use the following format for the page content:

---

### Test Suite Page Format

**Title:** `[DRAFT] Test Suite - {feature name}` (from `test_suite_title`)

**Content:**

```text
## Metadata

| Field | Value |
|---|---|
| PRD | {prd_url} |
| Generated | {generated_at from test_suite_json.metadata} |
| Requirements | {requirements_count} |
| Scenarios | {scenarios_count} |

## Requirements

| ID | Description |
|---|---|
| R1 | {description} |
...

## {Test Area Name}

**TC-001** · {type} · {automation}
> Covers: {requirement}

**Given:** {given}
**When:** {when}
**Then:** {then}

---
```

Repeat the scenario block for each scenario. Repeat the area section for each test area.

---

Store the Notion URL as `test_suite_notion_url` and the page ID as `test_suite_page_id`.

Set `test_suite_created = true`.

---

## Step 7: Agent 4 — Suite Reviewer

Use the Task tool to invoke the `suite-reviewer` agent.

Pass the following four sections as the input, separated by `---SECTION---`:

1. `test_suite_notion_url`
2. `prd_text`
3. `qa_brief_content`
4. `test_suite_json` serialized as a string

The agent will:

- Evaluate coverage, BDD quality, and automation tagging
- Remove `[DRAFT]` from the test suite Notion page title via MCP
- Return a JSON object with the verdict and review details

Parse and store:

- `suite_verdict` — `ready` or `review_first`
- `draft_removed` — `true` or `false`
- `suite_review_json` — full JSON output
- `risk_coverage_score` — from `suite_review_json.scores.risk_coverage`
- `bdd_quality_score` — from `suite_review_json.scores.bdd_quality`

After this step, the test suite title in Notion is (if `draft_removed` is true): `Test Suite - {feature name}`

---

## Step 8a: Find or Create Product Jira Initiative

Skip this step if `skip_external_actions = true`. Note the skip.

Look for a Jira project key in the PRD metadata or content (format: `ABC-123` or a project name/link). This is
the **product team's project** (e.g., DATA, INNO, BG — not TQA).

**Normalize the project key before use:**

- If the input looks like a ticket (e.g., `ABC-123`), extract the project key (`ABC`)
- If the input is a Jira URL, parse the project key from the URL
- If the input is a human name (e.g., "Data Platform"), map it to the known project key if possible
- Validate the result matches `/^[A-Z][A-Z0-9_]+$/` — if it does not, set `jira_key = ""`
  and skip the search/create. Note the failure.

**Search before creating:** Query the product project for an existing epic or initiative linked to this PRD:

- Search query: `project = {product_project_key} AND (summary ~ "{prd_title}" OR text ~ "{prd_url}")
  ORDER BY created DESC`
- If found: use the existing issue — store its key as `jira_key` and URL as `jira_url`, then update
  `customfield_12289` (Test Suite link) to `test_suite_notion_url`
- If not found: create a new Initiative with:
  - **Summary:** `QA: {prd_title}`
  - **Issue Type:** Initiative
  - **Description:**

    ```text
    QA Kickoff Pipeline — automated by Claude Code.

    PRD: {prd_url}
    QA Brief: {qa_brief_url}
    Test Suite: {test_suite_notion_url}
    Suite Verdict: {suite_verdict}
    Adoption Review: {adoption_verdict}
    ```

  - Set `customfield_12289` (Test Suite link) to `test_suite_notion_url`

Store:

- `jira_key` — the found or created issue key (e.g., `DATA-1234`)
- `jira_url` — the issue URL

If no product project key is found or Jira fails, set `jira_key = ""` and `jira_url = ""`
and note the failure. Continue.

**Confirm with user and update PRD:**

If `jira_key` is not empty, ask:

> Found/created `{jira_key}` ({jira_url}) — is this the correct Jira initiative for **{prd_title}**? (yes / no)

- If **yes**: update the `"JIRA"` property on the PRD Notion page with `jira_url` using
  `notion-update-page` (only when both `jira_key` and `jira_url` are non-empty).
  Store `jira_prd_updated = true`.
- If **no**: note the mismatch. Do not update the PRD.
  Set `jira_prd_updated = false` and clear `jira_key = ""` and `jira_url = ""`.

If `jira_key` is empty, skip this confirmation and set `jira_prd_updated = false`.

---

## Step 8b: Create TQA Tracking Epic

Always run this step — not gated by `skip_external_actions`.

**Search before creating:** Query TQA for an existing epic already linked to this PRD:

- Search query: `project = TQA AND summary ~ "[QA] {prd_title}" ORDER BY created DESC`
- If found: store its key as `tqa_key` and URL as `tqa_url`. Skip creation.
- If not found: create a new Jira Epic in the **TQA project**:

  - **Project:** `TQA`
  - **Summary:** `[QA] {prd_title}`
  - **Issue Type:** Epic
  - **Assignee:** Look up the Jira account ID for `qe_name` using `lookupJiraAccountId`.
    If not found, leave unassigned and note.
  - **Description:**

    ```text
    QA tracking epic — created by the QA Kickoff Pipeline.

    PRD: {prd_url}
    QA Brief: {qa_brief_url}
    Test Suite: {test_suite_notion_url}
    Product Jira: {jira_url}
    ```

Store:

- `tqa_key` — the found or created epic key (e.g., `TQA-456`)
- `tqa_url` — the epic URL

If TQA epic creation fails, set `tqa_key = ""` and note the failure. Continue.

---

## Step 9: Find or Create Slack Channel

Skip this step if `skip_external_actions = true`. Note the skip.

Search for an existing project Slack channel:

1. **Primary match:** Search for channels whose Topic contains the PRD URL
2. **Fallback:** Search for channels whose name matches `proj-{slugified prd_title}`

If a matching channel is found, store its ID as `slack_channel_id` and name as `slack_channel_name`.

If no matching channel is found, create a new channel:

- **Name:** `proj-{slugified prd_title}` — lowercase, hyphens, max 80 characters
- After creating the channel, set the Topic to the PRD URL

Invite the following members to the channel (look up Slack IDs by name or email if not known):

- Beatriz: `U08UEQ22H7W`
- Giovani (look up by name)
- Lial: `U08HTSEURPH`
- `eng_lead_name` (look up by name or email)
- `pm_name` (look up by name or email)

> **Config note:** Slack User IDs are documented in CLAUDE.md > Configuration. Update both locations when team members change.

Store `slack_channel_id` and `slack_channel_name`. Set `slack_channel_created = true`.

If channel creation or invite fails, note the failure and continue.

**Confirm with user and update PRD:**

If `slack_channel_id` is not empty, ask:

> Found/created `#{slack_channel_name}` — is this the correct Slack channel for **{prd_title}**? (yes / no)

- If **yes**: update the `"Slack #channel"` property on the PRD Notion page with the full
  Slack URL (`https://thanx.slack.com/archives/{slack_channel_id}`) using `notion-update-page`.
  Store `slack_prd_updated = true`.
- If **no**: note the mismatch. Do not update the PRD. Set `slack_prd_updated = false`
  and clear `slack_channel_id = ""` and `slack_channel_name = ""`.

If `slack_channel_id` is empty, skip this confirmation and set `slack_prd_updated = false`.

---

## Step 10: Post Kickoff Message to Project Channel

Skip this step if `skip_external_actions = true` or if `slack_channel_id` is empty or blank.

Post this message to `slack_channel_id` after Steps 7–9 complete (suite review, Jira, and channel are all ready):

```text
👋 QA Kickoff — {prd_title}

The QA pipeline has run for this project.

📄 PRD: {prd_url}
📋 QA Brief: {qa_brief_url}
📊 Adoption Review: {adoption_verdict_emoji} {adoption_verdict}
{adoption_review_line}
🧪 Test Suite: {test_suite_notion_url}
   {suite_verdict_emoji} Suite verdict: {suite_verdict}
   🎯 Coverage: {risk_coverage_score}%
   ✍️ BDD quality: {bdd_quality_score}%
{jira_line}
```

Where:

- `adoption_verdict_emoji` is `🟢` for `ready`, `🟡` for `needs_clarification`, `🔴` for `incomplete`
- `adoption_review_line` is `Page: {adoption_review_url}` (prefix with 3 spaces when rendering) if
  `adoption_review_url` is not empty, empty otherwise
- `suite_verdict_emoji` is `✅` for `ready`, `⚠️` for `review_first`
- `jira_line` is `🎯 Jira: {jira_url}` if product Jira was found/created, empty otherwise

---

## Step 11: Agent 5 — Scorecard Updater

Use the Task tool to invoke the `scorecard-updater` agent.

Pass the following JSON as the input:

```json
{
  "prd_notion_page_id": "{prd_page_id}",
  "qa_brief_created": true,
  "test_suite_created": true,
  "test_suite_verdict": "{suite_verdict}",
  "slack_channel_created": true
}
```

Set each field based on what actually completed in this pipeline run.

The agent returns a JSON object with `notion_page_id` and `properties` to apply.

---

## Step 12: Update PRD Notion Scorecard

Apply the property updates from the `scorecard-updater` agent to the PRD Notion page.

Use the `notion-update-page` MCP tool with `prd_page_id` and the `properties` map from the agent output.

Only apply the fields listed in `properties`. Do not touch formula fields or any other fields.

If the update fails, note the failure and continue.

---

## Step 13: Post Summary to #qa-test-suites-bot

Find the `#qa-test-suites-bot` Slack channel and post:

```text
🚀 QA Kickoff complete — {prd_title}

{suite_verdict_emoji} Suite: {suite_verdict}  {adoption_verdict_emoji} Adoption: {adoption_verdict}
📄 PRD: {prd_url}
📋 QA Brief: {qa_brief_url}
🧪 Test Suite: {test_suite_notion_url}
{jira_line}
{tqa_line}
{channel_line}

Eng Lead: {eng_lead_name} | PM: {pm_name} | QE: {qe_name}
{guardrail_note}
```

Where:

- `jira_line` is `🎯 Jira: {jira_url}` if product Jira was found/created, empty otherwise
- `tqa_line` is `📋 TQA: {tqa_url}` if TQA epic was created, empty otherwise
- `channel_line` is `💬 Channel: #{slack_channel_name}` if found/created, empty otherwise
- `guardrail_note` is `⚠️ Product Jira and Slack channel skipped — Eng Lead not set in PRD.` if
  `skip_external_actions = true`, empty otherwise

---

## Step 14: Output Summary to User

Print the final pipeline summary:

```text
✅ QA Kickoff complete — {prd_title}

📋 QA Brief: {qa_brief_url}
📊 Adoption Review: {adoption_review_url}
🧪 Test Suite: {test_suite_notion_url}
{suite_verdict_emoji} Suite verdict: {suite_verdict}
{adoption_verdict_emoji} Adoption Review: {adoption_verdict}
{jira_line}
{tqa_line}
{channel_line}

PRD fields updated: {jira_prd_updated_line} | {slack_prd_updated_line}
QA Scorecard updated on PRD.
```

Where:

- `jira_prd_updated_line` — `🎯 JIRA link saved` if `jira_prd_updated = true`, `⚠️ JIRA link not saved` otherwise
- `slack_prd_updated_line` — `💬 Slack channel saved` if `slack_prd_updated = true`, `⚠️ Slack channel not saved` otherwise

If any step failed, list failures at the end:

```text
⚠️ The following steps failed and require manual action:
- {step name}: {reason}
```
