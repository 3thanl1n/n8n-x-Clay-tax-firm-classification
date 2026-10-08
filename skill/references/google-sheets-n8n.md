# Google Sheets in n8n Reference

## Overview
Google Sheets is used as the data bridge between Clay and n8n (Clay writes enriched data → n8n reads it) and as the final output destination for processed results.

## Node Type
`n8n-nodes-base.googleSheets` (version 4.5)

## Credential Setup
- Type: `googleSheetsOAuth2Api`
- Requires: Google OAuth2 authentication
- Setup: Node → Credential → Create New → Sign in with Google → Grant Sheets permissions
- **Credentials are instance-specific** — must re-select after importing workflow JSON

## Operations

### Read Rows (Get Data from Sheet)
Used to read Clay's enriched output into n8n for processing.

**Configuration:**
- Resource: Sheet Within Document (default)
- Operation: default (Get Rows)
- Document: Select **By URL** → paste full Google Sheets URL
- Sheet: Select from list or by gid

**Important:**
- Returns all rows as an array of JSON objects
- Column headers in Row 1 become the JSON field names
- Empty cells become empty strings, not null
- Numeric values may come as strings — cast in Code node if needed

### Append Row (Add New Data)
Used to write classification results to the output sheet.

**Configuration:**
- Resource: Sheet Within Document
- Operation: **Append Row**
- Document: Select **By URL** → paste full Google Sheets URL
- Sheet: Select from list or by gid
- Column Mapping: **Auto Map Input Data** (matches JSON field names to column headers)

**CRITICAL Requirements:**
1. **Headers MUST exist in Row 1** before the first append operation. If headers are missing, the node errors with "The selected sheet has no header row defined."
2. **Field names must exactly match column headers.** Case-sensitive. If your Code node outputs `dba_name` but your sheet header is `DBA_Name`, it won't map.
3. **Use "Auto Map Input Data"** for mapping mode — it matches by field name automatically.

### Append or Update Row
Same as Append but checks for existing rows first. Useful when re-running workflows to avoid duplicate rows.

- Requires a **matching column** (like `ein`) to identify existing rows
- If match found: updates the row
- If no match: appends new row

## Sheet URL Format
Full URL: `https://docs.google.com/spreadsheets/d/{SPREADSHEET_ID}/edit?gid={SHEET_ID}#gid={SHEET_ID}`

Components:
- **Spreadsheet ID**: The long string after `/d/` (e.g., `1l8W1SYddrX2J2qQ_rDv1W6HlkVKAlrsb9oNi3ZPdQNI`)
- **Sheet ID (gid)**: The number after `gid=` (e.g., `0` for Sheet1)

## Column Header Setup

### For Clay Output Sheet (Input)
```
ein | legal_name | dba_name | city | state | avg_returns_2yr | domain | company_website
```

### For Classification Results Sheet (Output)
```
ein | legal_name | dba_name | website_scraped | is_tax_firm | confidence | firm_type | services_detected | reasoning
```

**Set these up BEFORE running any workflow that reads from or writes to the sheet.**

## Common Issues

### "The selected sheet has no header row defined"
Headers missing in Row 1. Go to the sheet, type headers in Row 1, come back to n8n.

### Data appears in wrong columns
Field names in Code node output don't match header names. Check exact spelling and case.

### Append creates blank rows
The "Use Append" option (under advanced settings) can cause alignment issues if there are gaps in the sheet data. Leave it off for reliability.

### OAuth token expired
Re-authenticate: Node → Credential → click the credential → "Reconnect" or delete and recreate.

### Numbers stored as text
Google Sheets auto-formats. To force number format, use the "Cell Format" option or format the column in Google Sheets directly.

## n8n Expression for Sheet URL
When setting the document in JSON workflow files:
```json
"documentId": {
  "__rl": true,
  "mode": "url",
  "value": "https://docs.google.com/spreadsheets/d/SPREADSHEET_ID/edit?gid=0#gid=0"
}
```

## Performance Notes
- Google Sheets API has rate limits (~100 requests per 100 seconds per user)
- For bulk appends in a loop, add a 1-2 second delay between writes
- For large datasets (500+ rows), consider batch append instead of row-by-row
- Maximum cells per spreadsheet: 10 million (Google limit)

## Two-Sheet Architecture Pattern
```
Sheet 1: "Clay Output" (INPUT)
  ← Clay writes enriched company data here
  → n8n reads from here at workflow start

Sheet 2: "Results" (OUTPUT)  
  ← n8n writes classified results here
  → Human reviews, filters, exports for CRM import
```

This separation keeps raw enrichment data separate from processed results, allowing re-runs without data corruption.
