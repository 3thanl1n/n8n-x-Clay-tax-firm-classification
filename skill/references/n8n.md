# n8n Workflow Reference

## Overview
n8n is an open-source workflow automation platform. Used as the orchestration layer connecting all tools in the GTM pipeline. Available as cloud (n8n.io) or self-hosted (Docker).

## Split In Batches (Loop Over Items) — CRITICAL NODE

### Output Ports
The Split In Batches node has TWO outputs. Getting these wrong breaks everything:

- **output[0] = DONE** — fires ONCE after ALL batches have been processed. Connect to your final/summary node.
- **output[1] = LOOP** — fires for EACH batch. Connect to the first node in your processing chain.

In the n8n UI, the top connector is "done" and the bottom connector is "loop".

In workflow JSON connections:
```json
"Loop Over Companies": {
  "main": [
    [ { "node": "All Done" } ],          // output[0] = done
    [ { "node": "Firecrawl Scrape" } ]    // output[1] = loop
  ]
}
```

### How the Loop Works
1. Items enter the Split In Batches node
2. Node sends `batchSize` items through output[1] (loop)
3. Processing chain runs (scrape → classify → save)
4. Last node in chain connects BACK to Split In Batches input
5. Node sends next batch through output[1]
6. Repeat until all items processed
7. Node fires output[0] (done) with ALL combined results

### Batch Size Guidelines
| Scenario | Batch Size | Why |
|---|---|---|
| API calls with rate limits | 1 | One request at a time prevents throttling |
| Bulk data transforms | 10-50 | Processing is local, no external limits |
| File operations | 5-10 | Balance throughput with memory |

### Checking Loop Status in Expressions
```
// Check if items remain:
{{ $("Loop Over Companies").context["noItemsLeft"] }}  // returns boolean

// Get current iteration index:
{{ $("Loop Over Companies").context["currentRunIndex"] }}
```

## Code Node

### Execution Modes
- **Run Once for All Items**: Receives all items as array. Use `$input.all()`.
- **Run Once for Each Item**: Receives one item at a time. Use `$input.item`.

For formatting/parsing in loops, use "Run Once for All Items" — it receives the single item from the current batch.

### Referencing Other Nodes
When nodes in a loop overwrite `$json` (like Firecrawl replacing company data with scrape results), reference earlier nodes directly:

```javascript
// Get data from the loop node (preserved original data)
const companyData = $('Loop Over Companies').item.json;

// Get data from current node's input
const items = $input.all();

// Combine them
return [{
  json: {
    ein: companyData.ein,
    dba_name: companyData.dba_name,
    scraped_content: items[0].json.data?.markdown
  }
}];
```

### Available Node References
```javascript
$input.all()              // All items from previous node
$input.item               // Single item (in "each item" mode)
$('NodeName').item.json   // Specific node's output in current loop iteration
$('NodeName').all()       // All items from a specific node
$json                     // Shorthand for current item's json
$execution.id             // Current execution ID
$now                      // Current timestamp
```

### Filtering Pattern (Return Null + Filter)
```javascript
const results = items.map(item => {
  if (!isValid(item)) return null;  // skip this item
  return { json: { /* transformed data */ } };
}).filter(item => item !== null);
return results;
```

### Error Handling in Code Nodes
```javascript
let classification = {};
try {
  const content = items[0].json.output?.[0]?.content?.[0]?.text || '{}';
  const cleaned = content.replace(/```json\n?/g, '').replace(/```\n?/g, '').trim();
  classification = JSON.parse(cleaned);
} catch (e) {
  classification = {
    status: 'parse_error',
    error: e.message
  };
}
```

## HTTP Request Node

### Configuration for API Calls
- **Method**: GET, POST, PUT, DELETE
- **URL**: Can use expressions: `={{ $json.company_website }}`
- **Headers**: Add via "Send Headers" → parameter pairs
- **Body**: "Send Body" → JSON format → parameter pairs or raw JSON
- **Error handling**: Set `onError: "continueRegularOutput"` to prevent loop breaks on 404s/500s

### Rate Limiting Pattern
Add a Wait node after each HTTP request in a loop:
```
HTTP Request → Wait (5 seconds) → back to Split In Batches
```

## Wait Node
- **Amount**: Number of seconds/minutes to pause
- **Unit**: seconds, minutes, hours
- Place AFTER the last processing node, BEFORE looping back to Split In Batches
- 5 seconds is safe for most APIs; increase to 10 for aggressive rate limiters (Clay)

## Workflow JSON Import/Export

### What Transfers
- Node configurations and parameters
- Connection map between nodes
- Node positions on canvas
- Tags and workflow metadata

### What Does NOT Transfer
- **Credentials**: All credential IDs are instance-specific. After import, click each node with a warning icon and re-select your saved credential.
- **Community node availability**: If the target instance doesn't have the Firecrawl community node installed, those nodes show as placeholders.

### Node Type Strings (for JSON authoring)
| Node | Type String | Version |
|---|---|---|
| Manual Trigger | `n8n-nodes-base.manualTrigger` | 1 |
| Split In Batches | `n8n-nodes-base.splitInBatches` | 3 |
| Code | `n8n-nodes-base.code` | 2 |
| HTTP Request | `n8n-nodes-base.httpRequest` | 4.2 |
| Google Sheets | `n8n-nodes-base.googleSheets` | 4.5 |
| Wait | `n8n-nodes-base.wait` | 1.1 |
| Set/Edit Fields | `n8n-nodes-base.set` | 3.4 |
| If | `n8n-nodes-base.if` | 2 |
| No Operation | `n8n-nodes-base.noOp` | 1 |
| Postgres | `n8n-nodes-base.postgres` | 2.5 |
| OpenAI | `@n8n/n8n-nodes-langchain.openAi` | 2.1 |
| Firecrawl | `@mendable/n8n-nodes-firecrawl.firecrawl` | 1 |
| Aggregate | `n8n-nodes-base.aggregate` | 1 |

## Expression Syntax
n8n uses `{{ }}` for expressions in node parameters:
```
{{ $json.fieldName }}                           // Current item field
{{ $('NodeName').item.json.fieldName }}          // Specific node field
{{ $json.value || 'default' }}                  // Default value
{{ $json.arr ? $json.arr.join(', ') : '' }}     // Conditional
{{ new Date().toISOString() }}                  // JavaScript expressions
```
