---
description: Generate a QA Brief from a Notion PRD. Identifies risk areas, integrations to test, and open questions. Creates a 📋 QA Brief subpage under the PRD.
---

# Generate QA Brief from PRD

Analyze a Notion PRD and generate a `📋 QA Brief` subpage that summarizes what the QE needs to know before writing test scenarios.

The QA Brief is a planning artifact — not a test suite. It identifies risk areas, external integrations that need test coverage, and open questions the team must resolve before testing begins.

This is a QA-internal command. It does not update Jira, send Slack notifications, or involve any third parties.

## Usage

```bash
/qa:qa-brief <notion_prd_url>
```

**Required:**

- `notion_prd_url` - The Notion page URL of the PRD to analyze

## What It Does

1. Reads the PRD from Notion (main page + subpages)
2. Extracts team fields (PM, Eng Lead, QE) from PRD metadata
3. Analyzes images embedded in the PRD
4. Identifies risk areas, integrations, and open questions
5. Creates (or updates) a `📋 QA Brief` subpage under the PRD in Notion

---

You are executing the `/qa:qa-brief` command.

## Step 1: Parse and Validate Input

Arguments: $ARGUMENTS

Parse the arguments:

- **First argument:** Notion PRD URL (required)

If the Notion PRD URL is missing:
> Error: A Notion PRD URL is required.
> Usage: /qa:qa-brief <notion_prd_url>

Identify the requester: the name of the user who invoked the command, if available.

---

## Step 2: Read the PRD from Notion

Fetch the main PRD page using the Notion MCP.

Then fetch each subpage found under the PRD:

- Use the Notion MCP to fetch child pages directly from the PRD page ID (not just linked pages in the content body — Notion child pages can exist without being linked inline, e.g. if created via the sidebar)
- Fetch each child page individually
- Consolidate all content before analyzing

If the PRD is not accessible, stop and return a clear error:
> Error: Could not access the PRD at [url]. Please verify the link and your Notion permissions.

### 2a: Extract team fields from the PRD

Look for the following properties in the Notion PRD page metadata:

- **Product Manager** - the PM responsible for this feature
- **Eng Lead** - the engineering lead for this project
- **QE** - the QA engineer assigned to this project

These fields are optional. If any are missing or empty, store them as empty strings and continue.

### 2b: Analyze images from the PRD

After fetching the PRD content, look for image URLs embedded in the content. They appear in the format `![](url)`.

For each image URL found:

1. Extract the file extension from the image URL. Only proceed if the extension is one of: `png`, `jpg`, `jpeg`, `gif`, `webp`, `svg`. If the extension is not in this list, skip the image and note: "Skipped image N - unsupported file type: {ext}".

1. Use the Bash tool to download the image to a temporary file with safety limits:

```bash
curl -s --max-time 30 --max-filesize 10485760 -o /tmp/notion_prd_image_N.ext "IMAGE_URL"
```

Replace `N` with a sequential number and `ext` with the validated file extension. The flags prevent hanging on slow servers (30s timeout) and reject files larger than 10 MB.

1. Use the Read tool to analyze the downloaded image and extract all relevant information:
   - UI layouts, user flows, or wireframes that reveal testable behavior
   - Labels, error states, edge cases, or boundary conditions visible in mockups
   - Integration touchpoints, data flows, or system boundaries shown in diagrams

1. Store the analysis for each image as additional context to use in Step 3.

1. After all images are analyzed (or if any download/analysis fails), always delete the temporary files:

```bash
rm -f /tmp/notion_prd_image_*.{png,jpg,jpeg,gif,webp,svg}
```

Always run this cleanup, even if image analysis was interrupted by an error or timeout.

If an image URL has expired or cannot be downloaded, note it as "Image not accessible (expired URL)" and continue. If no images are found, continue without this step.

---

## Step 3: Analyze the PRD for QA Risks

Using the full PRD content (main page + subpages + image analyses), generate the five sections of the QA Brief.

Apply these writing rules:

- Use plain English. Write short, direct sentences.
- Do not use the em dash character (—). Use a hyphen or rewrite the sentence.
- Be specific. Generic risks (e.g., "performance issues") are not useful. Name the specific flow or data path at risk.
- Never invent behavior. If something is unclear, document it as an open question.
- Distinguish clearly between: what is stated in the PRD / what can be inferred / what is missing.

### Section 1: Feature Summary

Write 2-3 sentences summarizing what this feature does and who it is for. Be specific — avoid marketing language. State what changes for the customer or merchant, and what system behavior changes.

### Section 2: Risk Areas

Identify 3-7 areas that carry the highest testing risk. For each risk:

- **Name** the risk area (e.g., "Reward deduction on failed orders", "Backward compatibility with existing loyalty configs")
- **Explain** why it is risky from a QA perspective (data integrity, state transitions, integrations, permissions)
- **Suggest** the testing approach (e.g., "Requires E2E test with real payment flow", "Needs regression on existing merchant configs")

Focus on:

- Data integrity and state transitions
- Integration points with external systems (Olo, Braze, Apple Pay, Stripe, etc.)
- Edge cases in user flows (empty states, error states, timeouts, retries)
- Permissions and multi-tenant behavior (merchant A should not see merchant B data)
- Backward compatibility with existing merchant configurations
- Areas mentioned in the PRD as "out of scope" (often become bugs at the boundary)

### Section 3: Integrations to Test

List every external system, API, or internal service that this feature touches. For each integration:

- **Name** - What the system is
- **How it is used** - What this feature sends to or receives from it
- **Testing note** - Sandbox available / Requires credentials / Already covered in E2E suite / No test environment available

If no external integrations are mentioned in the PRD, state that explicitly.

### Section 4: Open Questions

List questions the QE needs answered before writing or executing tests. These must be specific and actionable — not general curiosity.

Format each as a question with a designated owner:

- Questions for **PM** - about expected behavior, edge cases, acceptance criteria, or business rules
- Questions for **Eng Lead** - about implementation details, error handling, retry logic, or data contracts
- Questions for **QE** - internal reminders or dependencies on other test work (e.g., "Check if test data for X already exists in sandbox")

If the PRD is comprehensive and no open questions exist, write: "No open questions identified - PRD covers all relevant QA concerns."

### Section 5: QA Recommendation

In 2-4 sentences, state the overall QA approach for this feature:

- What type of testing is most critical (E2E, API, regression, exploratory)?
- What is the recommended test entry point (UI, API, or both)?
- Are there any blockers to starting test execution?
- Any dependency on test data, sandbox environments, or other teams?

---

## Step 4: Generate the QA Brief

Using the analysis from Step 3, format the QA Brief following the structure below.

---

### Output Structure

---

#### QA Brief Metadata Header

| Field | Value |
|---|---|
| Status | DRAFT |
| Generated by | {requester name or "Claude Code - /qa:qa-brief"} |
| Generated at | {current timestamp in ISO format} |
| Source PRD | {Notion PRD URL} |
| PRD last updated | {Notion last edited date if available} |

> This brief is a starting point. The QE should validate open questions with the PM and Eng Lead before finalizing the test plan.

---

#### Feature Summary

{2-3 sentences. What the feature does. Who it is for. What changes.}

---

#### Risk Areas

| # | Risk Area | Why It Is Risky | Testing Approach |
|---|---|---|---|
| 1 | {name} | {reason} | {approach} |
| 2 | ... | ... | ... |

---

#### Integrations to Test

| Integration | How It Is Used | Testing Note |
|---|---|---|
| {name} | {usage} | {note} |

---

#### Open Questions

| # | Question | Owner |
|---|---|---|
| 1 | {question} | {PM / Eng Lead / QE} |

---

#### QA Recommendation

{2-4 sentences. Test type priority, entry point, blockers, dependencies.}

---

#### Sign-Off

| Role | Name | Status |
|---|---|---|
| QE | {QE from PRD metadata, or leave blank} | Pending |
| PM | {PM from PRD metadata, or leave blank} | Pending |
| Eng Lead | {Eng Lead from PRD metadata, or leave blank} | Pending |

---

## Step 5: Create or Update the QA Brief Subpage in Notion

Before creating, fetch child pages directly from the PRD page ID using the Notion MCP and look for any with "QA Brief" in the title. Do not rely on links found in the PRD content — a prior brief page may exist as a sidebar child without being linked inline.

**If no existing QA Brief page is found:**
Create a new Notion subpage under the PRD with:

- Title: `📋 QA Brief - {PRD title}`
- Content: the full brief generated in Step 4
- Parent: the PRD page

**If exactly one QA Brief page is found:**
Update it in place - replace the full content with the new brief. Do not create a new page.

**If two or more QA Brief pages are found:**
Stop and report:
> Found {N} existing QA Brief pages under this PRD: {list titles and URLs}
> Please delete or consolidate them manually before running this command again.

After creating or updating the subpage, confirm to the user:
> QA Brief created: {Notion subpage URL}
> Risk areas identified: {N}
> Integrations to test: {N}
> Open questions: {N} (or "None")
