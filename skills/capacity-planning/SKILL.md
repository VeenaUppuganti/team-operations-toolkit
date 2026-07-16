---
name: capacity-planning
description: Use this skill when the user asks to create, set up, or regenerate a PI Planning Board HTML for a new PI or a new team. Guides through team profiles, sprint dates, PTO, Jira stories, and HTML generation.
---

# PI Planning Board Generator

## Purpose
Generate a self-contained PI Planning Board HTML file for any team and any PI. The board is a single HTML file with capacity tracking, drag-and-drop sprint assignment, PTO, Jira sync, and Confluence collaboration built in.

## When to use
- "Set up the board for PI <Version>"
- "Create a planning board for our team"
- "Generate a new PI planning board"
- "Our new PI starts next month, get the board ready"

## Workflow

---

### Step 1 — Load or create team profile

Team profiles are saved locally so team members don't need to be re-entered each PI.

Profile file path: `C:\Users\<username>\.claude\skills\capacity-planning\teams\<team-name>.json`

**Check for existing profiles:**
- List files matching `C:\Users\*\.claude\skills\capacity-planning\teams\*.json`
- If any exist, show them and ask: "Use an existing team profile, or define a new one?"

**If loading an existing profile:**
- Read the JSON file and display the current team members.
- Ask: "Any changes to the team for this PI?" (e.g., someone joined or left)
- Apply changes and save the updated profile.

**If creating a new profile, collect for each team member:**
| Field | Example |
|-------|---------|
| Name | Lax |
| Role | Dev / QA / PM / Scrum Master |
| Color (hex) | #0d9488 |
| Short ID (no spaces) | lax |

Suggest default colors: `#0d9488`, `#ea580c`, `#059669`, `#7c3aed`, `#2563eb`, `#db2777`, `#d97706`, `#475569`

Save the profile as:
```json
{
  "teamName": "SuperNova",
  "members": [
    {"id": "lax",      "name": "Lax",      "color": "#0d9488", "role": "Dev"},
    {"id": "piyush",   "name": "Piyush",   "color": "#ea580c", "role": "Dev"},
    {"id": "swapna",   "name": "Swapna",   "color": "#059669", "role": "QA"},
    {"id": "tejinder", "name": "Tejinder", "color": "#7c3aed", "role": "Dev"},
    {"id": "jeff",     "name": "Jeff",     "color": "#2563eb", "role": "Dev"}
  ]
}
```

Save with the Write tool to `C:\Users\<username>\.claude\skills\capacity-planning\teams\<team-name>.json`.

---

### Step 2 — PI template configuration

Ask the user for:

| Parameter | Example | Notes |
|-----------|---------|-------|
| PI name | `27.1` | Used in title and storage key |
| PI start date | `2026-07-08` | First day of Sprint 1 |
| Number of sprints | `7` | Default: 7 |
| Sprint length (days) | `14` | Default: 14 |
| Team / product name | `SuperNova` | Used in subtitle |
| Jira project key | `REAL` | e.g. REAL, JAZZ, STAR |
| Jira board ID | `1575` | From the board URL: `/boards/1575` |
| Atlassian cloud ID | `e2cebcc0-...` | From Atlassian admin or ask user |
| Atlassian site domain | `stratadecision` | e.g. `stratadecision.atlassian.net` |
| Confluence page ID | `5744820284` | Optional — for live collaboration |
| Save folder | `C:\Users\name\source\repos\...` | Where to save the HTML |
| File name | `pi-27-1-planning.html` | Default: `pi-<PI-name>-planning.html` |

Also ask for allocation percentages (defaults shown):

| Category | Label | Default % |
|----------|-------|-----------|
| feature | Feature / RTWM Work | 35 |
| support | Production Support | 15 |
| customer | Client Implementation | 20 |
| eng | Engineering Objectives | 20 |
| buffer | Buffer / Overhead | 10 |

---

### Step 3 — Calculate sprint dates automatically

From the PI start date and sprint length, calculate each sprint's date range. Account for any company holidays the user mentions.

**Algorithm:**
```
Sprint 1: start = PI start date,       end = start + (sprintDays - 1) days
Sprint 2: start = Sprint 1 end + 1,    end = start + (sprintDays - 1) days
... (repeat for all sprints)
```

Format dates as: `Jul 8–21`, `Jul 22–Aug 4`, etc. (abbreviated month name, no year).

For each sprint, ask: "Any US/company holidays in Sprint N (dates)?" → collect `holidays` count (integer, default 0).

Build the SPRINTS array:
```javascript
const SPRINTS = [
  {id:1, name:'Sprint 1', dates:'Jul 8–21',    holidays:0},
  {id:2, name:'Sprint 2', dates:'Jul 22–Aug 4', holidays:0},
  // ...
];
```

---

### Step 4 — PTO entry

For each team member, collect PTO days per sprint. Present as a table for clarity:

```
Enter PTO days for each team member (0 if none):

          Sp1  Sp2  Sp3  Sp4  Sp5  Sp6  Sp7
Lax         0    0    0    0    0    7    8
Piyush      0    0    2    2    2    2    2
...
```

Ask: "Enter PTO days as a comma-separated list for each person, one row at a time."

Build the PTO object:
```javascript
let pto = {
  lax:      [0,0,0,0,0,7,8],
  piyush:   [0,0,2,2,2,2,2],
  swapna:   [0,0,1,1,1,1,1],
  tejinder: [0,2,2,2,2,2,0],
  jeff:     [0,0,0,0,0,0,0],
};
```

---

### Step 5 — Pull Jira tickets

Ask: "Pull stories from Jira into the board? (yes/no)"

If yes, ask for:
- Fix version or sprint/label to filter by (or "all open stories in backlog")
- Whether to include Done/Completed tickets

**Fetch stories using the Atlassian Jira plugin:**
```
JQL: project = <PROJECT_KEY> AND fixVersion = "<version>" ORDER BY issuetype ASC, priority ASC, key ASC
```

Or for open backlog:
```
JQL: project = <PROJECT_KEY> AND sprint in openSprints() AND statusCategory != Done ORDER BY priority ASC, key ASC
```

For each issue, extract:
- `key` (e.g. `REAL-123`)
- `summary`
- `status` → map to board display class:
  - `To Do` / `Open` / `Backlog` → `{status:'To Do', cls:'jira-todo', icon:'○'}`
  - `In Progress` / `In Development` → `{status:'In Progress', cls:'jira-inprogress', icon:'◎'}`
  - `In Review` / `Code Review` → `{status:'In Review', cls:'jira-review', icon:'◑'}`
  - `Done` / `Completed` / `Closed` → `{status:'Done', cls:'jira-done', icon:'●'}`
- `issuetype.name` → story type
- `story_points` (if available) → default 0 if missing

Build two JavaScript objects:

**JIRA_META** (current status from Jira):
```javascript
const JIRA_META = {
  'REAL-123': {status:'In Progress', cls:'jira-inprogress', icon:'◎'},
  'REAL-456': {status:'Done',        cls:'jira-done',       icon:'●'},
};
```

**stories array** (all tickets start in backlog):
```javascript
let stories = [
  {id:1, sprint:'backlog', title:'REAL-123 Short summary', pts:0, owner:'', priority:'M', jira:'REAL-123', tags:[]},
  {id:2, sprint:'backlog', title:'REAL-456 Short summary', pts:0, owner:'', priority:'M', jira:'REAL-456', tags:[]},
];
let nextId = <count + 1>;
```

Priority defaults to `'M'` (Medium). Map `Highest`/`High` → `'H'`, `Low`/`Lowest` → `'L'`.

If no Jira pull, use:
```javascript
const JIRA_META = {};
let stories = [];
let nextId = 1;
```

---

### Step 6 — Generate the board HTML

Read the base template:
```
C:\Users\VUppuganti\source\repos\Veena-Requirements\pi-27-1-planning.html
```

Use a PowerShell script to perform targeted string replacements on the key configuration sections. The replacements target exact marker strings in the template.

**PowerShell generation script:**

```powershell
$templatePath = "C:\Users\VUppuganti\source\repos\Veena-Requirements\pi-27-1-planning.html"
$outputPath   = "<user-provided save folder>\<file name>"
$localOut     = "$env:USERPROFILE\pi_board_tmp.html"

$html = Get-Content $templatePath -Raw -Encoding UTF8

# 1. Storage key — unique per PI to avoid cross-PI localStorage collisions
$html = $html -replace "const STORAGE_KEY\s*=\s*'[^']*'", "const STORAGE_KEY = 'pi-<PI_NAME>-board-v2'"

# 2. Jsync storage key
$html = $html -replace "const JSYNC_KEY\s*=\s*'[^']*'", "const JSYNC_KEY = 'pi-<PI_NAME>-jsync-v2'"

# 3. Collab storage key
$html = $html -replace "const COLLAB_KEY\s*=\s*'[^']*'", "const COLLAB_KEY = 'pi-<PI_NAME>-collab-v1'"

# 4. SPRINTS array — replace entire array block
$sprintsJs = @"
const SPRINTS = [
<SPRINTS_LINES>
];
"@
$html = $html -replace "(?s)const SPRINTS\s*=\s*\[.*?\];", $sprintsJs.Trim()

# 5. TEAM array
$teamJs = @"
const TEAM = [
<TEAM_LINES>
];
"@
$html = $html -replace "(?s)const TEAM\s*=\s*\[.*?\];", $teamJs.Trim()

# 6. allocations
$allocJs = @"
let allocations = [
<ALLOC_LINES>
];
"@
$html = $html -replace "(?s)let allocations\s*=\s*\[.*?\];", $allocJs.Trim()

# 7. PTO object
$ptoJs = @"
let pto = {
<PTO_LINES>
};
"@
$html = $html -replace "(?s)let pto\s*=\s*\{.*?\};", $ptoJs.Trim()

# 8. JIRA constants
$html = $html -replace "const JIRA_BASE\s*=\s*'[^']*'",     "const JIRA_BASE  = 'https://<DOMAIN>.atlassian.net'"
$html = $html -replace "const JIRA_BOARD\s*=\s*JIRA_BASE\s*\+\s*'[^']*'", "const JIRA_BOARD = JIRA_BASE + '/jira/software/c/projects/<PROJECT_KEY>/boards/<BOARD_ID>'"
$html = $html -replace "const JIRA_BOARD_ID\s*=\s*\d+",     "const JIRA_BOARD_ID = <BOARD_ID>"
$html = $html -replace "const JIRA_CLOUD\s*=\s*'[^']*'",    "const JIRA_CLOUD = '<CLOUD_ID>'"
$html = $html -replace "const JSYNC_KEY\s*=\s*'[^']*'",     "const JSYNC_KEY  = 'pi-<PI_NAME>-jsync-v2'"

# 9. JIRA_META
$html = $html -replace "(?s)const JIRA_META\s*=\s*\{.*?\};", "const JIRA_META = <JIRA_META_JS>;"

# 10. stories + nextId
$html = $html -replace "(?s)let stories\s*=\s*\[.*?\];",     "let stories = <STORIES_JS>;"
$html = $html -replace "let nextId\s*=\s*\d+",               "let nextId = <NEXT_ID>"

# 11. Confluence page defaults
$html = $html -replace "let collabPageId\s*=\s*'[^']*'",     "let collabPageId = '<CONF_PAGE_ID>'"
$html = $html -replace "const CONF_CLOUD\s*=\s*'[^']*'",     "const CONF_CLOUD   = '<CLOUD_ID>'"

# 12. Page title / subtitle in HTML header
$html = $html -replace "Program Increment \S+ — Planning Board", "Program Increment <PI_NAME> — Planning Board"
$html = $html -replace "SuperNova · [^<\"]+", "<TEAM_NAME> · <PI_DATE_RANGE>"

# 13. Export filename
$html = $html -replace "PI-26\.3-Planning-Board-", "PI-<PI_NAME>-Planning-Board-"

# 14. Clear previously baked export data (clean template)
$html = $html -replace "(?s)/\*__EXPORT_HERE__\*/.*?/\*__EXPORT_END__\*/", "/*__EXPORT_HERE__*/"

$html | Set-Content $localOut -Encoding UTF8
Move-Item $localOut $outputPath -Force
Write-Host "Board saved to: $outputPath"
```

**Before running:** substitute all `<PLACEHOLDER>` values using the data collected in Steps 1–5.

**Generate the substitution strings in Claude before inserting into the PowerShell script:**

SPRINTS lines example:
```
  {id:1,name:'Sprint 1',dates:'Jul 8–21',   holidays:0},
  {id:2,name:'Sprint 2',dates:'Jul 22–Aug 4',holidays:0},
```

TEAM lines example:
```
  {id:'lax',    name:'Lax',    color:'#0d9488',role:'Dev'},
  {id:'piyush', name:'Piyush', color:'#ea580c',role:'Dev'},
```

Allocations lines example:
```
  {key:'feature', label:'RTWM / Feature Work',   pct:35, color:'#3b82f6',bg:'#eff6ff'},
  {key:'support', label:'Production Support',      pct:15, color:'#f59e0b',bg:'#fffbeb'},
  {key:'customer',label:'Client Implementation',   pct:20, color:'#8b5cf6',bg:'#f5f3ff'},
  {key:'eng',     label:'Engineering Objectives',  pct:20, color:'#10b981',bg:'#ecfdf5'},
  {key:'buffer',  label:'Buffer / Overhead',        pct:10, color:'#94a3b8',bg:'#f8fafc'},
```

PTO lines (one per member):
```
  lax:      [0,0,0,0,0,7,8],
  piyush:   [0,0,2,2,2,2,2],
```

---

### Step 7 — Validate and open

After generating the file:
1. Confirm the file exists at the output path.
2. Tell the user:
   - File path
   - PI name and date range
   - Team size
   - Number of sprints and sprint dates
   - Total stories loaded (or 0 if skipped)
3. Remind the user:
   - "Open the HTML file in Chrome or Edge — it works fully offline."
   - "Use the **Export Shareable** button in the toolbar to snapshot and share the board."
   - "Use the **Jira Sync** panel in the board to map sprint IDs and push story assignments back to Jira."
   - "If Confluence collaboration is set up, enter your Atlassian email and API token in the Collaboration panel."

---

### Step 8 — Snapshot for future PIs (optional)

After the board is in use and the PI is complete, the user can run this skill again to:
- Archive the current team profile (already saved in Step 1).
- Carry forward any stories not completed into the next PI's backlog — ask: "Carry over incomplete stories to PI X.Y?"
- Update PTO for the new PI.

The team profile JSON is the persistent artifact — it survives across PIs and only needs updating when the team changes.

---

## File locations

| Artifact | Path |
|----------|------|
| Base template | `C:\Users\VUppuganti\source\repos\Veena-Requirements\pi-27-1-planning.html` |
| Team profiles | `C:\Users\<username>\.claude\skills\capacity-planning\teams\<team-name>.json` |
| Generated board | User-specified save folder |

## Notes
- The board uses `localStorage` keyed by `STORAGE_KEY`. Using a unique key per PI (e.g. `pi-27-1-board-v2`) prevents data from bleeding between PIs.
- PTO and allocations are baked directly into the HTML so they survive browser/profile changes — do NOT restore them from localStorage.
- The `/*__EXPORT_HERE__*/` marker must remain in the HTML for the board's own Export Shareable feature to work.
- If the Atlassian cloud ID is unknown, find it at: `https://<domain>.atlassian.net/_edge/tenant_info` or ask the user to check their admin settings.
