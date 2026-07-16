---
name: release-notes-from-jira
description: Use this skill when the user asks to generate release notes from specific Jira tickets, issue keys, fix versions, epics, or ticket lists.
---

# Release Notes From Jira

## Purpose
Generate clear, customer-facing or internal release notes from Jira tickets. Works with any Jira project.

## When to use
Use this skill when the user provides:
- Jira ticket keys like `ABC-123`, `SDT-456`
- A list of Jira issues
- A fix version
- An epic
- A release name
- A request like "generate release notes from these tickets"

## Workflow

### Step 1 — Gather context
If the user has not already provided them, ask:
1. **Jira project key** — e.g. `REAL`, `JAZZ`, `STAR` (check the URL of their Jira board if unsure)
2. **Fix version or ticket list** — specific version name, or "auto-detect" to find the next unreleased version
3. **Audience** — customer-facing (default) or internal?

If the user says "auto-detect" or provides no version:
1. Query for the most recent released version:
   ```
   JQL: project = <PROJECT_KEY> AND fixVersion in releasedVersions() ORDER BY fixVersion DESC
   ```
2. Query for the first unreleased version:
   ```
   JQL: project = <PROJECT_KEY> AND fixVersion in unreleasedVersions() ORDER BY fixVersion ASC
   ```
3. Tell the user: "Last released: `X`. Next unreleased: `Y`. Shall I generate notes for `Y`?" and wait for confirmation.

### Step 2 — Fetch tickets
Use the Atlassian Jira plugin with:
```
JQL: project = <PROJECT_KEY> AND fixVersion = "<version name>" ORDER BY issuetype ASC, key ASC
```
Fields to inspect per ticket:
- Summary, Description, Acceptance criteria, Comments (if relevant)
- Status, Fix version, Labels, Components, Issue type

### Step 3 — Filter and group
- Exclude tickets that are not completed unless the user explicitly asks to include them.
- Group into sections:
  - New Features
  - Improvements
  - Bug Fixes
  - Technical / Internal Changes
  - Known Issues (incomplete tickets worth flagging)

### Step 4 — Write the notes
- Rewrite Jira language into plain release-note language.
- Do not expose internal implementation details unless the user asked for internal notes.
- Do not include untagged/unrelated tickets.
- Do not publish to Confluence unless explicitly asked.

## Output format

```
# Release Notes — <Version> (<Date>)

## New Features
- ...

## Improvements
- ...

## Bug Fixes
- ...

## Technical / Internal Changes
- ...

## Known Issues
- ...

## Source Tickets
- `ABC-123` — Short summary
- `ABC-456` — Short summary
- Release URL: https://<site>.atlassian.net/projects/<PROJECT>/versions/<id>/tab/release-report-all-issues
```

## Style rules
- Be concise.
- Write for product/business readers unless the user asks for technical notes.
- Avoid Jira jargon: no "story", "sub-task", "spike", "AC", "epic".
- Convert developer wording into user-facing impact.
- Use past tense for shipped work.
- If ticket details are missing, say what is missing.
- Never invent release contents.
