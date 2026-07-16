# Skills

Claude Code skills that automate common team operations. Each skill is a `SKILL.md` file that Claude reads to guide you through a workflow step by step.

## Available skills

| Skill | Folder | What it does |
|-------|--------|-------------|
| Capacity Planning | `capacity-planning/` | Generate a PI Planning Board HTML for any team — team profiles, sprint dates, PTO, Jira stories, live collaboration |
| Release Notes | `release-prep/release-notes/` | Generate customer-facing or internal release notes from a Jira fix version |
| Deployment Excel | `release-prep/deployment-excel/` | Generate a deployment `.xlsx` file from a Jira fix version |

## How to install

Skills live in your local Claude Code skills folder. Copy any skill folder there and Claude picks it up automatically.

### Windows

```bat
xcopy "skills\capacity-planning" "%USERPROFILE%\.claude\skills\capacity-planning" /E /I
xcopy "skills\release-prep\release-notes" "%USERPROFILE%\.claude\skills\release-notes-from-jira" /E /I
xcopy "skills\release-prep\deployment-excel" "%USERPROFILE%\.claude\skills\rtlp-deployment-excel" /E /I
```

### Mac / Linux

```bash
cp -r skills/capacity-planning ~/.claude/skills/
cp -r skills/release-prep/release-notes ~/.claude/skills/release-notes-from-jira
cp -r skills/release-prep/deployment-excel ~/.claude/skills/rtlp-deployment-excel
```

Restart Claude Code after copying — skills are loaded at startup.

## How to use

Once installed, just describe what you want in Claude Code:

| Say this… | Skill triggered |
|-----------|----------------|
| "Set up the board for PI 27.2" | capacity-planning |
| "Generate release notes for version 1.6.7" | release-notes-from-jira |
| "Create the deployment Excel for this release" | rtlp-deployment-excel |

You can also invoke a skill directly with `/skill-name`.

## Requirements

| Skill | Requires |
|-------|----------|
| Capacity Planning | Atlassian MCP plugin (for Jira story import), Microsoft Excel (for deployment sheet), Chrome or Edge (to open the board) |
| Release Notes | Atlassian MCP plugin |
| Deployment Excel | Atlassian MCP plugin, Microsoft Excel |

The Atlassian MCP plugin connects Claude to your Jira and Confluence instance. Install it from the Claude Code MCP marketplace or follow your org's setup guide.

## Contributing

Add a new skill by creating a folder under `skills/` with a `SKILL.md` file, then open a PR. See any existing `SKILL.md` for the format.
