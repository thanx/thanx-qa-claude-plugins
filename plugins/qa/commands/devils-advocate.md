---
description: Stress-test a QA strategy, test plan, or coverage decision before committing to it. Surfaces gaps, assumptions, and risk areas.
---

# Devils Advocate (QA)

Run structured critical analysis on QA strategies, test plans, release readiness
assessments, or coverage decisions before committing resources.

## Usage

```bash
/qa:devils-advocate [proposal text or description]
```

**Examples:**

```bash
/qa:devils-advocate "We should skip regression testing for this hotfix"
/qa:devils-advocate "Our test suite covers 80% of critical paths, we are ready to release"
/qa:devils-advocate                    # Prompts for input
```

Arguments: $ARGUMENTS

## Step 1: Load Proposal

If `$ARGUMENTS` is non-empty: treat as the proposal text.
If `$ARGUMENTS` is empty: prompt the user to paste the proposal or describe the QA decision.

## Step 2: Summarize the Proposal

One-paragraph statement. No editorial. State:

- What is being proposed (test plan, coverage decision, release call, process change)
- What outcome it targets
- What is at risk if the proposal is wrong

Present the summary and confirm understanding before proceeding.

## Step 3: Identify Logical Fallacies

Scan for reasoning errors common in QA decisions:

| Category | Fallacy | Where It Appears | Why Problematic |
|----------|---------|-----------------|-----------------|
| **Coverage** | "We tested the happy path so it works" / Survivorship bias | [quote] | [explanation] |
| **Risk** | "It worked in staging so it will work in production" / False equivalence | [quote] | [explanation] |
| **Process** | "We have never had a bug there" / Appeal to history | [quote] | [explanation] |
| **Resources** | "We do not have time to test more" / Sunk cost / Planning fallacy | [quote] | [explanation] |

Only output categories where fallacies are found. If none detected, state it and proceed.

## Step 4: Challenge Core Assumptions

Identify the 3 assumptions the proposal depends on. For each:

```text
Assumption: [X must be true for this QA decision to be safe]
Evidence for: [what supports this assumption today]
Failure scenario: [plausible situation where X is false]
Impact if false: [what breaks -- escaped defects, customer impact, rollback cost]
```

## Step 5: Missing Test Perspectives

What is not being tested or considered?

- **User segments:** Which user types, device types, or merchant configurations are untested?
- **Integration boundaries:** Which API contracts, third-party dependencies, or data flows are uncovered?
- **Temporal effects:** What about race conditions, timezone edge cases, or data migration timing?
- **Regression surface:** What existing features could break as a side effect?
- **Non-functional:** Performance, accessibility, security -- are these in scope?

List concretely. If none identifiable, say so.

## Step 6: Risk Assessment

| Risk Area | Likelihood | Impact | Current Mitigation | Gap |
|-----------|-----------|--------|-------------------|-----|
| [Escaped defect in X] | [High/Med/Low] | [High/Med/Low] | [Current test coverage] | [What is missing] |
| [Regression in Y] | ... | ... | ... | ... |
| [Data integrity in Z] | ... | ... | ... | ... |

## Step 7: Steelman the Counterargument

Write the strongest possible argument AGAINST this QA decision. The version a
senior engineer or product manager would make if challenging the QA team.

```text
Steelman argument: [2-4 sentences. Direct assertion.]
What makes it strong: [Why this objection is hard to dismiss]
```

## Step 8: Synthesis

```text
## Devils Advocate Analysis: [Proposal Title]

### Top 3 Concerns
1. [Most serious gap or risk. Specific, not general.]
2. [Second concern]
3. [Third concern]

### Recommended Additions to Test Plan
- [Specific test case or coverage area to add]
- [Second addition]

### Questions to Answer Before Proceeding
1. [Question that if unanswered could lead to escaped defects]
2. [Second question]
3. [Third question]

### Overall Assessment
[Pass / Conditional Pass / Not Ready]

Pass: No critical coverage gaps. Proceed with the plan.
Conditional Pass: Address the top concern before proceeding. One gap needs closing.
Not Ready: Significant coverage gap or untested risk area. Revise the plan.
```

## Important Notes

- This command is for internal QA team use -- stress-testing your own decisions
- The goal is to find gaps before they become escaped defects, not to block progress
- A "Not Ready" verdict means the plan needs revision, not that the feature is bad
- Run this before release readiness calls to surface risks early
- Works on any QA artifact: test plans, coverage reports, release go/no-go decisions, automation strategy proposals
