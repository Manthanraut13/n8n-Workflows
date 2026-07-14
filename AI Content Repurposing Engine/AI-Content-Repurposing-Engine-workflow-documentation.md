# AI Content Repurposing Engine — Workflow Documentation

| Field | Value |
|---|---|
| Workflow Name | AI Content Repurposing Engine |
| Workflow ID | `x7wKwkglpAiYAuhE` |
| Version | Internal revision `f0406c2c-547e-4d4f-9843-5a392fa54fb1` (n8n `versionId`, not a semantic version). Suggest **v1.0** for this initial documentation. |
| Owner | _[To be provided by workflow owner]_ |
| Last Updated | Workflow `updatedAt` not present in export. Document generated: see file metadata / today's date. |
| Active Status | `false` (currently deactivated in the export) |

## Purpose

This workflow accepts a single piece of long-form content via webhook and uses a chain of AI Agent nodes (via OpenRouter) to automatically repurpose it into five platform-specific content assets — a LinkedIn post, a Twitter/X thread, an Instagram caption, an email newsletter, and a list of reusable key takeaways. Results are logged to Google Sheets and a summary notification is sent via Telegram.

## Business Use Case

Automates the manual, repetitive work of adapting one piece of long-form content (blog post, article, transcript) into multiple platform-native formats, reducing content-team turnaround time and keeping messaging consistent across channels. _(Inferred from workflow structure — no sticky notes or documentation were present in the export to confirm the exact business justification; confirm with workflow owner.)_

## Workflow Summary

A client sends a `POST` request to the `/content-repurpose` webhook with the source content plus optional topic, tone, and audience fields. The **Set - Normalize Input** node standardizes these fields and computes a content length and timestamp. An **IF - Content Length Check** node gates execution: content longer than 500 characters proceeds to analysis; shorter content is routed to an error path. On the success path, an **AI Agent - Content Analyzer** node extracts the main theme, key points, hooks, and sentiment from the source content. This analysis fans out in parallel to four AI Agent generator nodes (LinkedIn, Twitter thread, Instagram caption, email newsletter) plus a fifth AI Agent that extracts key takeaways — each paired with its own OpenRouter Chat Model and its own Code node to parse the JSON returned by the model. All five parsed outputs are recombined by position in a **Merge** node, consolidated by **Set - Merge All Outputs**, written as a new row in **Google Sheets - Save Results**, and summarized in a **Telegram - Send Notification** message. If the content-length check fails, **Error Fields** captures error metadata and **Telegram - Error Notification** sends a short failure alert instead.

## Workflow Diagram

See the attached canvas screenshot (`AI_Content_Repurposing_Engine.png`) for the visual layout. Shape: **linear intake → single-branch IF gate → fan-out to 5 parallel AI Agent chains → 5-way positional Merge → linear consolidation, logging, and notification**, with a separate short error-notification branch off the IF node's false output.

```
Webhook
  └─ Set - Normalize Input
       └─ IF - Content Length Check
            ├─ (true) AI Agent - Content Analyzer → Parse output
            │        ├─ AI Agent - LinkedIn Generator        → Parse output1  ─┐
            │        ├─ AI Agent - Twitter Thread Generator  → Parse output2  ─┤
            │        ├─ AI Agent - Instagram Caption Gen.    → Parse output3  ─┼─ Merge (5-in, combine by position)
            │        ├─ AI Agent - Email Newsletter Gen.     → Parse output4  ─┤
            │        └─ AI Agent - Key Takeaways Extractor   → Parse output5  ─┘
            │                                                        └─ Set - Merge All Outputs
            │                                                             └─ Google Sheets - Save Results
            │                                                                  └─ Telegram - Send Notification
            └─ (false) Error Fields → Telegram - Error Notification
```

Each AI Agent node has its own dedicated OpenRouter Chat Model sub-node feeding it via the `ai_languageModel` connection (not shown above for brevity — see Node List).

## Trigger Details

- **Trigger type:** Webhook (`n8n-nodes-base.webhook`, v2.1)
- **HTTP Method:** `POST`
- **Path:** `content-repurpose`
- **Response Mode:** `lastNode` (the HTTP response returned to the caller is the output of the last node executed in the flow that runs — i.e. either the Telegram success path or the Telegram error path, depending on branch)
- **Authentication:** None configured (`options: {}` — no header/basic/JWT auth on the webhook)
- **Expected payload shape:** `{ "content": "...", "topic": "...", "tone": "...", "audience": "..." }`, read via `$json.body.*` in downstream nodes

This is the sole entry point into the workflow.

## Execution Flow

1. **Webhook** receives a `POST` request to `/content-repurpose` with the content payload.
2. **Set - Normalize Input** copies `body.content`, `body.topic`, `body.audience`, `body.tone` into the item, and adds `timestamp` (current ISO time) and `content_length` (character count of `body.content`).
3. **IF - Content Length Check** evaluates whether `body.content.length > 500`.
   - **True branch:** proceeds to step 4.
   - **False branch:** jumps to step 9 (error path).
4. **AI Agent - Content Analyzer** (OpenRouter Chat Model) analyzes the source content and returns JSON with `main_theme`, `key_points`, `hooks`, `sentiment`.
5. **Parse output** (Code node) safely `JSON.parse`s the analyzer's raw text output into a clean object, defaulting to empty values on parse failure.
6. The parsed analysis fans out to **five parallel AI Agent branches**, each with its own OpenRouter Chat Model and its own Code-node parser:
   - **AI Agent - LinkedIn Generator** → **Parse output1** → `linkedin_post`
   - **AI Agent - Twitter Thread Generator** → **Parse output2** → `twitter_thread` (array)
   - **AI Agent - Instagram Caption Generator** → **Parse output3** → `instagram_caption`
   - **AI Agent - Email Newsletter Generator** → **Parse output4** → `email_subject`, `email_body`
   - **AI Agent - Key Takeaways Extractor** → **Parse output5** → `key_takeaways` (joined string), `key_takeaways_list` (array)
7. **Merge** (5 inputs, `combine` mode, `combineByPosition`) recombines the five parsed outputs into a single item, matched by their execution position (item 1 from each of the 5 inputs merged together).
8. **Set - Merge All Outputs** assembles the final structured record (original content, topic, timestamp, all five generated assets, and a `status: "completed"` flag) → **Google Sheets - Save Results** appends a row → **Telegram - Send Notification** sends a summary message. Execution ends here for the success path.
9. **(Error path)** **Error Fields** builds an error payload (`error`, `message`, `content_length`, `timestamp`) → **Telegram - Error Notification** sends a fixed short alert message. Execution ends here for the error path.

## Node List

| # | Node Name | Type | Purpose | Disabled |
|---|---|---|---|---|
| 1 | Webhook | Webhook | Entry point; receives content-repurposing requests | No |
| 2 | Set - Normalize Input | Set (Edit Fields) | Normalizes payload fields, computes timestamp & content length | No |
| 3 | IF - Content Length Check | IF | Gates execution on content length > 500 chars | No |
| 4 | AI Agent - Content Analyzer | LangChain Agent | Extracts theme, key points, hooks, sentiment | No |
| 5 | OpenRouter Chat Model | LangChain OpenRouter Chat Model | LLM backing the Content Analyzer agent | No |
| 6 | Parse output | Code | Parses Content Analyzer's JSON output | No |
| 7 | AI Agent - LinkedIn Generator | LangChain Agent | Generates LinkedIn post | No |
| 8 | OpenRouter Chat Model1 | LangChain OpenRouter Chat Model | LLM backing the LinkedIn Generator agent | No |
| 9 | Parse output1 | Code | Parses LinkedIn Generator's JSON output | No |
| 10 | AI Agent - Twitter Thread Generator | LangChain Agent | Generates 5–7 tweet thread | No |
| 11 | OpenRouter Chat Model2 | LangChain OpenRouter Chat Model | LLM backing the Twitter Thread Generator agent | No |
| 12 | Parse output2 | Code | Parses Twitter Thread Generator's JSON output | No |
| 13 | AI Agent - Instagram Caption Generator | LangChain Agent | Generates Instagram caption | No |
| 14 | OpenRouter Chat Model3 | LangChain OpenRouter Chat Model | LLM backing the Instagram Caption Generator agent | No |
| 15 | Parse output3 | Code | Parses Instagram Caption Generator's JSON output | No |
| 16 | AI Agent - Email Newsletter Generator | LangChain Agent | Generates email subject + body | No |
| 17 | OpenRouter Chat Model4 | LangChain OpenRouter Chat Model | LLM backing the Email Newsletter Generator agent | No |
| 18 | Parse output4 | Code | Parses Email Newsletter Generator's JSON output | No |
| 19 | AI Agent - Key Takeaways Extractor | LangChain Agent | Extracts 5–8 reusable key takeaways | No |
| 20 | OpenRouter Chat Model5 | LangChain OpenRouter Chat Model | LLM backing the Key Takeaways Extractor agent | No |
| 21 | Parse output5 | Code | Parses Key Takeaways Extractor's JSON output | No |
| 22 | Merge | Merge | Recombines all 5 parallel branches by position | No |
| 23 | Set - Merge All Outputs | Set (Edit Fields) | Assembles final record for storage/notification | No |
| 24 | Google Sheets - Save Results | Google Sheets | Appends one row per run to the results sheet | No |
| 25 | Telegram - Send Notification | Telegram | Sends success summary notification | No |
| 26 | Error Fields | Set (Edit Fields) | Builds error payload for short-content case | No |
| 27 | Telegram - Error Notification | Telegram | Sends error alert notification | No |

## Node-by-Node Description

### Webhook
Receives `POST /content-repurpose` requests. No authentication configured. Response is deferred until the last node in the executed path finishes (`responseMode: lastNode`), so the HTTP caller receives whatever the final Telegram node returns.

### Set - Normalize Input
Copies `body.content`, `body.topic`, `body.audience`, `body.tone` from the incoming webhook payload into equivalent nested `body.*` fields on the item, and adds two computed fields: `timestamp` (`{{ $now.toISO() }}`) and `content_length` (`{{ $json.body.content.length }}`). Passes this normalized item downstream.

### IF - Content Length Check
Single condition: `{{ $json.body.content.length }} > 500` (number comparison, strict type validation). True output leads into the AI analysis chain; false output leads to the error-notification branch.

### AI Agent - Content Analyzer
LangChain Agent node (v3) using a `define`-type prompt. Takes the raw content plus topic/tone/audience context and instructs the model to return **only** a JSON object with `main_theme`, `key_points` (5 items), `hooks` (3 items), and `sentiment`. Backed by **OpenRouter Chat Model** (see AI/LLM Integration section for the model name and full prompts).

### Parse output
Code node. Reads `items[0].json.output` (the AI Agent's raw text), attempts `JSON.parse`, and returns a normalized object with `main_theme`, `key_points` (array, defaults to `[]`), `hooks` (array, defaults to `[]`), and `sentiment` — falling back to empty values if parsing fails, so a malformed model response doesn't crash the workflow.

### AI Agent - LinkedIn Generator
Takes the original content (`$('IF - Content Length Check').item.json.body.content`) plus the parsed analysis (`main_theme`, `key_points`, `hooks`) and target audience/tone, and generates a 150–250 word LinkedIn post with hashtags on the final line only. Returns JSON `{ "linkedin_post": "..." }`. Backed by **OpenRouter Chat Model1**.

### Parse output1
Code node. Parses the LinkedIn Generator's raw output; on JSON parse failure it falls back to returning the raw string itself as `linkedin_post` (rather than an empty string, unlike the other parser nodes) — a slightly inconsistent fallback strategy versus its sibling parser nodes.

### AI Agent - Twitter Thread Generator
Generates a 5–7 tweet thread from the original content and the first five key points, with structural constraints (hook tweet, one idea per tweet, ≤280 chars each, CTA closer). Returns JSON `{ "twitter_thread": [...] }`. Backed by **OpenRouter Chat Model2**.

### Parse output2
Code node. Parses the Twitter Thread Generator's output; defaults `twitter_thread` to `[]` on missing/invalid output or parse failure.

### AI Agent - Instagram Caption Generator
Generates a 100–150 word Instagram caption (attention-grabbing opener, 1–2 emojis, engagement CTA, 10–15 hashtags appended after a line break) from the original content and main theme only (does not receive the full key-points list). Returns JSON `{ "instagram_caption": "..." }`. Backed by **OpenRouter Chat Model3**.

### Parse output3
Code node. Parses the Instagram Caption Generator's output; defaults to an empty string on missing/invalid output or parse failure.

### AI Agent - Email Newsletter Generator
Generates a subject line (50–60 chars) and a 200–300 word email body (personal greeting, subheadings, 3–4 takeaways, CTA) from the original content and the first five key points. Returns JSON `{ "email_newsletter": { "subject": "...", "body": "..." } }`. Backed by **OpenRouter Chat Model4**.

### Parse output4
Code node. Parses the Email Newsletter Generator's nested output and flattens it into two top-level fields, `email_subject` and `email_body`, defaulting both to empty strings on failure.

### AI Agent - Key Takeaways Extractor
Extracts 5–8 standalone, reusable takeaway bullets (15–25 words each) from the original content and the analysis's key points. Returns JSON `{ "key_takeaways": [...] }`. Backed by **OpenRouter Chat Model5**.

### Parse output5
Code node. Parses the Key Takeaways Extractor's output into `key_takeaways_list` (the raw array) and `key_takeaways` (the same array joined with `" | "`), both defaulting to empty on failure.

### Merge
Combines the 5 parallel branches (LinkedIn, Twitter, Instagram, Email, Key Takeaways parser outputs) into a single item using `combine` mode with `combineByPosition` — i.e. it pairs up the Nth item from each of the 5 inputs, not by any matching key. This is safe here because every branch produces exactly one item per run.

### Set - Merge All Outputs
Assembles the final row-ready record from the merged item and from earlier nodes referenced directly by name:
- `original_content`, `topic`, `timestamp` — pulled from `$('IF - Content Length Check').item.json.body.*` / `.timestamp` (i.e., re-read from the pre-AI item, not from the merged AI outputs)
- `linkedin_post`, `instagram_caption`, `email_subject`, `email_body` — pulled from the merged item (`$json.*`)
- `twitter_thread[0]` — set to a single string containing all 6 tweet slots (`twitter_thread[0]` through `[5]`) joined with double line breaks. **Note:** because only index `[0]` is assigned, this produces a one-element array containing the whole joined thread text, not an array of individual tweets.
- `key_takeaways_list[0]` — similarly set to the already-joined `key_takeaways` string from Parse output5, producing a one-element array rather than a true list of takeaways.
- `status` — hardcoded to `"completed"`

See **Known Limitations** for the implications of the `[0]`-only array assignments.

### Google Sheets - Save Results
Appends one row per workflow run to a Google Sheet. See **Database/Sheets Operations** below for the exact column mapping.

### Telegram - Send Notification
Sends a Markdown-formatted summary message to a fixed Telegram chat, including topic, completion time, and preview snippets of each generated asset. See **Notification Message Format** below for the full template.

### Error Fields
Builds a simple error record (`error`, `message`, `content_length`, `timestamp`) when content is too short, pulling `content_length` and `timestamp` from `Set - Normalize Input`'s output.

### Telegram - Error Notification
Sends a fixed plain-text message ("Error content length is small") to the same Telegram chat as the success notification. Does not include any of the dynamic fields built by Error Fields.

## Branching Logic

**IF - Content Length Check** is the only conditional node in the workflow.

- **Condition:** `{{ $json.body.content.length }} > 500` (number, strict type validation, AND combinator with a single condition)
- **True branch →** AI Agent - Content Analyzer (full repurposing pipeline runs)
- **False branch →** Error Fields → Telegram - Error Notification (short failure alert; no content is repurposed or stored)

## Variables & Expressions

Notable expressions used across the workflow:

- `{{ $now.toISO() }}` / `{{ $now.format('MMM DD, YYYY - HH:mm') }}` — timestamps for storage and for the Telegram message.
- `{{ $json.body.content.length }}` — used both in Set - Normalize Input (to compute `content_length`) and independently re-evaluated in the IF node's condition.
- `{{ $('IF - Content Length Check').item.json.body.content }}` — multiple AI Agent prompts reference the original content directly from the IF node's item rather than from the Set - Normalize Input node, ensuring they always get the pre-branch payload regardless of what later Set nodes do to the item shape.
- `{{ $('Set - Merge All Outputs').item.json.topic }}`, `.linkedin_post`, `.twitter_thread[0]`, `.instagram_caption`, `.email_subject` — used throughout the Telegram success message to pull preview snippets, with `.substring(0, N)` truncation for long fields.
- `{{ $json['Key Takeways'] }}` in the Telegram success message — note this references the **Google Sheets node's output** (which echoes back the column named `"Key Takeways"` as written, including the sheet's typo), not a field from Set - Merge All Outputs directly. This only works because Google Sheets - Save Results runs immediately before Telegram - Send Notification and its output item carries the written column names as keys.

## Credentials Used

| Node | Credential Name | Credential Type | Service |
|---|---|---|---|
| OpenRouter Chat Model | OpenRouter account for AI Journal | `openRouterApi` | OpenRouter |
| OpenRouter Chat Model1 | OpenRouter account for AI Journal | `openRouterApi` | OpenRouter |
| OpenRouter Chat Model2 | OpenRouter account for AI Journal | `openRouterApi` | OpenRouter |
| OpenRouter Chat Model3 | OpenRouter account for AI Journal | `openRouterApi` | OpenRouter |
| OpenRouter Chat Model4 | OpenRouter account for AI Journal | `openRouterApi` | OpenRouter |
| OpenRouter Chat Model5 | OpenRouter account for AI Journal | `openRouterApi` | OpenRouter |
| Google Sheets - Save Results | Google Sheets account | `googleSheetsOAuth2Api` | Google Sheets |
| Telegram - Send Notification | Telegram account for AI Journal | `telegramApi` | Telegram Bot API |
| Telegram - Error Notification | Telegram account for AI Journal | `telegramApi` | Telegram Bot API |

No secret values are present in the export (n8n never exports them); only credential names/IDs are referenced above.

## Input Data

Triggered by a `POST` request to `/content-repurpose` with a JSON body shaped as:

```json
{
  "content": "Long-form source content (string, required — must exceed 500 characters)",
  "topic": "Optional topic/title (string)",
  "tone": "Optional tone descriptor, e.g. educational/casual/authority (string)",
  "audience": "Optional target audience description (string)"
}
```

All fields are read via `$json.body.*`, confirming they are expected in the HTTP request body (not query string or headers). No optional-field defaulting is done in Set - Normalize Input — if `topic`, `tone`, or `audience` are omitted, downstream prompts will contain empty/undefined values.

## Output Data

**Success path:**
- One new row appended to the configured Google Sheet (see schema below).
- One Telegram message sent to the configured chat with preview snippets of all five generated assets.
- HTTP response to the original webhook caller = output of the Telegram - Send Notification node (per `responseMode: lastNode`).

**Error path (content ≤ 500 characters):**
- No Google Sheets row is written.
- One Telegram message sent with a fixed short error text.
- HTTP response to the original webhook caller = output of the Telegram - Error Notification node.

## Data Transformation

- **Set - Normalize Input**: reshapes flat webhook body fields into nested `body.*` fields and adds computed `timestamp`/`content_length`.
- **Parse output(1–5)**: each converts an AI Agent's raw text response (expected to be a JSON string) into a structured n8n item, with safe fallbacks on parse failure.
- **Merge**: flattens 5 parallel single-item branches into 1 combined item by positional pairing.
- **Set - Merge All Outputs**: final reshaping into the flat record written to Google Sheets, including the `twitter_thread[0]` / `key_takeaways_list[0]` array-index assignment pattern noted in Known Limitations.

## AI/LLM Integration

**Provider:** OpenRouter (via `@n8n/n8n-nodes-langchain.lmChatOpenRouter`, one dedicated model sub-node per AI Agent)

**Model used (all 6 agents):** `nvidia/nemotron-nano-9b-v2:free`

All AI Agent nodes are `@n8n/n8n-nodes-langchain.agent` (v3) using `promptType: define` (a fixed prompt template with inline expressions, not a chat-history-driven agent). None of the agents have tools attached — they are single-shot prompt→JSON-response generators, not tool-calling agents.

| Agent Node | System Message (summary) | User Prompt (summary) | Expected JSON Output |
|---|---|---|---|
| AI Agent - Content Analyzer | Content analysis/synthesis expert; return only JSON | Analyze content + topic/tone/audience; extract theme, key points, hooks, sentiment | `{main_theme, key_points[], hooks[], sentiment}` |
| AI Agent - LinkedIn Generator | Senior LinkedIn strategist; professional, human tone; JSON only | Write 150–250 word LinkedIn post from content + analysis; hashtags on final line only | `{linkedin_post}` |
| AI Agent - Twitter Thread Generator | Twitter/X thread expert for educational threads; JSON only | Write 5–7 tweet thread, hook → ideas → CTA, ≤280 chars/tweet | `{twitter_thread: []}` |
| AI Agent - Instagram Caption Generator | Instagram content creator, high-engagement educational captions; JSON only | Write 100–150 word caption, 1–2 emojis, 10–15 hashtags at end | `{instagram_caption}` |
| AI Agent - Email Newsletter Generator | Professional email newsletter writer, warm/personal tone; JSON only | Write subject (50–60 chars) + 200–300 word body with takeaways and CTA | `{email_newsletter: {subject, body}}` |
| AI Agent - Key Takeaways Extractor | (system message not distinctly separated in export; follows same JSON-only convention) | Extract 5–8 reusable, standalone takeaway bullets | `{key_takeaways: []}` |

Full verbatim prompt text for each node is preserved in the workflow JSON (`parameters.text` and `parameters.options.systemMessage` per node) and can be pulled directly from the export if exact wording needs to be audited or edited.

**Downstream consumption:** Each agent's raw text response lands in `$json.output` and is parsed by its dedicated Code node (see Node-by-Node Description) into the structured fields consumed by Set - Merge All Outputs, Google Sheets, and Telegram.

## Google Sheets Schema

**Spreadsheet:** "AI Content Repurposing Engine" (`documentId: 1H3rn3hN-aky0OuxVf5XHN2pEtWTOHVo2kko6X1_YscA`)
**Sheet/tab:** `Sheet1` (`gid=0`)
**Operation:** Append

| Column Name (as configured) | Source Expression | Purpose |
|---|---|---|
| TimeStamp | `{{ $json.timestamp }}` | When the run completed |
| Topic | `{{ $json.topic }}` | Content topic/title for reference |
| Orginal content *(sic — typo in the sheet header as configured)* | `{{ $json.original_content }}` | Full source content for traceability |
| LinkedIn Post | `{{ $json.linkedin_post }}` | Generated LinkedIn post |
| Twitter Thread | `{{ $json.twitter_thread[0] }}` | Full thread text (all tweets pre-joined into one string — see Known Limitations) |
| Instagram Caption | `{{ $json.instagram_caption }}` | Generated Instagram caption |
| Email Subject | `{{ $json.email_subject }}` | Generated email subject line |
| Email Body | `{{ $json.email_body }}` | Generated email body |
| Key Takeways *(sic — typo in the sheet header as configured)* | `{{ $json.key_takeaways_list[0] }}` | Joined key-takeaways string |
| Status | `{{ $json.status }}` | Always `"completed"` for rows reaching this node |

Column matching mode is `defineBelow` with no `matchingColumns` configured, consistent with a pure append (no upsert/update-by-key behavior).

## Notification Message Format

**Telegram - Send Notification** (Markdown parse mode implied by `*bold*` syntax; note `additionalFields` does not explicitly set `parseMode`, so verify this renders as intended in your n8n Telegram node version):

```
✅ *Content Repurposing Complete!*

📝 *Topic:* {{ $('Set - Merge All Outputs').item.json.topic }}

⏰ *Completed:* {{ $now.format('MMM DD, YYYY - HH:mm') }}

📊 *Generated Assets:*  *LinkedIn:* {{ $('Set - Merge All Outputs').item.json.linkedin_post.substring(0, 150) }}...

*Twitter Thread:* {{ $('Set - Merge All Outputs').item.json.twitter_thread[0].substring(0, 100) }}... ({{ $('Set - Merge All Outputs').item.json.twitter_thread.length }} tweets)

*Instagram:* {{ $('Set - Merge All Outputs').item.json.instagram_caption.substring(0, 100) }}...

*Email Subject:* {{ $('Set - Merge All Outputs').item.json.email_subject }}

*Key Takeaways:* {{ $json['Key Takeways'] }}

insights extracted 🔗 All content saved to Google Sheets!
```

**Telegram - Error Notification** (plain text, fixed string, chat ID same as success notification):

```
Error content length is small
```

## Error Handling

- The only structural error handling is the **IF - Content Length Check** gate, which diverts short content to a dedicated error/notification branch instead of running it through the AI pipeline.
- None of the 27 nodes have `continueOnFail`/`onError` settings configured in the export — a runtime failure in any node (e.g. an OpenRouter API error, a Google Sheets auth failure, a Telegram send failure) will fail the entire execution rather than degrading gracefully.
- No workflow-level `errorWorkflow` is configured in `settings` — failed executions are not routed to a separate error-handling workflow; they'll only appear in n8n's own execution log.
- The Code node "Parse output" nodes each contain their own try/catch around `JSON.parse`, so a malformed AI response degrades to empty/default field values rather than throwing — this is the workflow's main defense against unpredictable LLM output, but it is silent (no flag is raised downstream when a parse failure occurs).

## Retry Strategy

No `retryOnFail`, `maxTries`, or `waitBetweenTries` settings are configured on any node in the export — including the six OpenRouter Chat Model nodes and the Google Sheets/Telegram nodes, which are the most likely points of transient failure (rate limits, network blips). **No retry logic is currently configured anywhere in this workflow.**

## Failure Notifications

**Telegram - Error Notification** fires only on the IF node's false branch (content too short) — it is not a general-purpose failure notifier and will not fire for AI Agent errors, Google Sheets write failures, or other runtime exceptions, since none of those failure modes have dedicated error-output routing in this workflow.

## Execution Order

`settings.executionOrder: "v1"` — nodes execute using n8n's newer connection-based execution order (as opposed to legacy top-to-bottom `v0`), meaning the 5 parallel AI Agent branches genuinely run based on graph dependencies/readiness rather than a fixed visual top-to-bottom sequence.

## Security Considerations

- The **Webhook** trigger has no authentication configured (`options: {}` — no header auth, basic auth, or JWT validation), meaning the `/content-repurpose` endpoint is publicly callable by anyone who knows the URL. Consider adding header-auth or an API-key check if this workflow is internet-reachable.
- Telegram `chatId` values are hardcoded directly in node parameters (`7350267365`) rather than sourced from credentials or environment variables — low risk since a chat ID isn't a secret, but it does mean the notification destination can't be changed without editing the workflow.
- All API credentials (OpenRouter, Google Sheets, Telegram) are referenced by ID/name only, as expected; no secret values are present in the export.
- _[Further security review / sign-off to be provided by workflow owner or security reviewer]_

## Performance Considerations

- Five AI Agent branches run in parallel following the Content Analyzer step, which should reduce total latency compared to a fully sequential chain, but total run time is still bounded by the slowest of the five OpenRouter calls plus the initial analysis call — i.e. at least 2 sequential LLM round-trips per execution.
- No batching, pagination, or rate-limiting configuration is present; a burst of concurrent webhook calls could hit OpenRouter's free-tier rate limits with no retry/backoff configured (see Retry Strategy).
- No SplitInBatches/loop node is present — each webhook call processes exactly one content item per execution.

## Execution Time

_[To be provided by workflow owner — requires execution data]_

## Resource Usage

_[To be provided by workflow owner — requires execution data]_

## Testing & Validation

_[To be provided by workflow owner]_

## Troubleshooting

- **No Telegram message arrives after a successful-looking run:** check that the Google Sheets - Save Results node actually appended a row and returned column data — the success Telegram message depends on `$json['Key Takeways']` coming from the Google Sheets node's own output, so a Sheets API failure or schema mismatch will break this specific field (and may fail the node entirely, per Error Handling above).
- **Twitter thread or key takeaways look wrong in Google Sheets:** remember that `Set - Merge All Outputs` stores these as single-element arrays (`twitter_thread[0]`, `key_takeaways_list[0]`) containing pre-joined text, not true per-tweet/per-takeaway arrays — see Known Limitations.
- **Webhook returns an error-branch response for content that looks long enough:** confirm the request body actually sends the field as `content` (not `text` or another key) and that it's valid JSON — the IF node checks `$json.body.content.length` directly, and a missing/undefined `content` field will error rather than gracefully failing.
- **AI-generated fields come back empty:** check the corresponding Parse output(N) Code node — empty/default values mean the model's response failed `JSON.parse`, which usually indicates the model added prose outside the JSON object despite the "return ONLY valid JSON" instruction.
- **Optional fields (topic/tone/audience) show as blank/undefined in generated content:** there is no default-value fallback for these fields anywhere in the workflow; if the webhook caller omits them, prompts will contain empty strings.

## Known Limitations

- **Single-element array fields:** `Set - Merge All Outputs` assigns `twitter_thread[0]` and `key_takeaways_list[0]` using n8n's array-index dot notation, but only ever sets index `0`. This produces one-element arrays holding pre-joined text rather than true multi-item arrays. Any future consumer expecting `twitter_thread.length` to equal the number of tweets, or `key_takeaways_list` to be iterable per-item, will get incorrect results (this exact issue already affects the Telegram message's `{{ ...twitter_thread.length }}` reference, which will always report `1`).
- **No retry logic** on any external API call (OpenRouter ×6, Google Sheets, Telegram) — see Retry Strategy.
- **No workflow-level or per-node error handling** beyond the content-length IF gate and the Code nodes' internal JSON try/catch — a single failed OpenRouter call anywhere in the 5-branch fan-out will fail the whole execution.
- **No webhook authentication** — the trigger endpoint is open to any caller.
- **Hardcoded destination values:** Telegram `chatId` and the Google Sheets `documentId`/`sheetName` are hardcoded in node parameters, so retargeting to a different chat or spreadsheet requires editing the workflow directly.
- **Two sheet column headers contain typos** as configured (`Orginal content`, `Key Takeways`) — functional but worth correcting for a production-facing document/report built from this sheet.
- **Free-tier model dependency:** all six AI Agent nodes rely on a single OpenRouter free-tier model (`nvidia/nemotron-nano-9b-v2:free`); availability, rate limits, and output quality/consistency for this specific free model are outside this workflow's control.
- **Optional input fields have no defaults:** `topic`, `tone`, and `audience` are used directly in prompts with no fallback value if omitted from the webhook payload.

## Assumptions

- Assumed `body.content` is always a plain string (the `content_length` and IF-node expressions call `.length` directly on it with no type guard).
- Assumed the n8n Set node's default behavior (options left empty) retains only explicitly assigned fields but preserves nested object structure created via dot-notation names (e.g. `body.content` → `{ body: { content: ... } }`), which is what allows the IF node to still read `$json.body.content.length` after Set - Normalize Input runs.
- Assumed the workflow's timezone context follows the n8n instance default, since no `settings.timezone` override is present in the export.
- Assumed the Telegram success message is intended to render as Markdown (based on `*asterisk*` bold syntax in the text), though no explicit `parseMode` field was found under `additionalFields` in the export — flagged for the owner to confirm this renders correctly in production.
- Assumed "AI Agent - Key Takeaways Extractor"'s system message follows the same "JSON only" convention visible in its sibling agents, based on consistent prompt structure, since the exact system message text should be verified directly against the JSON if precision is required.

## Version History / Change Log

_[To be provided by workflow owner]_

## References

_[To be provided by workflow owner]_

## Maintenance Notes

_[To be provided by workflow owner]_
