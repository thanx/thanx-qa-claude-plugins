---
description: Run the full QA kickoff pipeline for a Notion PRD. Generates QA Brief, Adoption Review, Test Suite, Suite Review, and Scorecard Update. Creates Jira Initiative and Slack channel when Eng Lead is set.
---

# QA Kickoff Pipeline

Orchestrate the full QA kickoff for a project: five agents, with QA Brief and Adoption Review running in parallel, followed by Test Suite generation, Suite Review, and Scorecard Update.

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
- Note: Jira Initiative and Slack channel creation will be skipped

Otherwise set `skip_external_actions = false`.

---

## Step 4: Agents 1 & 2 — QA Brief + Adoption Review (parallel)

Use the Task tool to invoke **both agents simultaneously** in a single step.

**Agent 1 — QA Brief:** Run the `qa-brief` command, passing the PRD URL as the argument. The agent reads the PRD, creates the `📋 QA Brief` subpage in Notion, and returns a summary including the subpage URL and full brief content.

**Agent 2 — Adoption Review:** Invoke the `adoption-review` agent, passing `prd_text` as the input. The agent evaluates the PRD against the 4 adoption criteria and returns a JSON object.

Wait for both to complete, then store:

- `qa_brief_url` — the Notion subpage URL
- `qa_brief_content` — the full brief text
- `qa_brief_created = true` if successful, `false` if the agent failed
- `adoption_verdict` — `ready`, `needs_clarification`, or `incomplete`
- `adoption_json` — full JSON output

If the adoption review agent fails, set `adoption_verdict = "unknown"` and continue.

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
- **One approved Test Suite page found (title does not start with `[DRAFT]`):** create a new `[DRAFT] Test Suite` subpage
- **Multiple Test Suite pages found:** create a new subpage and note the conflict in the output

Create or update the subpage using `notion-create-pages` or `notion-update-page`. Use the following format for the page content:

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

## Step 8: Create Jira Initiative

Skip this step if `skip_external_actions = true`. Note the skip.

Look for a Jira project key in the PRD metadata or content (format: `ABC-123` or a project name/link).

Create a Jira Initiative (not Epic) with:

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

- `jira_key` — the created issue key (e.g., `BG-1234`)
- `jira_url` — the issue URL

If Jira creation fails or no project key is found, set `jira_key = ""` and note the failure in the output. Continue.

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

Store `slack_channel_id` and `slack_channel_name`. Set `slack_channel_created = true`.

If channel creation or invite fails, note the failure and continue.

---

## Step 10: Post Kickoff Message to Project Channel

Skip this step if `skip_external_actions = true`.

Post this message to `slack_channel_id`:

```text
👋 QA Kickoff — {prd_title}

The QA pipeline has run for this project.

📄 PRD: {prd_url}
📋 QA Brief: {qa_brief_url}
✅ Adoption Review: {adoption_verdict_emoji} {adoption_verdict}
{jira_line}

Next step: review the QA Brief and resolve open questions with the PM and Eng Lead before the kickoff meeting.
```

Where:

- `adoption_verdict_emoji` is `🟢` for `ready`, `🟡` for `needs_clarification`, `🔴` for `incomplete`
- `jira_line` is `🎯 Jira: {jira_url}` if Jira was created, empty otherwise

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

## Step 13: Post Final Message to Project Channel

Skip this step if `skip_external_actions = true`.

Post this message to `slack_channel_id`:

```text
🧪 Test Suite ready — {prd_title}

{suite_verdict_emoji} Suite verdict: {suite_verdict}
📝 Test Suite: {test_suite_notion_url}
🎯 Coverage: {risk_coverage_score}%  ✍️ BDD quality: {bdd_quality_score}%
{missing_note}
```

Where:

- `suite_verdict_emoji` is `✅` for `ready`, `⚠️` for `review_first`
- `missing_note` is empty for `ready`, or `⚠️ {N} risk areas have coverage gaps — see the test suite for details.` for `review_first`

---

## Step 14: Post Summary to #qa-test-suites-bot

Find the `#qa-test-suites-bot` Slack channel and post:

```text
🚀 QA Kickoff complete — {prd_title}

{suite_verdict_emoji} Suite: {suite_verdict}  {adoption_verdict_emoji} Adoption: {adoption_verdict}
📄 PRD: {prd_url}
📋 QA Brief: {qa_brief_url}
🧪 Test Suite: {test_suite_notion_url}
{jira_line}
{channel_line}

Eng Lead: {eng_lead_name} | PM: {pm_name} | QE: {qe_name}
{guardrail_note}
```

Where:

- `jira_line` is `🎯 Jira: {jira_url}` if created, empty otherwise
- `channel_line` is `💬 Channel: #{slack_channel_name}` if found/created, empty otherwise
- `guardrail_note` is `⚠️ Jira and Slack channel skipped — Eng Lead not set in PRD.` if `skip_external_actions = true`, empty otherwise

---

## Step 15: Output Summary to User

Print the final pipeline summary:

```text
✅ QA Kickoff complete — {prd_title}

📋 QA Brief: {qa_brief_url}
🧪 Test Suite: {test_suite_notion_url}
{suite_verdict_emoji} Suite verdict: {suite_verdict}
{adoption_verdict_emoji} Adoption Review: {adoption_verdict}
{jira_line}
{channel_line}

QA Scorecard updated on PRD.
```

If any step failed, list failures at the end:

```text
⚠️ The following steps failed and require manual action:
- {step name}: {reason}
```
