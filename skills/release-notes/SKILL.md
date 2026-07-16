---
name: release-notes-from-jira
description: Use this skill when the user asks to generate release notes from specific Jira tickets, issue keys, fix versions, epics, or ticket lists.
---

# Release Notes From Jira

## Purpose
Generate clear, customer-facing or internal release notes from specific Jira tickets.

## When to use
Use this skill when the user provides:
- Jira ticket keys like `ABC-123`, `SDT-456`
- A list of Jira issues
- A fix version
- An epic
- A release name
- A request like "generate release notes from these tickets"

## Workflow
1. Identify the Jira tickets or filter criteria.
   - If no version or tickets are specified, find the first unreleased version after the last release:
     1. Query for the most recent released version:
        ```
        JQL: project = REAL AND fixVersion in releasedVersions() ORDER BY fixVersion DESC
        ```
        Note the version name (e.g. `RTLP 1.6.6`).
     2. Query for the first unreleased version:
        ```
        JQL: project = REAL AND fixVersion in unreleasedVersions() ORDER BY fixVersion ASC
        ```
        Take the first result (e.g. `RTLP 1.6.7`).
     3. Tell the user which version was found (last released + next unreleased) and confirm before proceeding.
2. Use the Atlassian/Jira plugin if available to retrieve ticket details.
3. For each ticket, inspect:
   - Summary
   - Description
   - Acceptance criteria
   - Comments if relevant
   - Status
   - Fix version
   - Labels/components
4. Exclude tickets that are not completed unless the user explicitly asks to include them.
5. Group notes into useful sections:
   - New Features
   - Improvements
   - Bug Fixes
   - Technical / Internal Changes
   - Known Issues, if any
6. Rewrite Jira language into release-note language.
7. Do not expose internal-only implementation details unless the user asks for internal release notes.
8. Do not include any of the untagged tickets
9. Do not publish these notes to a Confluence page

## Output format
Use this structure by default:

# Release Notes

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
- `ABC-123` - Short summary
- `ABC-456` - Short summary
- Release url (for example - https://stratadecision.atlassian.net/projects/REAL/versions/15348/tab/release-report-all-issues for monthly release)

## Style rules
- Be concise.
- Write for product/business readers unless the user asks for technical notes.
- Avoid Jira jargon such as "story", "sub-task", "spike", or "AC".
- Convert developer wording into user-facing impact.
- Use past tense for shipped work.
- If ticket details are missing, say what information is missing.
- Never invent release contents.