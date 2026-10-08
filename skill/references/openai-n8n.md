# OpenAI in n8n Reference

## Overview
The OpenAI node in n8n (`@n8n/n8n-nodes-langchain.openAi`) provides direct access to OpenAI's chat completion and response APIs. Used for AI classification, data extraction, and analysis within workflows.

## Node Versions & History
- **v1.x**: Original node, "Message a Model" operation
- **v2.0** (n8n 1.117.0): Split into "Generate a Chat Completion" (Chat API) and "Generate a Model Response" (Responses API). Removed Assistants API support.
- **v2.1**: Current latest. Uses "Message a Model" operation name in UI but maps to Chat Completions API.

## Node Configuration

### Basic Setup
- **Node type**: `@n8n/n8n-nodes-langchain.openAi`
- **typeVersion**: `2.1`
- **Credentials**: `openAiApi` — requires an OpenAI API key from platform.openai.com

### Parameters
| Parameter | Value | Notes |
|---|---|---|
| Resource | Text | For chat completions |
| Operation | Message a Model | Chat completions API |
| Model | From dropdown | gpt-4o-mini (cheap), gpt-4o (balanced), gpt-5.2 (best) |
| Simplify Output | ON | Cleaner output structure |

### Messages Configuration
Add messages using the **Messages** section → **"+"** button. Each message has:
- **Type**: Text (default), Image, or File
- **Role**: User, Assistant, or System
- **Prompt**: The message content (supports n8n expressions)

**Message order matters.** Put User message first (Message 1), System message second (Message 2). Both get sent to the API — system sets behavior, user provides the data.

### System Prompt (Role = System)
Sets the AI's behavior, persona, and output format. This is where you define:
- What the AI should do
- How it should respond (JSON schema)
- What signals to look for
- What to avoid

### User Prompt (Role = User)
Contains the actual data to analyze. Use n8n expressions to inject data from the workflow:
```
Company: {{ $('Loop Over Companies').item.json.dba_name }}
Location: {{ $('Loop Over Companies').item.json.city }}, {{ $('Loop Over Companies').item.json.state }}
Website Content: {{ $json.data?.markdown ? $json.data.markdown.substring(0, 3000) : 'No content available' }}
```

**IMPORTANT**: When the Firecrawl node runs before OpenAI in a loop, it overwrites `$json` with scrape results. To access original company data, reference the loop node:
```
{{ $('Loop Over Companies').item.json.fieldName }}
```

## JSON Schema Response Pattern

### System Prompt Template
```
You are a [role] specializing in [domain]. Analyze the provided [data type] and [task].

You MUST respond with ONLY a valid JSON object matching this exact schema, no other text or markdown:

{
  "field_1": boolean,
  "field_2": "enum_val_1" | "enum_val_2" | "enum_val_3",
  "field_3": "string description",
  "field_4": ["string array"],
  "field_5": "category_1" | "category_2" | "unknown"
}

Field definitions:
- field_1: [what this means]
- field_2: [what values mean]
- field_3: [expected content]
- field_4: [what to list]
- field_5: [category descriptions]

Signals FOR [positive classification]: [list indicators]
Signals AGAINST: [list negative indicators]

Do NOT wrap your response in markdown code blocks. Return raw JSON only.
```

### Concrete Example (Tax Firm Classification)
```
You are a business classifier specializing in identifying tax preparation and accounting firms.

You MUST respond with ONLY a valid JSON object matching this exact schema:
{
  "is_tax_firm": boolean,
  "confidence": "high" | "medium" | "low",
  "reasoning": "string",
  "services_detected": ["string"],
  "firm_type": "cpa" | "ea" | "tax_prep" | "bookkeeping" | "financial_services" | "unrelated" | "unknown"
}

Signals FOR tax firm: tax preparation, IRS, returns, CPA, EA, e-file, 1040, W-2, bookkeeping.
Signals AGAINST: directory listing, unrelated business, youth club, attorney-only practice.

Do NOT wrap your response in markdown code blocks. Return raw JSON only.
```

## Parsing OpenAI Response in Code Node

### Response Structure (v2.1 with Simplify Output ON)
The response nests the actual text at `output[0].content[0].text`:
```json
{
  "output": [
    {
      "content": [
        {
          "type": "output_text",
          "text": "{\"is_tax_firm\": true, \"confidence\": \"high\", ...}"
        }
      ],
      "role": "assistant"
    }
  ]
}
```

### Robust Parsing Code
```javascript
const items = $input.all();
const companyData = $('Loop Over Companies').item.json;

let classification = {};
try {
  const aiResponse = items[0].json;
  
  // Try all possible response paths (handles different node versions)
  const content = aiResponse.output?.[0]?.content?.[0]?.text  // v2.1 Message a Model
    || aiResponse.message?.content                              // v1.x
    || aiResponse.content                                       // simplified
    || aiResponse.text                                          // alternative
    || JSON.stringify(aiResponse);                               // fallback
  
  // Clean markdown code fences the model sometimes adds despite instructions
  const cleaned = content
    .replace(/```json\n?/g, '')
    .replace(/```\n?/g, '')
    .trim();
  
  classification = JSON.parse(cleaned);
} catch (e) {
  classification = {
    is_tax_firm: 'parse_error',
    confidence: 'low',
    firm_type: 'unknown',
    services_detected: [],
    reasoning: 'Failed to parse AI response: ' + e.message
  };
}

// Merge company data with AI classification
return [{
  json: {
    ein: companyData.ein,
    legal_name: companyData.legal_name,
    dba_name: companyData.dba_name,
    website_scraped: companyData.company_website,
    is_tax_firm: classification.is_tax_firm ?? 'unknown',
    confidence: classification.confidence || 'low',
    firm_type: classification.firm_type || 'unknown',
    services_detected: Array.isArray(classification.services_detected) 
      ? classification.services_detected.join(', ') 
      : (classification.services_detected || ''),
    reasoning: classification.reasoning || 'No classification available'
  }
}];
```

### Why Multiple Response Paths?
The OpenAI node's output structure varies by:
- Node version (v1.x vs v2.0 vs v2.1)
- Operation type (Chat Completion vs Model Response)
- "Simplify Output" toggle (ON vs OFF)
- Whether tools/functions are connected

Always check multiple paths with `||` chaining to handle any version.

## Model Selection Guide

| Model | Speed | Cost | Best For |
|---|---|---|---|
| gpt-4o-mini | Fast | ~$0.15/1M tokens | Batch classification, simple parsing |
| gpt-4o | Medium | ~$2.50/1M tokens | Complex analysis, nuanced classification |
| gpt-5.2 | Medium | Higher | Demo presentations, best quality |

For batch processing 30+ items in a loop, gpt-4o-mini saves significant cost with minimal quality loss for classification tasks.

## Token Management
- Limit scraped content with `.substring(0, 3000)` (~750 tokens) to control costs
- System prompt typically uses 200-400 tokens
- User prompt with company data adds ~100-200 tokens
- Total per request: ~1,000-1,200 tokens
- For 33 companies with gpt-4o-mini: ~$0.02 total

## Common Issues

### "Parse error" in classification
The model returned non-JSON text (apology, explanation, markdown fences). Fix: add "Return raw JSON only. Do NOT wrap in code blocks." to system prompt.

### Model ignores schema
Temperature too high. Set temperature to 0.2 for classification tasks (lower = more deterministic).

### Empty or truncated response
Max tokens too low. Default is usually fine, but set explicitly to 500+ for classification responses.

### "No website content available" for every company
Firecrawl markdown path is wrong. Check `$json.data?.markdown` vs `$json.markdown` — the correct path depends on whether Firecrawl returns data nested under `data` key.
