---
description: Generate a [DRAFT] Test Suite from a Notion PRD. Creates a Notion subpage, updates the Jira Epic, and notifies the QA Slack channel.
---

# Generate Test Suite from PRD

Generate a structured `[DRAFT] Test Suite` from a Notion PRD using the Thanx QA pipeline.

## Usage

```bash
/test-suite-prd <notion_prd_url> <jira_epic_or_initiative_url> [additional_context...]
```

**Required:**

- `notion_prd_url` - The Notion page URL of the PRD
- `jira_epic_or_initiative_url` - The Jira Epic or Initiative URL (e.g. <https://thanxapp.atlassian.net/browse/TQA-138>)

**Optional:**

- Additional context links: Figma, ADRs, API docs, or free-text notes
- Example: `/test-suite-prd https://notion.so/... https://thanxapp.atlassian.net/browse/ABC-123 https://figma.com/file/...`

## What It Does

1. Reads the PRD and all subpages from Notion
2. Extracts and numbers all requirements from the PRD
3. Generates a complete `[DRAFT] Test Suite` following the Thanx QA pipeline
4. Creates (or updates) the test suite as a subpage under the PRD in Notion
5. Updates the `Test Suite link` field on the Jira Epic or Initiative
6. Notifies the QA Slack channel

The generated test suite is always a **draft**. QA must review, refine, and remove `[DRAFT]` when approved.

---

You are executing the `/qa:test-suite-prd` command.

## Step 1: Parse and Validate Inputs

Arguments: $ARGUMENTS

Parse the arguments:

- **First argument:** Notion PRD URL (required)
- **Second argument:** Jira Epic or Initiative URL (required)
- **Remaining arguments:** Optional additional context (links or free-text notes)

If the Notion PRD URL is missing:
> Error: A Notion PRD URL is required.
> Usage: /qa:test-suite-prd <notion_prd_url> <jira_url> [optional_context...]

If the Jira URL is missing:
> Error: A Jira Epic or Initiative URL is required.
> Usage: /qa:test-suite-prd <notion_prd_url> <jira_url> [optional_context...]

Extract the Jira issue key from the URL (e.g. `TQA-138` from `https://thanxapp.atlassian.net/browse/TQA-138`).

Identify the requester: the name of the user who invoked the command, if available.

---

## Step 2: Read the Jira Issue

Use the Jira MCP to fetch the Epic or Initiative by key.

Extract:

- Issue summary (feature name)
- Description (for additional context)
- Current value of the `Test Suite link` field (if set, note it - the command will overwrite it)
- Jira cloud ID: `7d5d6532-069d-419b-bd1c-d8321b134435`

If the Jira issue is not found or not accessible, continue and surface this as a non-blocking error at the end.

---

## Step 3: Read the PRD from Notion

Fetch the main PRD page using the Notion MCP.

Then fetch each subpage found under the PRD:

- Look for linked child pages in the PRD content
- Fetch each one individually
- Consolidate all content before generating

For each additional context link provided by the user:

- Attempt to fetch it using the Notion MCP (if it is a Notion link) or note it as external
- Mark each source as: Accessible / Not accessible / Partial
- Store these results for Section 10 of the test suite

If the PRD is not accessible, stop and return a clear error:
> Error: Could not access the PRD at [url]. Please verify the link and your Notion permissions.

### 3b: Extract team fields from the PRD

Look for the following properties in the Notion PRD page metadata:

- **Product Manager** - the PM responsible for this feature
- **Eng Lead** - the engineering lead for this project
- **QE** - the QA engineer assigned to this project

These fields are optional. If any are missing or empty, store them as empty strings and continue - do not stop or surface an error.

### 3c: Analyze images from the PRD

After fetching the PRD content, look for image URLs embedded in the content. They appear in the format `![](url)`.

For each image URL found:

1. Use the Bash tool to download the image to a temporary file:

```bash
curl -s -o /tmp/notion_prd_image_N.ext "IMAGE_URL"
```

Replace `N` with a sequential number and `ext` with the file extension from the URL (png, jpg, gif, etc.).

2. Use the Read tool to analyze the downloaded image and extract all relevant information:
   - UI layouts, flows, or wireframes
   - Labels, field names, pricing structures, or configuration examples
   - Diagrams, tables, or any visible text
   - Anything that describes expected behavior or business rules

3. Store the analysis for each image as additional context to use in Step 4.

4. After all images are analyzed, delete the temporary files:

```bash
rm -f /tmp/notion_prd_image_*.{png,jpg,gif,webp}
```

If an image URL has expired or cannot be downloaded, skip it silently and continue - do not stop the command. Note it in Section 10 as "Image not accessible (expired URL)".

If no images are found in the PRD, continue without this step.

---

## Step 4: Generate the Test Suite

Using the full PRD content (main page + subpages + image analyses from Step 3c + accessible additional context), generate the test suite following the structure below.

Apply these writing rules throughout:

- Use plain English. Avoid jargon. Define technical terms when necessary.
- Write short, direct sentences.
- Do not use the em dash character. Use a hyphen or rewrite the sentence.
- Use quotation marks only for literal UI strings or direct quotes from the PRD.
- Use markdown consistently: headers, bullet lists, tables.
- Distinguish clearly between: user action / system behavior / expected outcome.
- Keep scenario titles descriptive but short.
- Use the same tense throughout all steps in a scenario.
- Never invent behavior. If something is unclear, document it as a question in Section 9.

---

### Output Structure

Generate the following sections in order. Do not skip any section marked [REQUIRED].

---

#### [DRAFT] Metadata Header [REQUIRED]

| Field | Value |
|---|---|
| Status | DRAFT - Pending QA Review |
| Generated by | {requester name or "Claude Code - /qa:test-suite-prd"} |
| Generated at | {current timestamp in ISO format} |
| Source PRD | {Notion PRD URL} |
| Jira Epic/Initiative | {Jira URL} |
| PRD last updated | {Notion last edited date if available} |

> This test suite is a starting point. It requires QA review before any scenario is executed.
> No scenario is considered approved until [DRAFT] is removed from the title.

---

#### Section 1: Overview [REQUIRED]

**What this feature does:**
{1 to 3 sentences. Plain English. No jargon.}

**In scope:**

- {list each item explicitly}

**Out of scope:**

- {list each item explicitly. If nothing is excluded, state what is intentionally not tested and why.}

**Integrations involved:**

- {e.g. Toast, Olo, Valutec, internal APIs, Dashboard, Segment. Only list what is relevant.}

---

#### Section 2: PRD Requirements [REQUIRED]

List every requirement extracted from the PRD and its subpages.
Include both explicit requirements (stated directly) and implicit requirements (behavior implied by the PRD).
Assign a unique ID to each.

| ID | Requirement | Type | Source |
|---|---|---|---|
| REQ-01 | {description} | Explicit / Implicit | {PRD section or subpage name} |

Marking rules:

- If a requirement is ambiguous or incomplete, add a warning before the description and add it to Section 9.
- If a requirement depends on a third party, add a link marker before the description.

---

#### Section 3: Preconditions [REQUIRED]

List all setup required before executing any test scenario.
Be specific. A tester should be able to prepare the environment from this section alone.

- **Environment:** {Sandbox / Staging / Production}
- **Feature flags:** {flag name + required state}
- **Merchant or account setup:** {specific configuration}
- **Test data required:** {users, orders, rewards, cards, balances}
- **Third-party dependencies:** {e.g. Olo sandbox account, Valutec test cards}
- **API keys or credentials:** {describe what is needed, without exposing values}

If a precondition cannot be met in the test environment, note it here and add it to Section 9.

---

#### Section 4: Test Scenarios [REQUIRED]

Each scenario must:

- Reference at least one requirement ID from Section 2
- Be written in BDD format (Given / When / Then)
- Distinguish clearly between user action, system behavior, and expected outcome

Minimum coverage:

- All happy paths for each integration or platform variant
- At least one negative or failure scenario per critical flow
- Platform-specific behavior (e.g. iOS vs Android, Toast vs Olo) as separate scenarios when behavior differs

For each scenario use this structure:

#### Scenario [N]: {Descriptive title}

**Covers:** REQ-XX, REQ-XX
**Objective:** {one sentence describing what this scenario validates}
**Priority:** High / Medium / Low
**Test type:** Functional / API / UI / Integration / Mobile

**Given** {initial state}
**When** {user or system action}
**Then** {expected result}

**Test data:**

| Field | Value |
|---|---|
| {field} | {value or description} |

**Notes:** {any clarification, environment-specific behavior, or known variation}

---

#### Section 5: PRD Coverage Matrix [REQUIRED]

Verify that every requirement in Section 2 has at least one test scenario.
If a requirement has no scenario, mark it UNCOVERED and add it to Section 9.

| Requirement ID | Requirement Summary | Covered by | Coverage Status |
|---|---|---|---|
| REQ-01 | {short summary} | Scenario 1, Scenario 3 | Covered |
| REQ-02 | {short summary} | - | UNCOVERED - add to open questions |

> A test suite with any UNCOVERED requirement cannot be approved.
> The QA reviewer must either add a scenario or document why coverage is not possible.

---

#### Section 6: Edge Cases [REQUIRED]

Cover at minimum:

- Boundary values: max and min limits, caps, thresholds, zero values
- Error states: API failures, timeouts, invalid inputs, missing fields
- Unexpected user behavior: rapid actions, back navigation mid-flow, repeated submissions
- Missing or incomplete configuration
- Multi-merchant or multi-location scenarios
- Integration-specific variations (e.g. Toast behavior vs Olo)
- Time-based conditions: expiration, delays, timeouts
- Permission and role differences

Use the same scenario structure as Section 4, or a variation table when multiple inputs map to the same flow.

---

#### Section 7: Regression Scope [REQUIRED]

List all areas that could be affected by this change, with specific risks.

| Area | Specific Risk | Priority |
|---|---|---|
| UI | {what visual or interaction behavior could break} | High / Medium / Low |
| API | {which endpoints or contracts could be affected} | |
| Dashboard | {which admin or merchant views could be impacted} | |
| Ordering flows | {which ordering scenarios could regress} | |
| Reward engine | {points, rewards, caps, redemption flows} | |
| Integrations | {which third-party connections could be affected} | |
| Activity logs | {what tracking or audit trails could change} | |
| Data exports | {which exports or reports could be impacted} | |
| Analytics and events | {Segment events or tracking changes} | |

Only list areas with a plausible risk. Omit areas that are not affected.

---

#### Section 8: Exploratory Testing [REQUIRED]

Document at least 3 exploratory scenarios. These go beyond scripted test cases.
Each must describe a specific risk area, not a generic instruction to explore.

**Exploratory Scenario 1: {title}**
**Why this matters:** {specific risk or uncertainty}
**What to do:** {free-form steps or areas to investigate}
**Watch for:** {specific failure symptoms, inconsistencies, or unexpected behaviors}

---

#### Section 9: Open Questions, Assumptions and Limitations [REQUIRED]

This section must never be empty.
Include any ambiguous requirement from Section 2.
Include any precondition that could not be confirmed.

| # | Description | Type | Expected Owner | Status |
|---|---|---|---|---|
| 1 | {question, assumption, or limitation} | Question / Assumption / Limitation | {real name from PRD if available, otherwise: PM / Engineering / QA} | Open |

When assigning Expected Owner, use the real names extracted from the PRD in Step 3b if available:

- Product questions - Product Manager name (or "PM" if not found)
- Technical questions - Eng Lead name (or "Engineering" if not found)
- Coverage gaps - QE name (or "QA" if not found)

- **Question:** information missing from the PRD that affects coverage
- **Assumption:** something treated as true because it was not specified
- **Limitation:** a constraint that reduces coverage (e.g. third party not testable in sandbox)

---

#### Section 10: Additional Context Used [REQUIRED]

Document every source beyond the PRD, whether accessible or not.

| Source | Type | Accessible | Used for | Notes |
|---|---|---|---|---|
| {link or description} | Figma / ADR / API docs / Image / Other | Yes / No / Partial | {what it informed} | {e.g. "endpoint unclear, added to open questions"} |

If no additional context was provided:
> No additional context was provided. This test suite was generated from the PRD and its subpages only.

---

#### Section 11: Test Evidence [Optional]

List the evidence testers should collect during execution:

- Before and after screenshots for UI changes
- API request and response logs
- Console errors or warnings
- Activity or audit history
- Merchant dashboard state
- Segment or analytics events fired

---

#### Section 12: Sign-Off [REQUIRED]

| Role | Name | Date | Status |
|---|---|---|---|
| Product Manager | {Product Manager from PRD, or blank if not found} | | Pending |
| Engineering Lead | {Eng Lead from PRD, or blank if not found} | | Pending |
| QA | {QE from PRD, or blank if not found} | | Pending |

> All three roles must sign off before [DRAFT] is removed from the title.
> QA is the final approver for test coverage completeness.

---

## Step 5: Create or Update the Notion Subpage

After generating the test suite content, write it to Notion as a subpage under the PRD.

### 5a: Check for existing subpages first

Before writing anything:

1. Fetch the PRD page again and list all child pages found in its content
2. Look for any pages whose title contains `Test Suite` (case-insensitive)
3. Classify each match as:
   - `[DRAFT]` version (title starts with `[DRAFT]`)
   - Approved version (no `[DRAFT]` in title)

Then apply the following decision rules:

| What was found | Action |
|---|---|
| Nothing | Create new subpage under PRD (use PRD page ID as parent) |
| One `[DRAFT]` version | Warn the user: "A [DRAFT] Test Suite already exists. It will be updated in place." Then update the existing page content using `replace_content`. Do not create a new page. |
| One approved version (no `[DRAFT]`) | Create a new `[DRAFT]` version alongside it. Do not touch the approved page under any circumstances. |
| Multiple pages matching `Test Suite` | Stop. Ask the user: "Found multiple Test Suite pages under this PRD. Please clarify which should be kept before running this command again." Do not write anything. |
| Any other ambiguous state | Stop. Describe what was found. Do not write anything. |

### 5b: Write the subpage

**If creating a new page** (no existing test suite, or approved-only exists):

- Always use the PRD page ID as the `parent_id`. Never use any other page as the parent.
- The title must be: `[DRAFT] Test Suite - {feature name from PRD title}`
- Write the full generated content from Step 4 as the page body
- Do not delete or modify any other existing subpage of the PRD

**If updating an existing `[DRAFT]` page** (update in place):

- Use `replace_content` on the existing `[DRAFT]` page ID
- Replace the entire content with the newly generated test suite
- Do not change the page title or move the page
- Do not create a new page

In both cases, capture the Notion page URL for use in Steps 6 and 7.

---

## Step 6: Update the Jira Epic or Initiative

Update the `Test Suite link` field on the Jira issue with the Notion subpage URL.

Always overwrite this field with the URL of the newly created or updated `[DRAFT]` page, regardless of any previous value. Do not skip this step because a value already exists.

The field ID for `Test Suite link` is `customfield_12289`. Use this directly - do not fetch edit metadata to discover it.

Update the field using the Jira MCP `editJiraIssue` tool with:

- `issueIdOrKey`: the Jira issue key
- `fields`: `{ "customfield_12289": "<Notion URL>" }`

If the field is not found or cannot be updated:

- Do not stop the command
- Surface this as a non-blocking error at the end:

> Warning: Could not update Jira field "Test Suite link". Please update it manually: [Notion URL]

Failure to update Jira does not invalidate the Notion test suite.

---

## Step 7: Notify the QA Slack Channel

Send a notification to the QA Slack channel after the Notion subpage is created or updated.

Use the Bash tool to POST to the Incoming Webhook:

- Webhook URL: Read from the `QA_SLACK_WEBHOOK_URL` environment variable. If not set, skip this step and surface it as a non-blocking error.

Build the message text using Slack mrkdwn format:

```text
*[DRAFT] Test Suite {created or updated}*

*Feature:* {PRD title}
*Generated by:* {requester}
*PRD:* <{Notion PRD link}|{PRD title}>
*Test Suite:* <{Notion subpage link}|{Test Suite title}>
*Jira:* <{Jira link}|{Jira issue key}>

Please review the open questions and assign owners before approving.
```

If the QE field was extracted from the PRD in Step 3b, append a mention at the end of the message:

- If QE name contains "Beatriz" (case-insensitive) - append: `<@U08UEQ22H7W> this one's yours to review.`
- If QE name contains "Giovani" (case-insensitive) - append: `<@U08UEQ5MJDA> this one's yours to review.`
- If QE name is empty or does not match either - do not append any mention

Use `jq` to safely encode the JSON payload, then pipe to curl:

```bash
jq -n --arg text "<message text with all values substituted>" '{"text": $text}' \
  | curl -s -X POST -H 'Content-type: application/json' -d @- \
  "$QA_SLACK_WEBHOOK_URL"
```

A successful response from the webhook is `ok`.

Use Slack's `<url|label>` format for all links to prevent URL parsing issues.

Indicate in the message whether the suite was created or updated.

If the Slack notification fails, do not stop or roll back the command. Surface it as a non-blocking error at the end.

---

## Step 8: Report Back to the User

After completing all steps, show a summary:

```text
Test Suite generated successfully.

Notion subpage: {link}
Jira updated: Yes / No (with reason if No)
Slack notification: Sent / Failed (with reason if Failed)

Open questions requiring owner assignment: {count}
UNCOVERED requirements: {count} (if any)
```

If there are UNCOVERED requirements, list them explicitly and remind the user they must be addressed before the suite can be approved.

If any non-blocking errors occurred (Jira or Slack), list them clearly so the user can follow up manually.
