# Clay Reference

## Overview
Clay is a data enrichment platform for GTM teams. It uses a spreadsheet-like table interface with enrichment columns that pull data from 75+ providers. NOT an API-first platform — data flows primarily through the table UI.

## Critical Limitation: No Public GET API
Clay does NOT expose API endpoints to fetch/query table rows. Confirmed by Clay support (Jan 2025):
> "Currently, we don't have exposed API endpoints that allow direct querying of data from Clay."

**Do NOT attempt to build HTTP Request nodes that fetch from Clay API endpoints.** URLs like `api.clay.com/v3/sources/{id}/rows` or `api.clay.com/v3/tables/{id}/rows` will return 404.

## Getting Data INTO Clay

### Webhook Source (Programmatic Input)
Every Clay table can have a webhook source. When you send data to the webhook URL, Clay creates new rows.

**Setup**:
1. In Clay table → Sources (left sidebar) → Add Source → Webhooks
2. Copy the webhook URL
3. Optionally create an auth token

**Sending data from n8n**:
- Use HTTP Request node with POST method
- **CRITICAL**: Send ONE record per webhook call with FLAT body parameters
- Do NOT send a `records` array — Clay will dump the entire array into a single cell

**Correct body params:**
```json
{
  "ein": "={{ $json.ein }}",
  "dba_name": "={{ $json.dba_name }}",
  "city": "={{ $json.city }}",
  "state": "={{ $json.state }}",
  "avg_returns_2yr": "={{ $json.avg_returns_2yr }}"
}
```

**Wrong (causes all records in one cell):**
```json
{
  "records": "={{ JSON.stringify($input.all().map(i => i.json)) }}"
}
```

**Rate limiting**: Batch size 1 in n8n, 10-second delay between calls. Clay returns "too many requests" error if you go faster.

### CSV Import
Tables → Import → Upload CSV. Columns auto-map to headers.

## Getting Data OUT of Clay

### Method 1: Google Sheets Action Column (Recommended)
1. In Clay table, add a new column → Action → Google Sheets → Add Row
2. Connect your Google account
3. Select or paste the target spreadsheet URL
4. **Set up headers in the Google Sheet FIRST**, then click "Refresh fields" in Clay
5. Map each Clay column using `/` to insert column references
6. Set "Only run if" condition: `{{Company Domain}}` or chain: `{{Company Domain}} and {{Dba Name}}`
7. Save and run

**Run in batches**: Toggle available for large tables (set rows per batch, e.g., 33)

### Method 2: CSV Export
Table menu → Export → Download as CSV. Manual process, not automatable.

### Method 3: Webhook Callback (Unreliable)
Clay can send enriched data back to an n8n webhook, but this is fire-and-forget. If n8n is slow to respond or misses a request, that row is lost. Not recommended for reliable pipelines.

## Clay Enrichment Columns

### Find Company Domain
- Looks up company domains from company name
- Often returns directory sites (Yelp, MapQuest, Yellow Pages) instead of actual company website
- Not reliable as sole domain source

### Google Search (Perform Search)
- Runs a Google search, returns 4+ results with titles, links, snippets
- More results than Find Company Domain
- Still includes directory noise

### Claygent (AI Agent Column)
Use for intelligent decision-making on enrichment data. Key use case: analyzing search results to pick the actual company website vs directories.

**Setup**:
1. Add column → Use AI → Claygent
2. Write a prompt telling it what to analyze and what to return
3. Set "Only run if" condition using `/` to insert column → `is not empty`
4. Define custom JSON response schema

**Example prompt**:
```
Look at these search results: {Perform Search results}. 
Identify the official company website for this tax/accounting firm. 
Ignore results from: Yelp, MapQuest, Yellow Pages, BBB, Facebook, LinkedIn, Manta.
Return ONLY the domain (e.g. suekatselcpa.com) or "none" if not found.
```

**Custom JSON response schema**:
```json
{
  "type": "object",
  "properties": {
    "domain": {
      "type": "string",
      "description": "Official company website domain or 'none'"
    },
    "link": {
      "type": "string"
    },
    "confidence": {
      "type": "string",
      "enum": ["high", "medium", "low"]
    },
    "reason": {
      "type": "string",
      "description": "Why this domain was selected"
    }
  },
  "required": ["domain", "link", "confidence", "reason"]
}
```

**Credits**: Claygent costs more per row than standard enrichment columns. Budget accordingly.

**"Only run if" syntax**: Use `/` to open column picker (don't type the reference manually — Clay needs the picker to create proper formula references).

## Clay Table ID
Found in the URL: `https://app.clay.com/workspaces/{workspace_id}/workbooks/{workbook_id}/tables/{TABLE_ID}/views/{view_id}`

## Clay Credits & Pricing
- Each enrichment action costs credits
- Standard enrichments: ~1 credit per row per column
- Claygent: ~3-5 credits per row
- Google Search: ~2 credits per row
- Monitor usage in Clay dashboard → Credits
