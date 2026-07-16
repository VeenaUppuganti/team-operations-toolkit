---
name: rtlp-deployment-excel
description: Use this skill when the user asks to generate an RTLP deployment Excel file from a Jira fix version. Produces a .xlsx file on the Desktop matching the deployment spreadsheet format.
---

# RTLP Deployment Excel Generator

## Purpose
Generate a deployment Excel file (`.xlsx`) for an RTLP release by pulling tickets from a Jira fix version and writing them into the standard RTLP deployment spreadsheet format.

## When to use
- User asks to "generate the deployment Excel for RTLP X.X.X"
- User asks to create a deployment file for an upcoming RTLP release
- User references the deployment spreadsheet format used by the RTLP team

## Workflow

### Step 1 — Identify the fix version
If the user has not specified a version:
1. Query Jira for the first unreleased version in the REAL project:
   ```
   JQL: project = REAL AND fixVersion in unreleasedVersions() ORDER BY fixVersion ASC
   ```
   Take the fix version name from the first result (e.g. `RTLP 1.6.7`).
2. Confirm the version with the user before proceeding.

### Step 2 — Fetch all tickets for the version
Use the Atlassian Jira plugin with this JQL:
```
project = REAL AND fixVersion = "<version name>" ORDER BY issuetype ASC, key ASC
```
Fields to retrieve: `summary`, `status`, `issuetype`, `assignee`, `reporter`, `parent`

Extract for each issue:
- `issueType` — e.g. Feature, Story, Bug
- `key` — e.g. REAL-6022
- `summary`
- `parentKey` — from `fields.parent.key`, or `-` if none
- `parentSummary` — from `fields.parent.fields.summary`, or `-` if none
- `assignee` — displayName, or empty string if null
- `reporter` — displayName, or empty string if null
- `status` — e.g. Done, Completed, Committed Ready to Test

### Step 3 — Determine the output file name
Ask the user: "What would you like to name the file? (Default: `YYYYMMDD - deployment.xlsx`)"
- If the user provides a name, use that (append `.xlsx` if not already present).
- If the user says nothing or accepts the default, use `YYYYMMDD - deployment.xlsx` where YYYYMMDD is today's date (e.g. `20260721 - deployment.xlsx`).

Save to: `C:\Users\VUppuganti\OneDrive - Strata Decision Technology\Desktop\SuperNova\`

### Step 4 — Copy the template and populate the JIRA sheet
The source template to copy from:
```
C:\Users\VUppuganti\OneDrive - Strata Decision Technology\Desktop\SuperNova\
```
Find the most recent `*RTLP deployment.xlsx` file in that folder to use as the template (it carries the Deployment sheet and header styling).

Use a PowerShell script with Excel COM automation:

```powershell
# 1. Copy the most recent existing deployment file as the new file
$folder  = "C:\Users\VUppuganti\OneDrive - Strata Decision Technology\Desktop\SuperNova"
$srcPath = Get-ChildItem $folder -Filter "*RTLP deployment.xlsx" |
               Sort-Object LastWriteTime -Descending |
               Select-Object -First 1 -ExpandProperty FullName
$destName = "<YYYYMMDD> - RTLP deployment.xlsx"   # use actual date
$destPath = Join-Path $folder $destName

# Copy source to avoid overwriting
$localSrc  = "C:\Users\VUppuganti\rtlp_src_tmp.xlsx"
$localDest = "C:\Users\VUppuganti\rtlp_dest_tmp.xlsx"
Copy-Item $srcPath $localSrc -Force

$excel = New-Object -ComObject Excel.Application
$excel.Visible = $false
$excel.DisplayAlerts = $false

$srcWb = $excel.Workbooks.Open($localSrc)
$srcWb.SaveCopyAs($localDest)
$srcWb.Close($false)

$newWb = $excel.Workbooks.Open($localDest)

# Find JIRA sheet
$ws = $null
for ($i = 1; $i -le $newWb.Sheets.Count; $i++) {
    if ($newWb.Sheets.Item($i).Name -match "(?i)jira") { $ws = $newWb.Sheets.Item($i); break }
}

# Clear all content (header + data) — headers will be rewritten
$ws.UsedRange.ClearContents()
$ws.UsedRange.ClearFormats()

# Write header row
$headers = @("Issue Type","Issue Key","Summary","Parent Key","Parent Summary","Assignee","Reporter","Status")
for ($c = 1; $c -le 8; $c++) {
    $cell = $ws.Cells.Item(1, $c)
    $cell.Value2 = $headers[$c - 1]
    $cell.Font.Bold = $true
    $cell.Interior.ColorIndex = -4142   # no fill — Excel default
    $cell.HorizontalAlignment = -4131
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
        $cell.Interior.ColorIndex = -4142   # no fill — Excel default
    }
    $row++
}

$ws.Rows.AutoFit() | Out-Null
$newWb.Save()
$newWb.Close($true)
$excel.Quit()
[System.Runtime.Interopservices.Marshal]::ReleaseComObject($excel) | Out-Null

# Move temp file to final destination
Move-Item $localDest $destPath -Force
Remove-Item $localSrc -Force
```

### Step 5 — Verify
After the script runs, confirm the file exists at `$destPath` and report:
- File name and path
- Number of tickets written
- Any tickets that were NOT in Done/Completed status (flag these to the user)

## Excel format spec
The JIRA sheet must have exactly these 8 columns:

| Col | Header | Notes |
|-----|--------|-------|
| A | Issue Type | e.g. Feature, Story, Bug |
| B | Issue Key | e.g. REAL-6022 |
| C | Summary | Full summary text |
| D | Parent Key | Parent issue key, or `-` if none |
| E | Parent Summary | Parent issue summary, or `-` if none |
| F | Assignee | Display name |
| G | Reporter | Display name |
| H | Status | e.g. Done, Completed |

**Cell formatting:**
- Row 1 (header): bold, no fill (`Interior.ColorIndex = -4142`), left-aligned
- Data rows (row 2+): no fill (`Interior.ColorIndex = -4142`), `WrapText = true`, `HorizontalAlignment = -4131` (left), `Font.Bold = false`
- Do NOT set any `Interior.Color` or border properties — let Excel use its default gridlines

## Notes
- The Deployment sheet is carried over from the template unchanged — do not modify it.
- If the destination file already exists, ask the user before overwriting.
- When writing the header row, use exactly these names (case-sensitive): `Issue Type`, `Issue Key`, `Summary`, `Parent Key`, `Parent Summary`, `Assignee`, `Reporter`, `Status`.
