# AI Website Copy Auditor & Conversion Advisor — Workflow Documentation

| Field | Value |
|---|---|
| Workflow Name | AI Website Copy Auditor & Conversion Advisor |
| Workflow ID | GOoQHLsn1HGaI9df |
| Version | Not tracked in export. Internal revision hash (`versionId`): `d4931afe-ba8e-4001-9d04-78afc9980415`. Recommend the owner assign a semantic version, e.g. **v1.0**. |
| Owner | _[To be provided by workflow owner]_ |
| Last Updated | `updatedAt` not present in this export (workflow save timestamp unknown). Document generated: 2026-07-10 |
| Active Status | `false` (currently deactivated in n8n) |

---

## Purpose

This workflow exposes an HTTP endpoint that accepts a website URL, scrapes the page's visible text, and uses an AI agent to run an automated conversion-rate-optimization (CRO) copy audit. The audit result (scores + issues + recommendations) is logged to Google Sheets and echoed back to the original caller as the webhook response.

## Business Use Case

_Inferred (not stated in the export — no Sticky Note nodes are present to confirm intent)_: Automates first-pass website copy audits that would otherwise require a manual CRO review, giving an instant scored report (headline, value proposition, CTA, trust signals, clarity) plus a persistent audit log in Google Sheets. Please confirm or replace this with the actual business justification.

## Workflow Summary

A client sends a `POST` request to the `/website-audit` webhook with a `website` URL and `email` in the JSON body. The **Validate URL** node extracts and trims the URL, captures the requester's email, and stamps an audit timestamp. **Fetch Website HTML** downloads the raw page HTML. **Extract & Clean Text Content** strips scripts, styles, and tags down to plain text (capped at 4,000 characters). This text is handed to the **AI Agent - Copy Audit Analysis** node, which uses an OpenRouter-hosted free-tier model to score the copy across five CRO dimensions and return structured JSON. **Format Audit Results** parses that JSON, **Save to Google Sheets** appends it as a row, and **Respond to Webhook** returns a formatted summary to the original caller.

## Workflow Diagram

See the attached canvas screenshot (`AI.png`) — it confirms a purely **linear, left-to-right** chain of 8 functional nodes, with the **OpenRouter Chat Model** node feeding into the **AI Agent - Copy Audit Analysis** node from below via the `ai_languageModel` connector. The Agent node's optional **Memory** and **Tool** inputs are visible on the canvas but not connected (shown as empty `+` sockets) — the JSON confirms no memory or tool sub-nodes exist. There is no branching, no loop, and a single entry point.

```
Webhook
  → Validate URL
    → Fetch Website HTML
      → Extract & Clean Text Content
        → AI Agent - Copy Audit Analysis  ← (ai_languageModel) OpenRouter Chat Model
          → Format Audit Results
            → Save to Google Sheets
              → Respond to Webhook
```

## Trigger Details

| Property | Value |
|---|---|
| Trigger Node | `Webhook` |
| Node Type | `n8n-nodes-base.webhook` (v2.1) |
| HTTP Method | `POST` |
| Path | `/website-audit` |
| Response Mode | `responseNode` (response is sent later, explicitly, by the **Respond to Webhook** node) |
| Authentication | None configured on the Webhook node |
| Webhook ID | `a6b6b1ff-493e-4a54-94dc-27f36ba95674` |

This is the single entry point into the workflow — there are no other trigger nodes.

## Execution Flow

1. **Webhook** receives a `POST /website-audit` request with a JSON body containing `website` (URL) and `email`.
2. **Validate URL** (Set node) extracts `url`, `user_email`, generates `audit_timestamp` (current time), and `clean_url` (trimmed URL) as new fields.
3. **Fetch Website HTML** (HTTP Request) performs a `GET` on `clean_url`, returning the raw HTML as plain text (10s timeout).
4. **Extract & Clean Text Content** (Code node) strips `<script>`/`<style>` blocks and all remaining HTML tags, decodes a small set of HTML entities, collapses whitespace, and truncates to 4,000 characters. Outputs `url`, `text_content`, `word_count`.
5. **AI Agent - Copy Audit Analysis** (LangChain Agent) sends the cleaned copy to the OpenRouter-hosted LLM (via the connected **OpenRouter Chat Model** node) with a CRO-audit system prompt, requesting a strict JSON response.
6. **Format Audit Results** (Code node) parses the agent's raw string output (`items[0].json.output`) into a structured object with score fields, `key_issues`, `conversion_risks`, `recommendations`, and `summary`.
7. **Save to Google Sheets** appends a new row to the connected spreadsheet with all score/issue/recommendation fields, plus `URL`, `Audit Date`, and `User Email` pulled back from the **Validate URL** node's output.
8. **Respond to Webhook** returns a plain-text formatted summary (score breakdown, summary, top issues, top recommendations) to the original HTTP caller, using the field values echoed back by the Google Sheets append operation.

There is no branching or looping — every request follows this exact eight-step path.

## Node List

| # | Node Name | Type | Purpose | Disabled |
|---|---|---|---|---|
| 1 | Webhook | Webhook (`n8n-nodes-base.webhook`) | Entry point; receives POST with URL + email | No |
| 2 | Validate URL | Set (`n8n-nodes-base.set`) | Extracts/cleans input fields, stamps timestamp | No |
| 3 | Fetch Website HTML | HTTP Request (`n8n-nodes-base.httpRequest`) | Downloads target page's raw HTML | No |
| 4 | Extract & Clean Text Content | Code (`n8n-nodes-base.code`) | Strips HTML down to plain visible text | No |
| 5 | AI Agent - Copy Audit Analysis | LangChain Agent (`@n8n/n8n-nodes-langchain.agent`) | Runs the CRO copy audit via LLM | No |
| 6 | OpenRouter Chat Model | LangChain OpenRouter Chat Model (`@n8n/n8n-nodes-langchain.lmChatOpenRouter`) | LLM provider for the Agent node | No |
| 7 | Format Audit Results | Code (`n8n-nodes-base.code`) | Parses AI's JSON string into structured fields | No |
| 8 | Save to Google Sheets | Google Sheets (`n8n-nodes-base.googleSheets`) | Appends audit row to spreadsheet | No |
| 9 | Respond to Webhook | Respond to Webhook (`n8n-nodes-base.respondToWebhook`) | Returns formatted summary to caller | No |

## Node-by-Node Description

### Webhook
Entry point. Listens for `POST /website-audit`. Response is deferred (`responseMode: responseNode`), meaning nothing is returned to the caller until the **Respond to Webhook** node executes at the end of the chain. No authentication is configured. Passes the full request (`body.website`, `body.email`) downstream.

### Validate URL
A Set node ("Edit Fields") that creates four new string fields from the incoming request body:
- `url` = `{{ $json.body.website }}`
- `user_email` = `{{ $json.body.email }}`
- `audit_timestamp` = `{{ $now.toISO() }}`
- `clean_url` = `{{ $json.body.website.trim() }}`

Despite the name, this node does not perform any validation (no format/URL-shape check, no empty-value guard) — it only extracts and trims. `url` and `clean_url` are set from the same source and will always be identical (the only difference is `.trim()`), which is a minor redundancy.

### Fetch Website HTML
An HTTP Request node performing a `GET` request to `{{ $json.clean_url }}`, with response format forced to `text` and a 10-second timeout. No authentication, no custom headers, no retry configuration. Output is a single `data` field containing the raw HTML string.

### Extract & Clean Text Content
A Code node (JavaScript) that:
1. Removes `<script>...</script>` and `<style>...</style>` blocks via regex.
2. Strips all remaining HTML tags.
3. Decodes five common HTML entities (`&nbsp;`, `&amp;`, `&lt;`, `&gt;`, `&quot;`).
4. Collapses all whitespace runs to single spaces and trims.
5. Truncates the result to the first 4,000 characters (token-safety measure for the LLM call).
6. Returns `{ url, text_content, word_count }`.

Note: `url` here is read from `$input.item.json.clean_url`, but the **preceding HTTP Request node's output does not carry forward the `clean_url` field** (see Known Limitations) — in practice this expression is likely to resolve to `undefined`/empty and fall back to `""`.

### AI Agent - Copy Audit Analysis
A LangChain Agent node (`promptType: define`) that sends a user prompt containing the URL, word count, and cleaned website copy, guided by a detailed system prompt instructing the model to act as a CRO consultant. See **AI/LLM Integration** below for the full prompt and expected schema. The agent's chat model is supplied by the connected **OpenRouter Chat Model** node (`ai_languageModel` connection); no memory or tool sub-nodes are attached, so this is a single-turn, tool-less completion. Output is placed in `output` as a raw string (expected to be a JSON-formatted string per the system prompt).

### OpenRouter Chat Model
Supplies the LLM used by the Agent node above. Configured with:
- Model: `nvidia/nemotron-nano-9b-v2:free` (OpenRouter free-tier model)
- Credential: `OpenRouter account for AI Journal` (OpenRouter API credential)

No temperature, max-token, or other generation options are explicitly overridden — defaults apply.

### Format Audit Results
A Code node that takes `items[0].json.output` (the AI Agent's raw string output) and runs `JSON.parse()` on it, then re-emits a flat object with: `overall_score`, `headline_score`, `value_prop_score`, `cta_score`, `trust_score`, `clarity_score`, `key_issues`, `conversion_risks`, `recommendations`, `summary`. There is no `try/catch` around the `JSON.parse()` call — if the model returns text that isn't valid JSON (e.g. wrapped in markdown code fences, or with commentary before/after), this node will throw and the execution will fail.

### Save to Google Sheets
Appends one row to a Google Sheet using `operation: append`. Column mapping (`defineBelow` mode):

| Sheet Column | Source Expression |
|---|---|
| URL | `{{ $('Validate URL').item.json.url }}` |
| Audit Date | `{{ $('Validate URL').item.json.audit_timestamp }}` |
| Overall Score | `{{ $json.overall_score }}` |
| Headline Score | `{{ $json.headline_score }}` |
| Value Prop Score | `{{ $json.value_prop_score }}` |
| CTA Score | `{{ $json.cta_score }}` |
| Trust Score | `{{ $json.trust_score }}` |
| Clarity Score | `{{ $json.clarity_score }}` |
| Key Issues | `{{ $json.key_issues[0] }}{{ $json.key_issues[1] }}{{ $json.key_issues[2] }}` |
| Conversion Risks | `{{ $json.conversion_risks[0] }}{{ $json.conversion_risks[1] }}` |
| Recommendations | `{{ $json.recommendations[0] }}{{ $json.recommendations[1] }}{{ $json.recommendations[2] }}` |
| Summary | `{{ $json.summary }}` |
| User Email | `{{ $('Validate URL').item.json.user_email }}` |

Target spreadsheet: **AI Website Copy Auditor & Conversion Advisor** (`1PFLrHcCY5yD8Ur7OInbBZGW5n5oxBGQw_ZHIpAj-N1M`), sheet **Sheet1** (`gid=0`). Note: the array-field expressions concatenate list items with no separator between them (see Known Limitations).

### Respond to Webhook
Returns a plain-text (`respondWith: text`) response built from field names matching the **Google Sheets column headers** (`$json['Overall Score']`, `$json.URL`, `$json['Key Issues']`, etc.) — this relies on the Google Sheets append operation echoing back the inserted row using those exact header names, which is standard n8n Google Sheets node behavior. Template:

```
📊 Overall Score: {{ $json['Overall Score'] }}/100

🌐 URL: {{ $json.URL }}

📈 BREAKDOWN:
- Headline: {{ $json['Headline Score'] }}/100
- Value Prop: {{ $json['Value Prop Score'] }}/100
- CTA: {{ $json['CTA Score'] }}/100
- Trust: {{ $json['Trust Score'] }}/100
- Clarity: {{ $json['Clarity Score'] }}/100

💡 SUMMARY:{{ $json.Summary }}

🚨 TOP ISSUES:{{ $json['Key Issues'] }}

✅ TOP RECOMMENDATIONS:{{ $json.Recommendations }}

📋 Full report saved to Google Sheets
```

## Node Configuration

Concrete configuration for nodes where exact settings matter operationally:

- **Fetch Website HTML**: `GET {{ $json.clean_url }}`, `responseFormat: text`, `timeout: 10000` ms. No auth, no headers, no query params, no retry.
- **AI Agent - Copy Audit Analysis** — full prompt/schema documented under **AI/LLM Integration**.
- **Save to Google Sheets** — full column map documented under **Node-by-Node Description** above.
- **Extract & Clean Text Content** and **Format Audit Results** — full Code node logic summarized above; see source JSON for exact code if needed for maintenance.

## Variables & Expressions

Notable n8n expressions used across the workflow:

- `{{ $json.body.website }}` / `{{ $json.body.email }}` — pulls fields out of the webhook's parsed JSON body (Validate URL node).
- `{{ $now.toISO() }}` — current execution timestamp in ISO 8601 (Validate URL node).
- `{{ $json.body.website.trim() }}` — same URL, whitespace-trimmed (Validate URL node).
- `{{ $('Validate URL').item.json.url }}`, `{{ $('Validate URL').item.json.user_email }}`, `{{ $('Validate URL').item.json.audit_timestamp }}` — cross-node references pulling the original request fields back in at the Google Sheets step, several nodes downstream of where they were set (Save to Google Sheets node).
- `{{ $json.key_issues[0] }}{{ $json.key_issues[1] }}{{ $json.key_issues[2] }}` (and equivalents for `conversion_risks`, `recommendations`) — indexes into the AI-returned arrays and concatenates them with no delimiter (Save to Google Sheets node).
- `{{ $json['Overall Score'] }}`, `{{ $json.URL }}`, etc. — reference the Google Sheets node's echoed-back row, keyed by sheet column header (Respond to Webhook node).

## Credentials Used

| Node | Credential Name | Type | Service |
|---|---|---|---|
| OpenRouter Chat Model | OpenRouter account for AI Journal | `openRouterApi` | OpenRouter (LLM API) |
| Save to Google Sheets | Google Sheets account | `googleSheetsOAuth2Api` | Google Sheets |

No secret values are present in the export — only credential names and internal IDs, as expected.

## Input Data

Triggered by an HTTP `POST` to `/website-audit` with a JSON body shaped like:

```json
{
  "website": "https://example.com",
  "email": "requester@example.com"
}
```

Both fields are referenced directly (`$json.body.website`, `$json.body.email`) with no presence/format validation, so a missing `website` field will propagate as `undefined`/empty through the rest of the chain rather than failing fast with a clear error.

## Output Data

Two side effects per execution:
1. **Google Sheets row appended** to `Sheet1` of the "AI Website Copy Auditor & Conversion Advisor" spreadsheet, with columns: URL, Audit Date, Overall Score, Headline Score, Value Prop Score, CTA Score, Trust Score, Clarity Score, Key Issues, Conversion Risks, Recommendations, Summary, User Email.
2. **HTTP response** returned to the original webhook caller: a plain-text formatted audit summary (score breakdown + summary + issues + recommendations).

## Data Transformation

- Raw HTML → plain text (script/style stripped, tags stripped, entities decoded, whitespace normalized, truncated to 4,000 chars) in **Extract & Clean Text Content**.
- AI's raw JSON-formatted string output → structured object (`JSON.parse`) in **Format Audit Results**.
- Structured object → flat spreadsheet row (with array fields collapsed to concatenated strings) in **Save to Google Sheets**.
- Spreadsheet row (object keyed by column header) → plain-text summary message in **Respond to Webhook**.

## API Integrations / External Services

| Service | Node | Method/Operation | Auth | Purpose |
|---|---|---|---|---|
| Target website (arbitrary, caller-supplied URL) | Fetch Website HTML | `GET` | None | Retrieve page HTML to audit |
| OpenRouter | OpenRouter Chat Model | Chat completion (`nvidia/nemotron-nano-9b-v2:free`) | API key (`openRouterApi` credential) | LLM backend for the audit |
| Google Sheets | Save to Google Sheets | Append row | OAuth2 (`googleSheetsOAuth2Api` credential) | Persist audit results |

## AI/LLM Integration

- **Node**: AI Agent - Copy Audit Analysis (`@n8n/n8n-nodes-langchain.agent`, v3)
- **Model provider**: OpenRouter, via the connected OpenRouter Chat Model node
- **Model**: `nvidia/nemotron-nano-9b-v2:free`
- **Memory**: none attached
- **Tools**: none attached (pure text-completion agent, no tool-calling)

**System prompt:**
> You are an expert conversion rate optimization (CRO) consultant specializing in website copy audits. Analyze the provided website copy and evaluate it across five dimensions — Headline Clarity, Value Proposition, Call-to-Action, Trust Signals, and Copy Clarity (each 0–100) — and return the result in a strict JSON schema (see below). Be specific and actionable; focus on conversion impact.

**User prompt template:**
```
Analyze this website copy:

URL: {{ $json.url }}
Word Count: {{ $json.word_count }}

WEBSITE COPY:
{{ $json.text_content }}
```

**Expected structured JSON output schema** (per system prompt instructions):
```json
{
  "overall_score": 0,
  "headline_score": 0,
  "value_prop_score": 0,
  "cta_score": 0,
  "trust_score": 0,
  "clarity_score": 0,
  "key_issues": ["string", "string", "string"],
  "conversion_risks": ["string", "string"],
  "recommendations": ["string", "string", "string"],
  "summary": "string"
}
```

Downstream, **Format Audit Results** consumes this via `JSON.parse(items[0].json.output)` — the workflow assumes the model reliably returns *only* valid JSON with no surrounding text or markdown fencing (see Known Limitations).

## Error Handling

No error handling is configured anywhere in this workflow:
- No node has `continueOnFail`/`onError` set.
- No `errorWorkflow` is configured in workflow `settings`.
- No Error Trigger sub-workflow exists.
- No conditional (IF/Switch) branches exist to handle failure states (e.g. non-200 HTTP response, empty scraped text, malformed AI output).

Any failure at any step (unreachable URL, HTTP timeout, LLM returning non-JSON, Google Sheets auth failure) will fail the entire execution, and the caller will receive n8n's default webhook error response rather than a graceful message.

## Retry Strategy

No retry logic is configured on any node (including **Fetch Website HTML**, the most failure-prone step due to external site availability). `retryOnFail`/`maxTries`/`waitBetweenTries` are not set anywhere in the export.

## Execution Order

`settings.executionOrder: "v1"` — n8n's newer connection-based execution order (rather than legacy top-to-bottom v0). Since this workflow is strictly linear with a single trigger, execution order has no practical effect on branch interleaving here.

## Security Considerations

- The **Webhook** trigger has **no authentication configured** — the `/website-audit` endpoint is publicly callable by anyone who knows (or guesses) the path, with no header/basic/JWT auth.
- The workflow will fetch and forward the content of **any** URL supplied by the caller (`Fetch Website HTML`), with no allow-list/deny-list — this could be used to make the n8n instance issue arbitrary outbound GET requests (SSRF-adjacent risk) if exposed publicly.
- Beyond these two structurally-visible facts, a full security review (rate limiting, input sanitization, credential scoping) is _[to be provided by workflow owner / security reviewer]_.

## Performance Considerations

- `Fetch Website HTML` has a fixed 10-second timeout; very slow target sites will fail the whole run.
- Scraped text is hard-capped at 4,000 characters before being sent to the LLM — this bounds LLM cost/latency but means very long pages are only partially audited (only the first 4,000 characters of cleaned text are seen by the model).
- No batching/pagination is relevant here (single-item flow).

## Execution Time

_[To be provided by workflow owner — requires execution data]_

## Resource Usage

_[To be provided by workflow owner — requires execution data]_

## Testing & Validation

_[To be provided by workflow owner]_

## Troubleshooting

- **No response / timeout from the webhook caller's perspective**: check whether the target site in `Fetch Website HTML` is slow or unreachable — there's a hard 10s timeout and no retry.
- **Google Sheets row has garbled/run-together issue text**: expected given the current expression (`key_issues[0]}}{{key_issues[1]}}...`) has no separator between array items — this is a template issue, not a data issue (see Known Limitations).
- **Execution fails at "Format Audit Results" with a JSON parse error**: the AI model likely returned extra text, markdown code fences, or an incomplete JSON object around/instead of the expected schema. Check the raw `output` field from **AI Agent - Copy Audit Analysis** in the execution log.
- **`url` field is blank in the audit text sent to the AI**: check whether `Extract & Clean Text Content` is correctly receiving `clean_url` — per Known Limitations below, the HTTP Request node's output likely does not carry this field forward.
- **Webhook returns a generic n8n error instead of a friendly message**: expected, since no error-handling branch exists in this version of the workflow.

## Known Limitations

- **No input validation** on the incoming `website`/`email` fields — a missing or malformed `website` value is not caught early; it will silently propagate as an empty/broken URL into `Fetch Website HTML`.
- **`clean_url` likely doesn't survive into `Extract & Clean Text Content`**: n8n's HTTP Request node output typically replaces item JSON with the response data (`data` field only) rather than merging in the upstream item's other fields, so `$input.item.json.clean_url` in the Code node is likely `undefined`, causing the `url` field written into the AI prompt (and intended for logging) to fall back to an empty string.
- **No delimiter between concatenated array items** in the Google Sheets mapping for `Key Issues`, `Conversion Risks`, and `Recommendations` — multiple list items will be written to the sheet as one run-together string.
- **No `try/catch` around `JSON.parse`** in `Format Audit Results` — a non-JSON or malformed AI response will hard-fail the execution with no fallback/error message.
- **No retry or error handling** anywhere in the workflow (see Error Handling / Retry Strategy above) — single point of failure at each step.
- **No authentication on the public webhook**, and no allow-list on which URLs can be fetched.
- **Redundant fields**: `url` and `clean_url` in `Validate URL` are set from the same source and always resolve to (near-)identical values.

## Assumptions

- Assumed the Google Sheets **append** operation echoes back the newly-written row keyed by the sheet's column headers (this is standard n8n Google Sheets node behavior), since `Respond to Webhook` depends on this to reference `$json['Overall Score']`, `$json.URL`, etc.
- Assumed the workflow runs in whatever timezone the hosting n8n instance is configured for, since no explicit `settings.timezone` override is present in the export.
- Assumed "Validate URL" is named for its intent rather than its actual behavior, since no validation logic (format check, required-field check) is present in the node — documented as observed rather than as named.
- Assumed the canvas screenshot and JSON describe the same workflow state; no discrepancies were found between them (node set, connections, and Agent's unconnected Memory/Tool sockets all match).

## Version History / Change Log

_[To be provided by workflow owner]_

## References

_[To be provided by workflow owner]_

## Maintenance Notes

_[To be provided by workflow owner]_
