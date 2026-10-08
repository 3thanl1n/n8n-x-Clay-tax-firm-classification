# Firecrawl Reference

## Overview
Firecrawl is a web scraping API that converts websites into clean, LLM-ready markdown. It handles JavaScript rendering, anti-bot protection, proxy rotation, and PDF parsing automatically. Available as API or n8n community node.

## Plans & Credits

| Plan | Credits | Concurrent Browsers | Cost |
|---|---|---|---|
| Free (direct signup) | 500 one-time | 2 | $0 |
| n8n Cloud promo | 3,000/month + 100,000 bonus | 5 | $0 |
| Hobby | 3,000/month | 5 | $16/month |
| Standard | 100,000/month | 50 | $83/month |
| Growth | 500,000/month | 100 | $333/month |

**Credit costs per operation:**
- Scrape: 1 credit per page
- Crawl: 1 credit per page discovered
- Search: 2 credits per 10 results
- Screenshot: +0.5 credits
- JSON extract: +1 credit
- Enhanced mode: +4 credits

**Check usage**: firecrawl.dev/app → Usage page

**Getting n8n Cloud promo credits**: Install the Firecrawl node on n8n Cloud and click "Connect to Firecrawl". It creates/links your account with 100k bonus credits automatically.

## API Endpoints

### POST /v2/scrape — Single Page
```bash
curl -X POST https://api.firecrawl.dev/v2/scrape \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer fc-YOUR-API-KEY' \
  -d '{
    "url": "https://example.com",
    "formats": ["markdown"],
    "onlyMainContent": true,
    "timeout": 15000
  }'
```

**Response:**
```json
{
  "success": true,
  "data": {
    "markdown": "# Page Title\n\nContent...",
    "metadata": {
      "title": "Page Title",
      "sourceURL": "https://example.com",
      "statusCode": 200,
      "creditsUsed": 1
    }
  }
}
```

### POST /v2/crawl — Multi-Page (Async)
```bash
curl -X POST https://api.firecrawl.dev/v2/crawl \
  -H 'Authorization: Bearer fc-YOUR-API-KEY' \
  -H 'Content-Type: application/json' \
  -d '{
    "url": "https://example.com",
    "limit": 10,
    "scrapeOptions": {
      "formats": ["markdown"],
      "onlyMainContent": true
    }
  }'
```
Returns a job ID. Poll `GET /v2/crawl/{jobId}` for results.

### POST /v2/search — Web Search
```bash
curl -X POST https://api.firecrawl.dev/v2/search \
  -H 'Authorization: Bearer fc-YOUR-API-KEY' \
  -H 'Content-Type: application/json' \
  -d '{
    "query": "tax preparation firm Columbus OH",
    "limit": 5
  }'
```

### POST /v2/map — URL Discovery
Returns all URLs found on a website without scraping content. Useful for planning targeted crawls.

### POST /v2/agent — AI-Driven Research
Accepts a natural language prompt. The agent autonomously navigates, clicks, scrolls, and extracts data. Available in sync (waits for result) and async (returns job ID) modes.

## Scrape Options (Full Parameter Reference)

| Parameter | Type | Default | Description |
|---|---|---|---|
| `url` | string | required | URL to scrape |
| `formats` | array | `["markdown"]` | Output formats: `"markdown"`, `"html"`, `"rawHtml"`, `"links"`, `"screenshot"`, `{"type":"json",...}` |
| `onlyMainContent` | boolean | `true` | Strip nav, footer, sidebar, ads |
| `includeTags` | array | `[]` | CSS selectors to include (e.g., `["article", "main", ".content"]`) |
| `excludeTags` | array | `[]` | CSS selectors to exclude (e.g., `["#ad", "#footer", ".cookie-banner"]`) |
| `waitFor` | integer | `0` | Milliseconds to wait before scraping (for JS-rendered content) |
| `timeout` | integer | `30000` | Request timeout in milliseconds |
| `mobile` | boolean | `false` | Emulate mobile device |
| `actions` | array | `[]` | Browser actions: click, type, scroll, wait, screenshot |
| `location` | object | `null` | Geo-location for region-specific content |
| `parsers` | array | `[]` | `["pdf"]` enables PDF text extraction |

## n8n Community Node Configuration

### Node Type
`@mendable/n8n-nodes-firecrawl.firecrawl` (version 1)

### Credential Setup
1. Sign up at firecrawl.dev
2. Get API key from firecrawl.dev/app → API Keys
3. In n8n: Firecrawl node → Credential → Create New → paste API key

### CRITICAL: Getting Markdown Content
**The node does NOT return page content by default.** Without explicit format configuration, Firecrawl returns only metadata (title, og tags, source URL) with NO `markdown` field.

**To fix:**
1. In the Firecrawl node, expand **Scrape Options → Options**
2. Under **Formats**, click **"+ Add format"**
3. Select **Markdown** from the Type dropdown
4. Toggle ON **"Only Main Content"**

### Node Parameters
- **Resource**: Scraping, Crawling, Searching, Mapping, Extracting, Batch operations, Agent
- **Operation**: `/scrape`, `/crawl`, `/search`, `/map`, `/extract`, etc.
- **URL**: The URL to process. Accepts n8n expressions: `={{ $json.company_website }}`
- **Parsers**: For PDF parsing only (not for output format selection)
- **Scrape Options → Options → Formats**: WHERE YOU SET OUTPUT FORMAT
- **Use Custom Body**: Toggle for raw JSON body control

### Output Data Paths
```javascript
// Markdown content (after enabling Markdown format):
$json.data.markdown

// Page metadata:
$json.data.metadata.title
$json.data.metadata.description
$json.data.metadata.sourceURL
$json.data.metadata.statusCode
$json.data.metadata.creditsUsed

// Success flag:
$json.success  // boolean
```

### Error Handling
Set `onError: "continueRegularOutput"` in the node settings so failed scrapes (timeouts, blocked sites) don't kill the entire loop. The node will output whatever it got (possibly just metadata).

### Sites That Block Content Extraction
These sites return `success: true` but no `markdown` field:
- Yelp (aggressive anti-bot)
- Some MapQuest pages
- LinkedIn (login wall)
- Facebook (login wall)
- Sites with Cloudflare bot protection

Your downstream AI node should handle this with a fallback prompt: "No website content available — classify based on company name and filing data only."

## Scrape vs Crawl Decision Matrix

| Factor | Use Scrape | Use Crawl |
|---|---|---|
| Need homepage only | ✅ | |
| Need subpages (about, services, team) | | ✅ |
| Budget-sensitive (1 credit vs 5-50) | ✅ | |
| Speed matters (3s vs 30-60s) | ✅ | |
| Competitor product scanning | | ✅ |
| Business classification | ✅ | |
| Full-site content audit | | ✅ |
| Running in a loop over 30+ companies | ✅ | |

## HTTP Request Alternative (When Node Isn't Available)
If the Firecrawl community node isn't installed, use an HTTP Request node:
```
Method: POST
URL: https://api.firecrawl.dev/v1/scrape
Headers:
  Authorization: Bearer YOUR_API_KEY
  Content-Type: application/json
Body (JSON):
{
  "url": "{{ $json.company_website }}",
  "formats": ["markdown"],
  "onlyMainContent": true,
  "timeout": 15000
}
```
