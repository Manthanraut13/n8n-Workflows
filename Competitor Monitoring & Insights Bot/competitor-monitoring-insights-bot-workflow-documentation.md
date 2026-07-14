# Competitor Monitoring & Insights Bot — Workflow Documentation

| Field | Value |
|---|---|
| Workflow Name | Competitor Monitoring & Insights Bot |
| Workflow ID | `8VoCSaLN3lqRDCX5` |
| Version | Internal revision `4e37cff6-62a4-4f27-adac-784a081c0869` (n8n `versionId`, not a semantic version) — _[human-readable version number to be provided by workflow owner]_ |
| Owner | _[To be provided by workflow owner]_ |
| Last Updated | Not present in this export (no `updatedAt` field). Document generated: July 10, 2026. |
| Active Status | `false` (workflow is currently **deactivated** in n8n) |

## Purpose

This workflow is designed to periodically check a list of competitors' websites for content changes, use an AI agent to assess whether the change is strategically significant, log the finding to Google Sheets, and email a notification when a change is judged medium or high significance.

## Business Use Case

Automates competitive intelligence gathering so a team doesn't have to manually revisit competitor websites on a schedule. _(Inferred from workflow structure and node names — no sticky notes or author commentary are present in the export to confirm this in the author's own words.)_

## Workflow Summary

A Schedule Trigger fires once daily at 09:00. It reads a competitor list from a Google Sheet, then a routing node (named "Split Into Batches" but actually implemented as a **Switch** node, see [Known Limitations](#known-limitations)) selects specific rows to process. For each selected row, the workflow fetches the competitor's website HTML, strips it to plain text, and computes a simple content hash. An IF node checks whether the hash indicates a change worth analyzing. If so, an AI Agent (via OpenRouter) analyzes the content and returns a structured JSON assessment (changes detected, significance level, business impact, recommended actions). A second IF node filters for `medium` or `high` significance; qualifying results are appended to a Google Sheets log, formatted into an email, and sent via Gmail. In all paths, the competitor's stored content hash is updated in the source sheet, and a Loop Over Items node cycles back to process the next selected row.

## Workflow Diagram

See the attached canvas screenshot (`Competitor_Monitoring___Insights_Bot.png`), which was cross-checked against the JSON and matches it. Shape: **linear entry → routing/filter → branching → loop-back**, specifically:

```
Schedule Trigger
   → Load Competitor List
   → Split Into Batches (Switch, 2 hardcoded rules)
   → Fetch Website Content
   → Extract Text Content
   → Detect Changes (IF)
        ├─ true → Prepare AI Analysis Prompt → AI Analysis Agent (+ OpenRouter Chat Model)
        │           → Parse AI Response → Filter Significant Changes (IF)
        │                ├─ true  → Log to Google Sheets → Format Notification Message
        │                │           → Send Notification (Gmail) → Update Competitor Hash
        │                └─ false → Update Competitor Hash (direct)
        └─ false → (no further connection — path ends)
   Update Competitor Hash → Loop Over Items
        ├─ done  → (no further connection — workflow ends)
        └─ loop  → back to Split Into Batches
```

## Trigger Details

**Node:** `Schedule Trigger` (`n8n-nodes-base.scheduleTrigger`, v1.3)

- Type: Interval-based schedule
- Configuration: `triggerAtHour: 9` — fires once per day at 09:00
- Timezone: Not overridden in `settings` — uses the n8n instance's default timezone
- This is the workflow's only entry point (single trigger)

## Execution Flow

1. **Schedule Trigger** fires daily at 09:00 and starts execution.
2. **Load Competitor List** reads rows `A1:D` from the "Sheet1" tab of the linked Google Sheet ("Competitor Monitoring & Insights Bot" spreadsheet).
3. **Split Into Batches** (a Switch node, not a true batching node) evaluates each item's `row_number`:
   - Rule 1: `row_number == 2` → routed to output 0 → **Fetch Website Content**
   - Rule 2: `row_number == 3` → routed to output 1 → **no downstream connection** (dead end)
   - Any other row number → matches no rule → dropped
4. **Fetch Website Content** issues an HTTP GET to `{{ $json.website_url }}` and returns the raw HTML as text. Note: `executeOnce: true` is set (see [Known Limitations](#known-limitations)).
5. **Extract Text Content** (Code node) strips HTML/script/style tags, extracts up to 10 RSS `<title>` values (if RSS XML happened to be present), computes a custom numeric hash of the cleaned text, and outputs `company_name`, `website_url`, `cleaned_text`, `rss_headlines`, `current_hash`, `previous_hash`, `timestamp`.
6. **Detect Changes** (IF node) evaluates: `current_hash != previous_hash` **AND** `previous_hash is empty`.
   - **True** → continues to **Prepare AI Analysis Prompt**
   - **False** → no downstream connection; this branch of execution simply ends (the row is not analyzed and its hash is not updated for that run)
7. **Prepare AI Analysis Prompt** (Set node) builds `company_context` (from the original competitor row) and `content_snapshot` (the cleaned text) for the AI agent.
8. **AI Analysis Agent** (LangChain Agent node, backed by **OpenRouter Chat Model**) sends a system prompt + user prompt to the model and expects a structured JSON response describing detected changes, significance, business impact, insights, and recommended actions.
9. **Parse AI Response** (Code node) extracts and validates the JSON from the agent's raw text output, with a fallback structure if parsing fails.
10. **Filter Significant Changes** (IF node) checks whether `significance_level` is `medium` OR `high`.
    - **True** → **Log to Google Sheets** → **Format Notification Message** → **Send Notification** (Gmail) → **Update Competitor Hash**
    - **False** → goes **directly** to **Update Competitor Hash**, bypassing the logging/notification path (see [Known Limitations](#known-limitations) for a data-reference issue this causes)
11. **Update Competitor Hash** writes `last_checked_hash` back to the "Sheet1" tab, matched on `company_name`.
12. **Loop Over Items** (SplitInBatches node, `reset: true`) either finishes (`done` output — no downstream connection) or loops (`loop` output) back to **Split Into Batches** to process the next matching row.

## Node List

| # | Node Name | Type | Purpose | Disabled |
|---|---|---|---|---|
| 1 | Schedule Trigger | Schedule Trigger | Starts the workflow daily at 09:00 | No |
| 2 | Load Competitor List | Google Sheets (Read) | Loads competitor rows from "Sheet1" | No |
| 3 | Split Into Batches | Switch | Routes only rows 2 and 3 onward (see limitations) | No |
| 4 | Fetch Website Content | HTTP Request | Downloads competitor website HTML | No |
| 5 | Extract Text Content | Code (JavaScript) | Cleans HTML, extracts RSS titles, hashes content | No |
| 6 | Detect Changes | IF | Gates further processing based on hash comparison | No |
| 7 | Prepare AI Analysis Prompt | Set (Edit Fields) | Builds context fields for the AI agent | No |
| 8 | OpenRouter Chat Model | OpenRouter Chat Model (LangChain) | LLM backing the AI Analysis Agent | No |
| 9 | AI Analysis Agent | AI Agent (LangChain) | Analyzes content and returns structured JSON insights | No |
| 10 | Parse AI Response | Code (JavaScript) | Parses/validates the agent's JSON output | No |
| 11 | Filter Significant Changes | IF | Only continues for medium/high significance | No |
| 12 | Log to Google Sheets | Google Sheets (Append) | Writes insight record to "Insights_History" tab | No |
| 13 | Format Notification Message | Set (Edit Fields) | Builds the email body text | No |
| 14 | Send Notification | Gmail | Emails the notification | No |
| 15 | Update Competitor Hash | Google Sheets (Update) | Writes new content hash back to "Sheet1" | No |
| 16 | Loop Over Items | SplitInBatches | Loops back for the next matching competitor row | No |

## Node-by-Node Description

### Schedule Trigger
Fires once daily at 09:00 based on the instance's default timezone. Sole entry point into the workflow.

### Load Competitor List
Reads range `A1:D` from the "Sheet1" tab of the Google Sheet document ID `1PRhpxkhLOO238TIX0kvQ_6TcNQdod5MWXqLNZkg23Ro` ("Competitor Monitoring & Insights Bot"). `returnFirstMatch: false`, so all matching rows are returned as separate items. `alwaysOutputData: false`.

### Split Into Batches
Despite its name, this is a **Switch node**, not `n8n-nodes-base.splitInBatches`. It defines exactly two rules, both comparing `{{ $json.row_number }}` (strict number equality) against literal values `2` and `3`. Only items whose `row_number` is 2 or 3 proceed (to output 0); items matching rule 2 (`row_number == 3`) go to output 1, which has no downstream connection; any other row is dropped entirely. See [Known Limitations](#known-limitations).

### Fetch Website Content
HTTP Request node. Method defaults to GET. URL is set dynamically via `{{ $json.website_url }}`. Response is requested as plain text with a 10-second timeout. **`executeOnce: true`** is set on this node — see [Known Limitations](#known-limitations) for the implications.

### Extract Text Content
Code node (JavaScript). Strips `<script>`/`<style>` blocks and all remaining HTML tags from the fetched page, collapses whitespace, and truncates to 15,000 characters. Attempts to pull up to 10 `<title>` values out of an `rss_data` field (which is never actually populated elsewhere in this workflow — there is no RSS-fetching node present). Computes a simple custom 32-bit rolling hash of the cleaned text (not MD5/SHA — a hand-rolled `hashString()` function) and outputs it as `current_hash`, alongside `previous_hash` (read from `last_checked_hash` on the incoming item, which will be undefined at this point in the flow — see limitations).

### Detect Changes
IF node (v2.2). Condition (AND):
- `current_hash` **notEquals** `previous_hash`
- `previous_hash` **is empty** (unary "empty" operator)

Only the **true** output is wired to a downstream node; the **false** output has no connection, so non-matching items simply stop there.

### Prepare AI Analysis Prompt
Set node. Defines three fields:
- `prompt_text` — a static placeholder string, `"Formatted analysis request"` (not actually used in the AI Agent's prompt)
- `company_context` — `={{ $('Load Competitor List').first([0][2]).json }}` (see [Known Limitations](#known-limitations) regarding this expression)
- `content_snapshot` — `={{ $json.cleaned_text }}`

### OpenRouter Chat Model
LangChain OpenRouter Chat Model node supplying the LLM to the AI Analysis Agent via the `ai_languageModel` connection type.
- Model: `qwen/qwen-2.5-coder-32b-instruct`
- Temperature: `0.3`
- Credential: "OpenRouter account" (`openRouterApi`)

### AI Analysis Agent
LangChain Agent node (`promptType: define`). Builds a user prompt from `company_context` (parsed as JSON) and `content_snapshot`, with a system message instructing the model to act as a competitive intelligence analyst and return a strict JSON object. See [AI/LLM Integration](#aillm-integration) for the full prompt text.

### Parse AI Response
Code node (JavaScript). Reads the agent's raw text output (`$input.item.json.output`), attempts to extract JSON either from a fenced ```` ```json ```` block or the first `{...}` object found, parses it, and maps the result to a normalized output object (`changes_detected`, `significance_level`, `business_impact`, `key_insights`, `recommended_actions`, `summary`). On parse failure, returns a fallback object with `significance_level: 'medium'` and an explanatory summary/impact.

### Filter Significant Changes
IF node (v2.2). Condition (OR): `significance_level equals "medium"` OR `significance_level equals "high"`. True → logging/notification path. False → skips directly to hash update.

### Log to Google Sheets
Google Sheets **Append** operation targeting the "Insights_History" tab (`gid=79908687`) of the same spreadsheet. See [Node Configuration](#node-configuration) for the column mapping detail.

### Format Notification Message
Set node building a single `notification_message` string with emoji section headers (COMPETITOR UPDATE DETECTED, SUMMARY, CHANGES DETECTED, KEY INSIGHTS, BUSINESS IMPACT, RECOMMENDED ACTIONS) using expressions against the parsed AI insight fields.

### Send Notification
Gmail node. Sends to a hardcoded address `storysnippents@gmail.com`, subject `🚨 COMPETITOR UPDATE DETECTED`, body = `{{ $json.notification_message }}`. Uses OAuth2 credential "Gmail account". **Note:** this is a Gmail node, not Telegram — despite Telegram being discussed as an option previously, the actual export uses email.

### Update Competitor Hash
Google Sheets **Update** operation on "Sheet1", matched on `company_name`, writing `last_checked_hash` from `{{ $('Log to Google Sheets').item.json.content_hash }}`. This expression references the **Log to Google Sheets** node explicitly — see [Known Limitations](#known-limitations) for why this breaks on the "false" path from Filter Significant Changes.

### Loop Over Items
SplitInBatches node with `options.reset: true`. Its `done` output has no downstream connection (workflow simply ends there); its `loop` output feeds back into **Split Into Batches** to continue with the next matching row.

## Node Configuration

**Fetch Website Content (HTTP Request)**
- URL: `={{ $json.website_url }}`
- Response format: `text`
- Timeout: `10000` ms
- `executeOnce: true`

**Extract Text Content (Code)** — summarized above; key logic: HTML stripping via regex, custom hash function (`hashString`), 15,000-character truncation.

**Detect Changes (IF)** — conditions detailed above in [Node-by-Node Description](#detect-changes).

**Prepare AI Analysis Prompt (Set)** — fields detailed above.

**AI Analysis Agent — user prompt template:**
```
Analyze changes for: {{ $json.company_context.parseJson().company_name }}

Website: {{ $json.company_context.parseJson().website_url }}

Content snapshot:
{{ $json.content_snapshot }}

RSS Headlines (if any):
{{ $json.company_context.parseJson().rss_feed_url }}

Provide competitive intelligence in JSON format.
```
Note the "RSS Headlines" line actually interpolates `rss_feed_url` (a URL string), not the RSS headline text extracted in the earlier Code node — likely a copy/paste mismatch.

**Parse AI Response (Code)** — summarized above; full fallback/parse logic included.

**Filter Significant Changes (IF)** — OR condition on `significance_level` detailed above.

**Log to Google Sheets — column mapping:**
| Column | Expression |
|---|---|
| `timestamp` | `={{ $('Schedule Trigger')}}` |
| `company_name` | `={{ $('Load Competitor List').item.json.company_name }}` |
| `website_url` | `={{ $('Load Competitor List').item.json.website_url }}` |
| `significance_level` | `={{ $json.significance_level }}` |
| `summary` | `={{ $json.summary }}` |
| `changes_detected` | Concatenation of `changes_detected[0]` through `[6]` (7 indices) |
| `business_impact` | `={{ $json.business_impact }}` |
| `key_insights` | Concatenation of `key_insights[0]` through `[3]` (4 indices) |
| `recommended_actions` | Concatenation of `recommended_actions[0]` through `[4]` (5 indices) |
| `content_hash` | `={{ $('Extract Text Content').item.json.current_hash}}` |

See [Known Limitations](#known-limitations) regarding the `timestamp` expression and the fixed-index array concatenation.

**Format Notification Message (Set)** — single `notification_message` field templated with emoji headers; array fields (`changes_detected`, `key_insights`, `recommended_actions`) are interpolated directly (n8n will stringify the array, typically as a comma-separated list, not the bulleted list shown in earlier drafts of this workflow).

**Update Competitor Hash — column mapping:**
| Column | Expression |
|---|---|
| `company_name` (match key) | `={{ $('Log to Google Sheets').item.json.company_name }}` |
| `last_checked_hash` | `={{ $('Log to Google Sheets').item.json.content_hash }}` |

## Branching Logic

- **Split Into Batches (Switch):** Two hardcoded numeric rules on `row_number` (2 and 3). Rule 1 → Fetch Website Content. Rule 2 → no connection (effectively discarded). Any row not matching either rule is dropped from the flow.
- **Detect Changes (IF):** AND of "hash changed" and "previous hash is empty." True → AI analysis path. False → dead end (no further nodes).
- **Filter Significant Changes (IF):** OR of significance being `medium` or `high`. True → log + notify + update hash. False → update hash only (skipping log/notify), via an expression that depends on a node that didn't run in that branch (see limitations).

## Loop Logic

**Loop Over Items** (SplitInBatches, `reset: true`) sits at the end of the main chain and loops back to **Split Into Batches**. No explicit batch size is configured (default). The loop's exit condition is its `done` output firing, which has no downstream node — so the workflow simply terminates silently on completion of the loop rather than performing any wrap-up action (e.g., a summary notification).

Because **Split Into Batches** is a Switch node keyed on literal `row_number` values (2 and 3) rather than a true "next batch" mechanism, each pass through the loop re-evaluates the same fixed two rows rather than advancing through the full competitor list. See [Known Limitations](#known-limitations).

## Variables & Expressions

Notable expressions (beyond simple `$json.field` mappings):

- `={{ $('Load Competitor List').first([0][2]).json }}` (Prepare AI Analysis Prompt → `company_context`) — unusual syntax; `.first()` does not normally accept an argument like `[0][2]` in n8n's expression API. Likely intended to reference a specific row/column and may not behave as intended.
- `={{ $json.company_context.parseJson().company_name }}`, `.website_url`, `.rss_feed_url` (AI Analysis Agent prompt) — parses the `company_context` string back into an object; depends on `company_context` having been serialized as valid JSON text upstream.
- `={{ $('Schedule Trigger')}}` (Log to Google Sheets → `timestamp` column) — references the entire Schedule Trigger node output object rather than a timestamp value or expression like `$now`; will likely serialize as `[object Object]` or similar rather than a usable date.
- `={{ $('Extract Text Content').item.json.current_hash}}` and `={{ $('Log to Google Sheets').item.json.content_hash }}` — cross-node references pulling specific fields from earlier/prior executed nodes by name.
- `{{ $json.significance_level.toUpperCase() }}` and `{{ $json.timestamp.substring(0,10) }}` (Format Notification Message) — inline JS string methods used directly in expressions.

## Credentials Used

| Node | Credential Name | Credential Type | Service |
|---|---|---|---|
| OpenRouter Chat Model | OpenRouter account | `openRouterApi` | OpenRouter |
| Load Competitor List | Google Sheets account | `googleSheetsOAuth2Api` | Google Sheets |
| Log to Google Sheets | Google Sheets account | `googleSheetsOAuth2Api` | Google Sheets |
| Update Competitor Hash | Google Sheets account | `googleSheetsOAuth2Api` | Google Sheets |
| Send Notification | Gmail account | `gmailOAuth2` | Gmail |

No secret values are present in the export — only credential names and internal IDs, as expected.

## Input Data

The workflow's effective input is the Google Sheet "Sheet1" tab, expected to contain (based on downstream field usage) at minimum: `company_name`, `website_url`, `rss_feed_url`, `last_checked_hash`, plus n8n's automatically supplied `row_number`.

## Output Data

- **Google Sheets — "Insights_History" tab:** one appended row per medium/high-significance finding, with columns as listed in [Node Configuration](#node-configuration).
- **Google Sheets — "Sheet1" tab:** `last_checked_hash` updated per processed competitor row.
- **Email:** sent via Gmail to `storysnippents@gmail.com` for medium/high-significance findings only.

## Data Transformation

- **Extract Text Content:** raw HTML → cleaned plain text (script/style/tag stripped, whitespace-collapsed, truncated to 15,000 chars) + a custom hash of that text.
- **Parse AI Response:** unstructured LLM text output → normalized JSON object with defaulted fields on parse failure.
- **Format Notification Message:** structured insight fields → a single formatted string for the email body.

## API Integrations / External Services

| Service | Node(s) | Method / Operation | Auth |
|---|---|---|---|
| Competitor website (dynamic URL) | Fetch Website Content | HTTP GET | None configured |
| OpenRouter | OpenRouter Chat Model | Chat completion (LangChain) | API key credential |
| Google Sheets API | Load Competitor List, Log to Google Sheets, Update Competitor Hash | Read / Append / Update | OAuth2 credential |
| Gmail API | Send Notification | Send message | OAuth2 credential |

## AI/LLM Integration

- **Provider:** OpenRouter
- **Model:** `qwen/qwen-2.5-coder-32b-instruct`
- **Temperature:** `0.3`
- **Agent type:** LangChain Agent node (`@n8n/n8n-nodes-langchain.agent`) with `promptType: define`
- **System prompt:**
  > You are a competitive intelligence analyst. Analyze website content changes and provide actionable business insights.
  >
  > Your task:
  > 1. Identify what specifically changed (new products, pricing, messaging, features, blog topics)
  > 2. Assess the strategic significance (low/medium/high)
  > 3. Explain potential business impact
  > 4. Recommend response actions
  >
  > Output ONLY valid JSON with this exact structure:
  > ```json
  > {
  >   "changes_detected": ["change 1", "change 2", "change 3"],
  >   "significance_level": "low|medium|high",
  >   "business_impact": "Brief explanation of what this means for our business",
  >   "key_insights": ["insight 1", "insight 2"],
  >   "recommended_actions": ["action 1", "action 2"],
  >   "summary": "One-sentence summary of the change"
  > }
  > ```
  > Be concise. Focus on strategic implications, not technical details.
- **User prompt:** see [Node Configuration](#node-configuration).
- **Tools/Memory:** none connected — the AI Agent node has no `ai_tool` or `ai_memory` inputs wired, despite the canvas showing placeholder "+" connectors for them.
- **Downstream consumption:** the agent's raw text output (`output` field) is parsed by the **Parse AI Response** Code node into the structured fields used throughout the rest of the workflow.

## Error Handling

No node in this workflow has `continueOnFail`/`onError` explicitly configured, and `settings` does not define an `errorWorkflow`. This means:
- A failed HTTP request to a competitor site will halt that execution.
- A malformed AI response is the one exception with built-in resilience — the **Parse AI Response** Code node has a `try/catch` fallback that substitutes default values rather than throwing.

## Retry Strategy

No `retryOnFail`, `maxTries`, or `waitBetweenTries` settings are present on any node (including **Fetch Website Content**, where this would be most relevant). Failed HTTP requests to competitor sites are not retried.

## Logging

Findings that pass the significance filter are logged to the "Insights_History" Google Sheets tab (see [Output Data](#output-data)). Beyond this and n8n's own execution history, there is no separate audit/logging mechanism.

## Execution Order

`settings.executionOrder: "v1"` — the modern connection-based execution order (not the legacy top-to-bottom v0 mode).

## Scheduling Details

Schedule Trigger uses an interval rule with `triggerAtHour: 9`, i.e., **once daily at 09:00**. No explicit timezone override is set in `settings`, so the trigger uses whatever timezone is configured on the n8n instance itself.

## Dependencies

This workflow depends on the continued availability and correct configuration of:
- The linked Google Sheet (`1PRhpxkhLOO238TIX0kvQ_6TcNQdod5MWXqLNZkg23Ro`), specifically its "Sheet1" and "Insights_History" tabs and their column headers
- The OpenRouter API and the specific model `qwen/qwen-2.5-coder-32b-instruct` remaining available
- The Gmail account credential remaining authorized
- The target competitor website(s) being reachable and returning HTML within the 10-second timeout

## Security Considerations

- The trigger is schedule-based, not webhook-based, so there is no inbound authentication surface to assess.
- The Gmail recipient address (`storysnippents@gmail.com`) is hardcoded in the node parameters rather than sourced from a variable or credential — anyone with workflow edit access can see it in plain text (this is normal for a recipient address, but worth noting for handover).
- No secrets are present in the JSON export; all three credentials (OpenRouter, Google Sheets, Gmail) are referenced by name/ID only.
- _[Further security review/sign-off to be provided by workflow owner or security reviewer]_

## Performance Considerations

- **Fetch Website Content** has `executeOnce: true`, which caps this node to a single execution per workflow run regardless of how many items reach it — see [Known Limitations](#known-limitations).
- **Split Into Batches** limits processing to at most 2 rows (`row_number` 2 and 3) per run, which bounds the workflow's per-run load but also bounds its actual competitor coverage.
- No pagination or rate-limiting is configured on the Google Sheets or Gmail nodes.

## Execution Time

_[To be provided by workflow owner — requires execution data]_

## Resource Usage

_[To be provided by workflow owner — requires execution data]_

## Testing & Validation

_[To be provided by workflow owner]_

## Troubleshooting

- **No competitors are ever processed:** Check that competitor rows in "Sheet1" actually occupy `row_number` 2 or 3 — the **Split Into Batches** Switch node silently drops every other row.
- **Only one competitor's content ever appears, even with multiple rows matching:** Check the `executeOnce: true` setting on **Fetch Website Content** — this is very likely why.
- **Detect Changes never fires as expected:** Its condition requires `previous_hash` to be **empty**, not populated — if you intended to alert on hash changes for previously-seen competitors, this condition is inverted from that intent. See [Known Limitations](#known-limitations).
- **Update Competitor Hash errors out or writes blank/undefined values on low-significance runs:** its expressions reference `$('Log to Google Sheets')`, but the false branch of **Filter Significant Changes** routes directly to **Update Competitor Hash** without ever running **Log to Google Sheets** in that execution path.
- **`timestamp` column in Insights_History looks wrong (e.g., `[object Object]`):** the mapped expression is `={{ $('Schedule Trigger')}}`, which references the whole trigger node's output item rather than a date/time value.
- **No email arrives despite a real competitor change:** confirm the AI Agent actually returned `significance_level: "medium"` or `"high"` — anything else (including its `"low"` default or a parse failure defaulting to `"medium"`) determines whether **Filter Significant Changes** lets the item through.

## Known Limitations

- **Naming/implementation mismatch:** the node named "Split Into Batches" is implemented as a **Switch** node with two hardcoded row-number rules (2 and 3), not `n8n-nodes-base.splitInBatches`. It does not dynamically iterate all rows returned by **Load Competitor List** — only rows with `row_number` exactly 2 or 3 are ever processed, and rule 2's matched items have no downstream connection at all.
- **`executeOnce: true` on Fetch Website Content:** this forces the node to run only once per workflow execution using a single item's data, which conflicts with the apparent intent of fetching content for multiple competitor rows in the same run.
- **Detect Changes condition looks inverted:** it requires `previous_hash` to be **empty** in addition to the hashes differing — this means it will only proceed on what looks like a "never checked before" state, not on a genuine content change for an already-tracked competitor (where `previous_hash` would be populated).
- **Broken cross-node reference on the low-significance path:** **Update Competitor Hash** pulls `company_name`/`content_hash` from `$('Log to Google Sheets')`, but the false branch of **Filter Significant Changes** reaches **Update Competitor Hash** without **Log to Google Sheets** having executed in that run — this expression will not resolve as intended on that path.
- **`timestamp` mapping in Log to Google Sheets** references the entire **Schedule Trigger** node output rather than an actual timestamp expression (e.g. `{{ $now }}` or `{{ $json.timestamp }}` from Extract Text Content).
- **Fixed-index array concatenation** for `changes_detected` (indices 0–6), `key_insights` (0–3), and `recommended_actions` (0–4) in the Google Sheets column mapping will produce `undefined` text for any index beyond what the AI actually returned, and will silently truncate if the AI returns more items than the indices cover.
- **RSS headline extraction is effectively dead code:** **Extract Text Content** looks for `$input.item.json.rss_data`, but no node in this workflow fetches or populates `rss_data` — there is no RSS-fetching HTTP Request node in this export (despite one being part of the originally discussed design).
- **`company_context` expression syntax is non-standard:** `$('Load Competitor List').first([0][2]).json` is not a typical n8n expression pattern and may not resolve as intended.
- **Notification channel is Gmail, not Telegram:** the actual implementation emails `storysnippents@gmail.com` rather than sending a Telegram message, differing from the Telegram-based design discussed prior to this export.
- **No retry logic** on the HTTP Request to competitor websites, so transient failures (timeouts, temporary 5xx) will fail that run outright.
- **No error notification path** — if any node errors, there is no branch or error workflow configured to alert anyone.

## Assumptions

- Assumed the linked Google Sheet's "Sheet1" tab contains a header row and that data rows begin at row 2 (consistent with the Switch node's hardcoded `row_number` values of 2 and 3).
- Assumed `company_context` in **Prepare AI Analysis Prompt** is expected to serialize to a JSON string (since it's later parsed with `.parseJson()`), even though the field `type` is declared as `"string"` on a value that's actually an object expression.
- Assumed the workflow's intended design (per node names and structure) was to iterate over **all** competitor rows via true batching, and that the current Switch-based "Split Into Batches" node is either a work-in-progress or a testing configuration rather than final intended behavior — flagged rather than corrected, since only the workflow owner can confirm intent.
- Assumed the instance's default timezone applies to the Schedule Trigger, since no override is present in `settings`.

## Version History / Change Log

_[To be provided by workflow owner]_

## References

_[To be provided by workflow owner]_

## Maintenance Notes

_[To be provided by workflow owner]_
