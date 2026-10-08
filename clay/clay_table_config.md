# Clay Table Configuration

The Clay table sits between the two n8n workflows. Workflow 1 sends it one IRS firm at a time. The table finds each firm's own website and writes the row to the Clay Output Google Sheet, where Workflow 2 picks it up.

Clay tables can't be exported as files, so this doc is the written record of the setup. It comes from the project's handoff notes ([`HANDOFF.md`](../HANDOFF.md), [`skill/references/clay.md`](../skill/references/clay.md)), the Workflow 1 export and the live [Clay Output sheet](https://docs.google.com/spreadsheets/d/1bHruYu5TiWrYruBZITFwb3Hduz6np1-HIf9eMqZfgOA/edit?gid=0#gid=0).

## At a glance

| Setting | Value |
|---|---|
| **Job** | Find each firm's own website |
| **Rows come from** | A webhook source. n8n Workflow 1 posts one firm per request |
| **Columns, left to right** | 6 webhook fields → Find Company Domain → Google Search → Claygent → Google Sheets: Add Row |
| **Rows go to** | [IRS Clay Output → Clay Output: Find](https://docs.google.com/spreadsheets/d/1bHruYu5TiWrYruBZITFwb3Hduz6np1-HIf9eMqZfgOA/edit?gid=0#gid=0) |
| **Auto-run** | On, so new rows enrich by themselves |
| **Cost per firm** | About 6–8 Clay credits: Find Company Domain ~1, Google Search ~2, Claygent ~3–5 |

## 1. Webhook source (rows in)

**Setup:** in the table, Sources (left sidebar) → Add Source → Webhooks → copy the URL. Creating an auth token is optional.

**Where the URL goes:** n8n Workflow 1 → Send to Clay Webhook → URL. This repo has `YOUR_CLAY_WEBHOOK_URL` in its place.

**What n8n sends:** one POST per firm, JSON, flat fields. Each field shows up as a column in the table.

| Field | Comes from (IRS query) | Example (made up) |
|---|---|---|
| `ein` | `ein` | `1234567` |
| `organization_name` | `legal_name` | `MAIN STREET TAX LLC` |
| `dba_name` | `dba_name` | `Main Street Tax Service` |
| `city` | `city` | `DAYTON` |
| `state` | `state` | `OH` |
| `avg_returns_2yr` | `avg_returns_2yr` | `912` |

**Rules:**

- **One firm per request, flat fields.** A body like `{"records": [...]}` puts the whole array into one cell.
- **Pace.** Workflow 1 sends batches of 5 with a 0.5-second wait between batches. If Clay answers "too many requests," drop to batches of 1 with a longer wait (the reference notes use 10 seconds).

## 2. Find Company Domain (enrichment)

| Setting | Value |
|---|---|
| Type | Enrichment: Find Company Domain |
| Input | The firm's name |
| Output | A domain or URL |
| Goes to sheet column | `domain` |
| Cost | ~1 credit per row |

**What to expect:** it often returns a listing page instead of the firm's own site. The `domain` column in the Clay Output sheet includes Yelp, MapQuest, LinkedIn, TaxBuzz, Experience.com and D&B pages. Columns 3 and 4 exist to fix that.

## 3. Google Search: Perform Search (enrichment)

| Setting | Value |
|---|---|
| Type | Enrichment: Google Search → Perform Search |
| Input | A search for the firm |
| Output | 4+ results, each with a title, link and snippet |
| Feeds | Claygent's prompt |
| Cost | ~2 credits per row |

The results still include listing sites. Claygent sorts them out.

## 4. Claygent: pick the firm's own website (AI)

| Setting | Value |
|---|---|
| Type | Use AI → Claygent |
| Input | The Google Search results, inserted with `/` |
| Only run if | The search results column is not empty |
| Output | JSON, schema below |
| Goes to sheet column | Its `domain` field → `company_website` |
| Cost | ~3–5 credits per row (more than standard columns) |

**Prompt:**

```
Look at these search results: {Perform Search results}.
Identify the official company website for this tax/accounting firm.
Ignore results from: Yelp, MapQuest, Yellow Pages, BBB, Facebook, LinkedIn, Manta.
Return ONLY the domain (e.g. suekatselcpa.com) or "none" if not found.
```

`{Perform Search results}` is a column reference inserted with the `/` picker, not typed.

**Response schema** (Claygent's custom JSON output):

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

| Field | Meaning |
|---|---|
| `domain` | The firm's official domain, or `none` |
| `link` | A URL returned with the pick (the schema doesn't describe it further) |
| `confidence` | `high`, `medium` or `low` |
| `reason` | Why this domain was chosen |

Only `domain` reaches the sheet, as `company_website`. The other three stay in Clay for spot checks.

## 5. Google Sheets → Add Row (rows out)

| Setting | Value |
|---|---|
| Type | Action → Google Sheets → Add Row |
| Account | The Google account connected in Clay |
| Spreadsheet | [IRS Clay Output](https://docs.google.com/spreadsheets/d/1bHruYu5TiWrYruBZITFwb3Hduz6np1-HIf9eMqZfgOA/edit) |
| Tab | Clay Output: Find (gid 0) |
| Only run if | `{{Company Domain}}`, inserted with `/`, so only rows with a domain are written |
| Run in batches | Optional toggle for big tables (rows per batch, e.g. 33) |

**Setup order matters:**

1. Type the 8 headers into row 1 of the tab first.
2. Add the action column, pick the spreadsheet and tab, then click **Refresh fields**.
3. Map each sheet header to a Clay column with `/`.

**Field mapping:**

| Sheet header | Clay column |
|---|---|
| `ein` | `ein` (webhook) |
| `legal_name` | `organization_name` (webhook) |
| `dba_name` | `dba_name` (webhook) |
| `city` | `city` (webhook) |
| `state` | `state` (webhook) |
| `avg_returns_2yr` | `avg_returns_2yr` (webhook) |
| `domain` | Find Company Domain result |
| `company_website` | Claygent → `domain` |

## Table settings

- **Auto-run:** on (bottom-right toggle). When it's off, webhook rows arrive but no column runs.
- **Table ID:** in the URL, `app.clay.com/workspaces/{workspace_id}/workbooks/{workbook_id}/tables/{TABLE_ID}/views/{view_id}`.
- **Credits:** track usage under Credits in Clay. Sending the same firms twice costs twice.

## Why the output goes through Google Sheets

Clay has no public API for reading table rows (confirmed by Clay support, Jan 2025). `GET api.clay.com/v3/sources/{id}/rows` returns 404.

| Way out of Clay | Used here | Why |
|---|---|---|
| Google Sheets action column | Yes | Runs on its own for each row |
| CSV export | No | Manual |
| Webhook back to n8n | No | Fire-and-forget: if n8n misses a request, that row is lost |

## What the output looks like (demo sheet)

- **66 rows for 33 firms.** Every firm was written twice. Workflow 2 skips repeats by EIN (see "Re-running safely" in the [README](../README.md#re-running-safely)).
- **`domain`:** 13 rows hold a clean domain (like `firmname.com`). The other 53 hold full page URLs, many of them listings (Yelp, MapQuest, LinkedIn, TaxBuzz, Experience.com, D&B).
- **`company_website`:** filled on 30 rows, all among the 53 URL rows. None of the 13 clean-domain rows has one.
- **Workflow 2** scrapes `company_website` when it's there and falls back to `domain`.

## Rebuild from scratch

1. Create a table and add a Webhook source. Copy its URL into Workflow 1's Send to Clay Webhook node.
2. Send one test firm (run Workflow 1, or POST a sample body) so the six input columns appear.
3. Add **Find Company Domain** on the firm's name.
4. Add **Google Search → Perform Search** for the firm.
5. Add **Claygent** with the prompt and JSON schema above. Set Only run if to "search results not empty."
6. In the IRS Clay Output spreadsheet, make the **Clay Output: Find** tab with the 8 headers in row 1.
7. Add **Google Sheets → Add Row**, click Refresh fields, map the columns and set Only run if.
8. Turn **auto-run** on.
9. Send a test firm and check that a row lands in the sheet.

## Not recorded in the handoff

These settings live only in Clay. Check them in the table (column header → Edit column) if you need them exact:

- **Google Search query:** which columns build the search.
- **Find Company Domain input:** `organization_name` or `dba_name`.
- **Where `domain` comes from.** The notes say Find Company Domain, but 53 of 66 values are full page URLs (for example a Yelp listing path), which look more like search-result links.
- **Run conditions.** Rows with a clean `domain` never have a `company_website`. Either Claygent only runs when the first lookup misses, or the Add Row action fires before Claygent finishes. If you want Claygent's pick on every row, set Add Row to wait for the Claygent column.
