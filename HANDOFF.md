# IRS Competitor Domain Scrape Pipeline — Handoff Document

## Project Summary

An end-to-end GTM data pipeline that takes raw IRS e-file provider data, filters for ICP, enriches with company domains via Clay, scrapes websites with Firecrawl, and uses OpenAI to classify whether each company is a legitimate tax preparation firm. Results land in a final Google Sheet.

---

## Two Workflows, One Chain

```
Workflow 1                          Clay (external)                    Workflow 2
─────────────────────────          ──────────────────                  ─────────────────────────────────────
Supabase (IRS data)                                                   
  → dedup against Final Sheet                                         
  → send new firms to Clay ──────→ Clay enriches domains              
                                    Claygent picks best domain        
                                    writes to Clay Output Sheet ────→ Read Clay Output Sheet
                                                                        → dedup against Final Sheet
                                                                        → Firecrawl scrape each website
                                                                        → OpenAI classifies (tax firm?)
                                                                        → append results to Final Sheet
```

---

## Google Sheets (Verified)

### IRS Clay Output (bridge between Clay and n8n)
**URL**: `https://docs.google.com/spreadsheets/d/1bHruYu5TiWrYruBZITFwb3Hduz6np1-HIf9eMqZfgOA`
**Spreadsheet ID**: `1bHruYu5TiWrYruBZITFwb3Hduz6np1-HIf9eMqZfgOA`

| Sheet Name | gid | Headers | Written by | Read by |
|---|---|---|---|---|
| Clay Output: Find | 0 | ein, legal_name, dba_name, city, state, avg_returns_2yr, domain, company_website | Clay (Google Sheets action column) | Workflow 2 |

### IRS Project Output (final classified results)
**URL**: `https://docs.google.com/spreadsheets/d/1l8W1SYddrX2J2qQ_rDv1W6HlkVKAlrsb9oNi3ZPdQNI`
**Spreadsheet ID**: `1l8W1SYddrX2J2qQ_rDv1W6HlkVKAlrsb9oNi3ZPdQNI`

| Sheet Name | gid | Headers | Written by | Read by |
|---|---|---|---|---|
| Final Outut: Find | 0 | ein, legal_name, dba_name, website_scraped, is_tax_firm, confidence, firm_type, services_detected, reasoning | Workflow 2 | Workflow 1 (dedup) + Workflow 2 (dedup) |

Both workflows dedup against the same Final sheet using ein.

---

## Workflow 1: ICP_Filter___Clay_Enrichment_Final.json

**Purpose**: Query IRS data from Supabase, skip firms already processed, send new ones to Clay for domain enrichment.

**Flow**:
```
Manual Trigger
  → Query IRS Data (Supabase SQL — avg returns >= 400, last 2 years)
  → Read Results Sheet ("Final Outut: Find", gid=0)
  → Filter New & Format for Clay (dedup by ein, try/catch for empty sheet)
  → Has New Companies? (If: ein is not empty)
      → TRUE: Split In Batches (size 5) → Send to Clay Webhook (1 record per call, flat params) → Wait 0.5s → loop back
      → FALSE: No New Data (stop)
  → Done
```

**Key details**:
- SQL query: filters irs_filings table, groups by firm, averages returns over 2 years
- Threshold of 400 returns ~33 firms from demo dataset (production was 1000-10000)
- Clay webhook receives flat params: ein, organization_name, dba_name, city, state, avg_returns_2yr
- Dedup Code node has alwaysOutputData: true and try/catch for empty results sheet

**After Workflow 1 runs**: Clay table auto-enriches (if table is ON), then Clay's Google Sheets action column writes enriched data to "Clay Output: Find" sheet. Wait for Clay to finish before running Workflow 2.

---

## Workflow 2: Find_Companies___Scrape___AI_Classify___Google_Sheets.json

**Purpose**: Read Clay-enriched companies, scrape their websites, classify with AI, write results.

**Note**: This workflow file contains some extra disconnected nodes from a Find Companies experiment (Pipeline B). Only the chain starting from Manual Trigger is the active flow.

**Flow**:
```
Manual Trigger
  → Read Clay Output ("Clay Output: Find", gid=0)
  → Read Results Sheet ("Final Outut: Find", gid=0)
  → Select Best Domain (dedup by ein, domain selection: company_website first → domain fallback)
  → If (ein is not empty)
      → TRUE: Loop Over Companies (batch size 1)
          → Firecrawl Scrape (markdown format, onlyMainContent)
          → AI Classify Tax Firm (OpenAI gpt-5.2, JSON schema response)
          → Format for Google Sheets (merge company data + AI classification)
          → Append to Results Sheet ("Final Outut: Find", gid=0)
          → loop back to Loop Over Companies
      → FALSE: Send a message (Slack: "No new companies to process")
  → Loop done: Send a message1 (Slack: "Pipeline complete")
```

**Domain selection priority** (Select Best Domain Code node):
1. company_website — AI-validated domain from Claygent (if not empty)
2. domain — default Clay enrichment (even if it's Yelp/MapQuest — let AI classify it)
3. Skip row if neither has a value

**OpenAI classification** returns JSON:
```json
{
  "is_tax_firm": true/false,
  "confidence": "high | medium | low",
  "reasoning": "1-2 sentence explanation",
  "services_detected": ["tax preparation", "bookkeeping"],
  "firm_type": "cpa | ea | tax_prep | bookkeeping | financial_services | unrelated | unknown"
}
```

**Format for Google Sheets** merges company data from $('Loop Over Companies').item.json with parsed AI response, outputs: ein, legal_name, dba_name, website_scraped, is_tax_firm, confidence, firm_type, services_detected, reasoning

---

## Supabase Configuration

| Parameter | Value |
|---|---|
| Host | aws-0-{region}.pooler.supabase.com (pooler, NOT direct) |
| Port | 6543 (NOT 5432) |
| User | postgres.{project-ref} |
| Database | postgres |
| SSL | ON |
| Table | irs_filings |
| Columns | ein, legal_name, dba_name, city, state, zip_code, filing_year, total_returns, accepted_returns, rejected_returns |
| Data | 287 unique firms, 700 rows (3 filing years) |

Common mistakes: using direct host instead of pooler, using postgres instead of postgres.{project-ref}, wrong port.

---

## Clay Configuration

### Table Setup
- **Source**: Webhook (receives data from Workflow 1)
- **Enrichment columns**: Find Company Domain → Google Search (Perform Search) → Claygent AI (picks best domain from search results)
- **Action column**: Google Sheets → Add Row → writes to "Clay Output: Find" sheet
- **Table must be ON** (bottom-right toggle) for auto-enrichment on new webhook rows

### Claygent JSON Response Schema
```json
{
  "type": "object",
  "properties": {
    "domain": { "type": "string", "description": "Official company domain or none" },
    "link": { "type": "string" },
    "confidence": { "type": "string", "enum": ["high", "medium", "low"] },
    "reason": { "type": "string", "description": "Why this domain was selected" }
  },
  "required": ["domain", "link", "confidence", "reason"]
}
```

### Critical Limitation
Clay has NO public GET API to fetch table rows. Data exits Clay only via Google Sheets action columns, CSV export, or webhook push. Do not attempt GET api.clay.com/v3/sources/{id}/rows — it returns 404.

---

## Firecrawl Configuration (CRITICAL)

The Firecrawl n8n community node (@mendable/n8n-nodes-firecrawl.firecrawl) does NOT return page content by default. After importing any workflow JSON, you MUST manually configure:

1. Open the Firecrawl node
2. Expand Scrape Options → Options
3. Under Formats, click "+ Add format" → select Markdown
4. Toggle ON "Only Main Content"

Without this, output has no markdown field and OpenAI gets nothing to analyze.

- Output path: $json.data.markdown (NOT $json.markdown)
- Error handling: onError: continueRegularOutput — failed scrapes don't kill the loop
- Credits: 1 per page. Free plan = 500 one-time. n8n Cloud promo = 100,000+
- Dashboard: firecrawl.dev/app → Usage

---

## OpenAI Node Configuration

- Node type: @n8n/n8n-nodes-langchain.openAi v2.1
- Operation: Message a Model
- Model: gpt-5.2 (or gpt-4o-mini for cheaper runs)
- Messages: User (company data + scraped markdown capped at 3000 chars) + System (classifier + JSON schema)
- Simplify Output: ON

Response parsing (in Format for Google Sheets Code node):
```javascript
const content = aiResponse.output?.[0]?.content?.[0]?.text
  || aiResponse.message?.content
  || aiResponse.content
  || aiResponse.text
  || JSON.stringify(aiResponse);
```
Response lives at output[0].content[0].text in v2.1 — this is non-obvious.

---

## Deduplication Pattern

Both workflows use the same pattern against the Final sheet:
```javascript
let processedEINs = new Set();
try {
  if (processedItems.length > 0 && processedItems[0].json.ein !== undefined) {
    processedEINs = new Set(
      processedItems.map(item => String(item.json.ein || '')).filter(id => id !== '')
    );
  }
} catch (e) {
  processedEINs = new Set();
}
```
- Handles empty sheet on first run (try/catch + undefined check)
- Code nodes have alwaysOutputData: true so the If node downstream still fires

---

## Slack Bot
- Channel ID: C0APH6RJ3UJ
- False branch: "No new companies to process. Clay Output: X rows. Already processed: Y unique companies."
- Done branch: "Pipeline complete. Processed X new companies."

---

## After Import Checklist

1. Re-select Postgres credential on Query IRS Data
2. Re-select Google Sheets OAuth2 on all Google Sheets nodes
3. Re-select Firecrawl API credential
4. Re-select OpenAI API credential
5. Re-select Slack API credential
6. Manually add Markdown format in Firecrawl node (Scrape Options → Options → Formats → + Add format → Markdown)
7. Verify Clay table is ON for auto-enrichment
8. Verify Google Sheet URLs point to correct spreadsheets

---

## Handoff to New Claude Chat

Paste this into a new Claude project chat:

---

I'm continuing work on an IRS Competitor Domain Scrape Pipeline. All context is in the project knowledge — start by reading HANDOFF.md.

Quick summary: two n8n workflows form one chain.
- Workflow 1: Supabase SQL → dedup against final sheet → send new firms to Clay webhook
- Clay enriches domains → Claygent validates → writes to "Clay Output: Find" Google Sheet
- Workflow 2: reads Clay Output sheet → dedup → Firecrawl scrape → OpenAI classify (is this a tax firm?) → append to "Final Outut: Find" sheet

Google Sheets:
- Clay Output: https://docs.google.com/spreadsheets/d/1bHruYu5TiWrYruBZITFwb3Hduz6np1-HIf9eMqZfgOA
- Final Output: https://docs.google.com/spreadsheets/d/1l8W1SYddrX2J2qQ_rDv1W6HlkVKAlrsb9oNi3ZPdQNI

Critical details:
- Clay has NO public GET API
- Firecrawl node needs Markdown format manually added after import
- OpenAI v2.1 response: aiResponse.output?.[0]?.content?.[0]?.text
- Split In Batches: output[0]=done, output[1]=loop
- Both workflows dedup by ein against the same final sheet
- Workflow 2 JSON has disconnected Pipeline B nodes from a Find Companies experiment — ignore them, only the Manual Trigger chain is active

---

## Skills Reference

A comprehensive skill (gtm-pipeline-tools) is installed as a Personal Skill. It has detailed reference docs for Supabase, n8n, Clay, Firecrawl, OpenAI, and Google Sheets in skill/references/. It triggers automatically when working with any of these tools.
