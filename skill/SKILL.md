---
name: gtm-pipeline-tools
description: "Comprehensive reference for building GTM (Go-To-Market) data pipelines using Supabase, n8n, Clay, Firecrawl, OpenAI, and Google Sheets. Use this skill whenever the user is working with ANY of these tools — even individually — including: Supabase Postgres setup/pooling/queries, n8n workflow building (Split In Batches, HTTP Request, Code nodes, Google Sheets nodes, loop patterns), Clay table enrichment/webhooks/domain finding/Claygent AI agents, Firecrawl web scraping (scrape vs crawl vs agent, markdown extraction, n8n community node config), OpenAI node in n8n (Message a Model, system/user prompts, JSON schema responses), or Google Sheets read/append operations. Also trigger for: webhook architectures, data enrichment pipelines, ICP filtering, competitor signal detection, AI classification workflows, batch processing patterns, or any workflow combining multiple API services. This skill contains hard-won lessons about actual node configurations, API quirks, and failure modes learned through real implementation."
---

# GTM Pipeline Tools — Comprehensive Skill

This skill covers six tools used to build GTM data enrichment and AI classification pipelines. Each tool has a detailed reference file in `references/`. Read the relevant reference before implementing.

## When to Read Which Reference

| If working with... | Read | Key topics |
|---|---|---|
| Supabase / Postgres DB | `references/supabase.md` | Pooler setup, credentials, SQL patterns, free tier limits |
| n8n workflows | `references/n8n.md` | Loop patterns, Code node, node references, batch processing, workflow JSON |
| Clay enrichment | `references/clay.md` | Webhooks, Claygent, domain finding, Google Sheets export, API limitations |
| Firecrawl scraping | `references/firecrawl.md` | Scrape vs crawl vs agent, n8n node config, formats, API endpoints |
| OpenAI in n8n | `references/openai-n8n.md` | Message a Model node, system/user prompts, JSON schemas, response parsing |
| Google Sheets in n8n | `references/google-sheets-n8n.md` | Read/append operations, column mapping, credential setup |

## Quick Troubleshooting

**Supabase connection fails in n8n?**
→ Check: pooler host (not direct), port 6543, user = `postgres.{project-ref}`, SSL on. See `references/supabase.md`.

**Clay webhook sends all records into one cell?**
→ Send one record per call with flat body params, not a `records` array. See `references/clay.md`.

**Firecrawl returns only metadata, no page content?**
→ You MUST add Markdown format in Scrape Options → Options → Formats → + Add format → Markdown. See `references/firecrawl.md`.

**OpenAI node output shows "unknown" / parse error?**
→ Response is at `output[0].content[0].text` in v2.1. Your Code node parser must check this path. See `references/openai-n8n.md`.

**Google Sheets append creates wrong columns?**
→ Headers must exist in Row 1 before first append. Field names must exactly match. See `references/google-sheets-n8n.md`.

**n8n loop processes only first item then stops?**
→ Split In Batches output[0]=done, output[1]=loop. Connect loop processing to output[1]. See `references/n8n.md`.

## End-to-End Architecture Pattern

```
Data Source (Supabase / CSV)
  → n8n: SQL filter → format → send to Clay webhook
  → Clay: enrich domains → Claygent picks best domain → write to Google Sheet
  → n8n: read Google Sheet → select best domain → loop:
      → Firecrawl scrape → OpenAI classify → format → append to results sheet
  → Google Sheets: final output
```

### Key Design Principles
1. **Pull over push**: Read from Google Sheets, don't rely on Clay webhooks (they drop rows)
2. **AI fallback**: When scraping fails, classify on structured data alone (company name + filing volume)
3. **Single page scrape**: One page per company suffices for classification; crawl is for deep analysis
4. **JSON schema in system prompt**: Forces structured AI output for reliable parsing
5. **Domain source tracking**: Tag rows as `ai_validated` vs `default_enrichment` for transparency
6. **Rate limiting**: Batch size 1 with delays between API calls (5-10 seconds per service)
