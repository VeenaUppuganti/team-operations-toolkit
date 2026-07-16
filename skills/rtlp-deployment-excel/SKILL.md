---
name: rtlp-deployment-excel
description: Use this skill when the user asks to generate a deployment Excel file from a Jira fix version. Produces a .xlsx file with ticket details in the standard deployment spreadsheet format.
---

# Deployment Excel Generator

## Purpose
Generate a deployment `.xlsx` file for a release by pulling tickets from a Jira fix version. Works with any Jira project and any save location.

## When to use
- User asks to "generate the deployment Excel for release X.X.X"
- User asks to create a deployment spreadsheet for an upcoming release
- User wants a Jira ticket list exported to Excel in deployment format

## Workflow

### Step 1 — Gather context
If the user has not already provided them, ask:
1. **Jira project key** — e.g. `REAL`, `JAZZ`, `STAR`
2. **Fix version** — specific version name, or "auto-detect" to find the next unreleased version
3. **Save folder** — where to save the file (e.g. `C:\Users\<name>\Desktop\`)
4. **File name** — default is `YYYYMMDD - deployment.xlsx` using today's date; user can override

If the user says "auto-detect" for the version:
1. Query Jira for the first unreleased version:
   ```
   JQL: project = <PROJECT_KEY> AND fixVersion in unreleasedVersions() ORDER BY fixVersion ASC
   ```
2. Confirm with the user before proceeding: "Found next unreleased version: `X`. Proceed?"

### Step 2 — Fetch all tickets
Use the Atlassian Jira plugin with:
```
JQL: project = <PROJECT_KEY> AND fixVersion = "<version name>" ORDER BY issuetype ASC, key ASC
```
Fields to retrieve: `summary`, `status`, `issuetype`, `assignee`, `reporter`, `parent`

Extract for each issue:
- `issueType` — e.g. Feature, Story, Bug
- `key` — e.g. ABC-123
- `summary`
- `parentKey` — from `fields.parent.key`, or `-` if none
- `parentSummary` — from `fields.parent.fields.summary`, or `-` if none
- `assignee` — displayName, or empty string if null
- `reporter` — displayName, or empty string if null
- `status` — e.g. Done, Completed, In Progress

### Step 3 — Determine file name and save location
- Use the folder and file name provided by the user in Step 1.
- Default file name: `YYYYMMDD - deployment.xlsx` (today's date).
- Append `.xlsx` if the user's name doesn't include it.
- If the file already exists at the destination, ask the user before overwriting.

### Step 4 — Create the Excel file
Use a PowerShell script with Excel COM automation. Copy temp files locally first to avoid issues with paths containing spaces (e.g. OneDrive folders):

```powershell
$saveFolder = "<user-provided folder>"   # e.g. "C:\Users\name\Desktop"
$fileName   = "<user-provided or default name>"  # e.g. "20260721 - deployment.xlsx"
$destPath   = Join-Path $saveFolder $fileName
$localDest  = "$env:USERPROFILE\deployment_tmp.xlsx"

$excel = New-Object -ComObject Excel.Application
$excel.Visible = $false
$excel.DisplayAlerts = $false

# Create a new blank workbook
$wb = $excel.Workbooks.Add()

# Rename the first sheet to JIRA
$ws = $wb.Sheets.Item(1)
$ws.Name = "JIRA"

# Write header row
$headers = @("Issue Type","Issue Key","Summary","Parent Key","Parent Summary","Assignee","Reporter","Status")
for ($c = 1; $c -le 8; $c++) {
    $cell = $ws.Cells.Item(1, $c)
    $cell.Value2 = $headers[$c - 1]
    $cell.Font.Bold = $true
    $cell.HorizontalAlignment = -4131   # xlLeft
}

# Write each ticket — $tickets is the array of hashtables built from Step 2
$row = 2
foreach ($t in $tickets) {
    $cols = @(
        $t.issueType, $t.key, $t.summary,
        $t.parentKey, $t.parentSummary,
        $t.assignee, $t.reporter,
        $t.status
    )
    for ($c = 1; $c -le 8; $c++) {
        $cell = $ws.Cells.Item($row, $c)
        $cell.Value2 = $cols[$c - 1]
        $cell.WrapText = $true
        $cell.HorizontalAlignment = -4131
        $cell.Font.Bold = $false
    }
    $row++
}

$ws.Columns.AutoFit() | Out-Null

# Save locally first, then move to destination (handles OneDrive/space-in-path issues)
$wb.SaveAs($localDest)
$wb.Close($false)
$excel.Quit()
[System.Runtime.Interopservices.Marshal]::ReleaseComObject($excel) | Out-Null

Move-Item $localDest $destPath -Force
Write-Host "Saved to: $destPath"
```

### Step 5 — Verify
Confirm the file was created and report:
- Full file path
- Number of tickets written
- Any tickets NOT in Done/Completed status (flag these to the user)

## Excel format spec
The JIRA sheet has exactly these 8 columns:

| Col | Header | Notes |
|-----|--------|-------|
| A | Issue Type | e.g. Feature, Story, Bug |
| B | Issue Key | e.g. ABC-123 |
| C | Summary | Full summary text |
| D | Parent Key | Parent issue key, or `-` if none |
| E | Parent Summary | Parent issue summary, or `-` if none |
| F | Assignee | Display name |
| G | Reporter | Display name |
| H | Status | e.g. Done, Completed |

**Cell formatting:**
- Row 1 (header): bold, left-aligned, no fill
- Data rows (row 2+): `WrapText = true`, left-aligned (`HorizontalAlignment = -4131`), not bold, no fill
- Do NOT set `Interior.Color` or borders — use Excel default gridlines

## Notes
- No hardcoded paths, project keys, or version names — always ask the user.
- Works on any OS/machine as long as Microsoft Excel is installed (uses COM automation).
- If Excel is not available, inform the user and suggest exporting to CSV instead.
