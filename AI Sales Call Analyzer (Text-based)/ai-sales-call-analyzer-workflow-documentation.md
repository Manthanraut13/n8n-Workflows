# AI Sales Call Analyzer (Text-based) — Workflow Documentation

| Field | Value |
|---|---|
| Workflow Name | AI Sales Call Analyzer (Text-based) |
| Workflow ID | `986IZttP6ryfEWl4` |
| Version | Not tracked as a semantic version in the export. Internal n8n revision hash (`versionId`): `de1848f2-8590-4c79-aec1-dfb3d27da924`. Suggest `v1.0 (initial documentation)`. |
| Owner | _[To be provided by workflow owner]_ |
| Last Updated | Not present in export (`updatedAt` not included in this JSON). Document generated: 2026-07-10. |
| Active | `false` (workflow is currently deactivated in the source instance) |
| Execution Order | `v1` (connection-based) |

## Purpose

This workflow ingests sales call transcripts or chat logs via webhook, uses an AI Agent (via OpenRouter) to extract structured sales intelligence (intent, buying signals, objections, risks, sentiment, deal score, recommended action, follow-ups), stores every analysis in a Google Sheet, and sends a Telegram alert for high-value or at-risk deals. Four additional scheduled sub-flows roll this raw data up into objection trend tracking, rep-wise performance analysis, a weekly at-risk deal report, and a daily summary dashboard.

## Business Use Case

Automates the manual work of reviewing sales call transcripts, so sales managers get consistent, structured intelligence (deal score, risks, objections) on every call without listening to recordings or reading notes by hand, and are proactively alerted to deals that are either close to closing or at risk. _(Inferred from workflow structure and sticky-note group titles; not explicitly stated by the workflow owner — confirm/replace if there is an official use-case description.)_

## Workflow Summary

An external system (e.g. a call-recording tool, CRM, or manual form) POSTs a transcript to a webhook. The text is cleaned and enriched with metadata (deal name, company, rep, call date, a generated analysis ID), then routed through a length check. If the transcript is short enough, it's sent to an AI Agent backed by an OpenRouter-hosted free model, which returns a structured JSON sales analysis. That JSON is parsed, written as a new row to the "Call Intelligence" Google Sheet, and — if the deal score is ≥75 or any risk was detected — a Telegram alert is sent to the sales team. Independently of the webhook path, four scheduled jobs periodically re-read the Call Intelligence sheet to produce: (1) a top-10 objection frequency table, (2) per-rep performance stats broadcast to Telegram weekly, (3) a weekly at-risk deals report emailed via Gmail, and (4) a daily rollup of call volume/score/risk statistics.

## Workflow Diagram

A screenshot of the canvas was provided (`AI_Sales_Call_Analyzer__Text-based_.png`) and matches the JSON structure. The canvas is organized into five labeled frames (via Sticky Notes), reflecting five largely independent flows sharing one Google Sheet as common storage:

```
┌─────────────────────────── MAIN AUTOMATION ───────────────────────────┐
│ Webhook → Set (Clean & Normalize Input) → If ──(false)──► AI Agent    │
│                                             │(true)         (Sales     │
│                                             ▼ [no output]   Intelligence│
│                                                              Extraction)│
│                          OpenRouter Chat Model ──(ai_languageModel)──┘ │
│  AI Agent → PARSE OUTPUT → STORE DATA → If1 ──(true)──► SEND MESSAGE  │
│                                          │(false) [no output]          │
└─────────────────────────────────────────────────────────────────────┘

┌── Objection Trend Tracking ──┐   ┌── Daily Summary Dashboard ──┐
│ Schedule Trigger (10:00 daily)│   │ Schedule Trigger3 (23:59)    │
│ → Get row(s) in sheet         │   │ → Get row(s) in sheet3       │
│ → Code in JavaScript1         │   │ → Code in JavaScript6        │
│ → Clear sheet                 │   │ → Append row in sheet4       │
│ → Append row in sheet1        │   └──────────────────────────────┘
└────────────────────────────────┘

┌────────────── Rep-Wise Performance Analysis ──────────────┐
│ Schedule Trigger1 (weekly, Mon 08:00)                       │
│ → Get row(s) in sheet1 → Code in JavaScript2 → Clear sheet1 │
│ → Append row in sheet2 → Code in JavaScript3                │
│ → Send a text message1 (Telegram)                            │
└───────────────────────────────────────────────────────────┘

┌──────────────── Weekly Deal Risk Summary ───────────────────┐
│ Schedule Trigger2 (weekly, Mon 21:59)                         │
│ → Get row(s) in sheet2 → Code in JavaScript4 → If2            │
│    (true)→ Append row in sheet3 → Code in JavaScript5         │
│    → Send a message (Gmail)                                    │
│    (false)→ [no output]                                        │
└────────────────────────────────────────────────────────────┘
```

## Trigger Details

This workflow has **five independent trigger nodes**, each starting a separate execution path that shares the same Google Sheet document for data.

| Trigger Node | Type | Fires |
|---|---|---|
| Webhook | `n8n-nodes-base.webhook` | On `POST` request to path `/sales-call-analysis`. Response mode: `lastNode` (caller receives whatever the last-executed node in that run outputs). No authentication configured on the webhook itself. |
| Schedule Trigger | `n8n-nodes-base.scheduleTrigger` | Daily at 10:00 (server/instance timezone — no explicit `timezone` override in `settings`). Feeds Objection Trend Tracking. |
| Schedule Trigger1 | `n8n-nodes-base.scheduleTrigger` | Weekly, Monday at 08:00. Feeds Rep-Wise Performance Analysis. |
| Schedule Trigger2 | `n8n-nodes-base.scheduleTrigger` | Weekly, Monday at 21:59. Feeds Weekly Deal Risk Summary. |
| Schedule Trigger3 | `n8n-nodes-base.scheduleTrigger` | Daily at 23:59. Feeds Daily Summary Dashboard. |

## Execution Flow

### 1. Main Automation (webhook-triggered)
1. **Webhook** receives a `POST /sales-call-analysis` request containing the transcript and optional metadata.
2. **Set (Clean & Normalize Input)** trims/collapses whitespace in the transcript and defaults missing metadata fields; generates a unique `analysis_id`.
3. **If** checks whether `transcript_clean.length > 8000`.
   - **True** (long transcript): no downstream connection exists — the execution ends here and the transcript is **not analyzed**. See Known Limitations.
   - **False** (transcript ≤ 8000 chars): proceeds to the AI Agent.
4. **AI Agent (Sales Intelligence Extraction)** sends the transcript and deal context to the LLM (via the **OpenRouter Chat Model** sub-node) with a system prompt instructing it to return sales intelligence as strict JSON.
5. **PARSE OUTPUT** (Code node) strips any Markdown code-fence wrapper from the model's raw text, parses the JSON, and flattens arrays (buying signals, objections, risks, follow-up points) into semicolon-delimited strings. Throws an error if parsing fails.
6. **STORE DATA** (Google Sheets – Append) writes one row to the spreadsheet (sheet reference `gid=0`, cached name `Sheet1`) with the parsed analysis plus the original metadata (pulled back from the **Set (Clean & Normalize Input)** node by name reference).
7. **If1** checks whether `Deal Score ≥ 75` OR the `Risks` string is non-empty.
   - **True**: proceeds to **SEND MESSAGE** (Telegram), which posts a formatted alert.
   - **False**: no downstream connection — execution ends silently, no notification sent.

### 2. Objection Trend Tracking (daily, 10:00)
1. **Schedule Trigger** fires.
2. **Get row(s) in sheet** reads all rows from the "Call Intelligence" tab (`gid=0`).
3. **Code in JavaScript1** aggregates the `Objections` column across all rows, counts frequency per distinct objection, and keeps the top 10.
4. **Clear sheet** clears range `A2:C100` on the "Objection Trends" tab.
5. **Append row in sheet1** writes the top-10 objection/count/timestamp rows back to "Objection Trends".

### 3. Rep-Wise Performance Analysis (weekly, Monday 08:00)
1. **Schedule Trigger1** fires.
2. **Get row(s) in sheet1** reads all rows from "Call Intelligence".
3. **Code in JavaScript2** groups rows by `Sales Rep` and computes total calls, average deal score, high-intent call count/percentage, total buying signals, total objections, and positive-sentiment percentage per rep; sorts reps by average deal score descending.
4. **Clear sheet1** clears range `A2:I100` on the "Rep Performance" tab.
5. **Append row in sheet2** writes the per-rep stats to "Rep Performance".
6. **Code in JavaScript3** re-sorts the appended rep records and builds a formatted Telegram message highlighting the top performer and a full team ranking.
7. **Send a text message1** (Telegram) posts that message to the configured chat.

### 4. Weekly Deal Risk Summary (weekly, Monday 21:59)
1. **Schedule Trigger2** fires.
2. **Get row(s) in sheet2** reads all rows from "Call Intelligence".
3. **Code in JavaScript4** filters to calls from the last 7 days that have a non-empty, non-"none identified" `Risks` value, computes days-since-call, and classifies each into `Active - Monitor` / `Active - Requires Attention` / `At Risk - Urgent Action` based on deal score thresholds (≥70 / 50–69 / <50).
4. **If2** checks whether any risky deals were found (`$input.all().length > 0`).
   - **True**: proceeds to **Append row in sheet3**.
   - **False**: no downstream connection — no report is generated or sent that week.
5. **Append row in sheet3** writes the at-risk deals to the "Weekly Risk Report" tab.
6. **Code in JavaScript5** classifies deals into "urgent" vs. "attention" buckets and formats a plain-text email subject/body summarizing both groups.
7. **Send a message** (Gmail) emails the report to a fixed recipient address.

### 5. Daily Summary Dashboard (daily, 23:59)
1. **Schedule Trigger3** fires.
2. **Get row(s) in sheet3** reads all rows from "Call Intelligence".
3. **Code in JavaScript6** determines the latest call date present in the data (not strictly "today", to tolerate timing/timezone drift), filters calls to that date, and computes: call count, average deal score, high-intent count, at-risk count, most frequent objection, and best-performing rep for that day.
4. **Append row in sheet4** writes the daily stats to the "Daily Summary Stats" tab.

## Node List

| # | Node Name | Type | Purpose | Disabled |
|---|---|---|---|---|
| 1 | Webhook | Webhook | Entry point; receives transcript + metadata via POST | No |
| 2 | Set (Clean & Normalize Input) | Set | Cleans transcript text, defaults metadata, generates `analysis_id` | No |
| 3 | If | If | Routes based on transcript length (>8000 chars) | No |
| 4 | OpenRouter Chat Model | LangChain LM Chat (OpenRouter) | Supplies the LLM used by the AI Agent | No |
| 5 | AI Agent (Sales Intelligence Extraction) | LangChain Agent | Extracts structured sales intelligence via LLM | No |
| 6 | PARSE OUTPUT | Code (JavaScript) | Strips code fences, parses AI JSON, flattens arrays | No |
| 7 | STORE DATA | Google Sheets (Append) | Writes analysis row to "Call Intelligence" sheet | No |
| 8 | If1 | If | Checks deal score ≥75 OR risks present | No |
| 9 | SEND MESSAGE | Telegram | Sends high-value/at-risk deal alert | No |
| 10 | Schedule Trigger | Schedule Trigger | Daily 10:00 — starts objection trend job | No |
| 11 | Get row(s) in sheet | Google Sheets (Read) | Reads all Call Intelligence rows | No |
| 12 | Code in JavaScript1 | Code (JavaScript) | Aggregates top-10 objections | No |
| 13 | Clear sheet | Google Sheets (Clear) | Clears "Objection Trends" tab (A2:C100) | No |
| 14 | Append row in sheet1 | Google Sheets (Append) | Writes objection trend rows | No |
| 15 | Schedule Trigger1 | Schedule Trigger | Weekly Mon 08:00 — starts rep performance job | No |
| 16 | Get row(s) in sheet1 | Google Sheets (Read) | Reads all Call Intelligence rows | No |
| 17 | Code in JavaScript2 | Code (JavaScript) | Computes per-rep performance stats | No |
| 18 | Clear sheet1 | Google Sheets (Clear) | Clears "Rep Performance" tab (A2:I100) | No |
| 19 | Append row in sheet2 | Google Sheets (Append) | Writes rep performance rows | No |
| 20 | Code in JavaScript3 | Code (JavaScript) | Formats weekly performance Telegram message | No |
| 21 | Send a text message1 | Telegram | Sends weekly performance report | No |
| 22 | Schedule Trigger2 | Schedule Trigger | Weekly Mon 21:59 — starts risk summary job | No |
| 23 | Get row(s) in sheet2 | Google Sheets (Read) | Reads all Call Intelligence rows | No |
| 24 | Code in JavaScript4 | Code (JavaScript) | Filters/classifies at-risk deals (last 7 days) | No |
| 25 | If2 | If | Checks whether any at-risk deals were found | No |
| 26 | Append row in sheet3 | Google Sheets (Append) | Writes at-risk deal rows to "Weekly Risk Report" | No |
| 27 | Code in JavaScript5 | Code (JavaScript) | Formats weekly risk email subject/body | No |
| 28 | Send a message | Gmail | Emails weekly at-risk deals report | No |
| 29 | Schedule Trigger3 | Schedule Trigger | Daily 23:59 — starts daily summary job | No |
| 30 | Get row(s) in sheet3 | Google Sheets (Read) | Reads all Call Intelligence rows | No |
| 31 | Code in JavaScript6 | Code (JavaScript) | Computes daily summary stats | No |
| 32 | Append row in sheet4 | Google Sheets (Append) | Writes daily summary row to "Daily Summary Stats" | No |
| 33 | Sticky Note | Sticky Note | Canvas label: "MAIN AUTOMATION" | No |
| 34 | Sticky Note1 | Sticky Note | Canvas label: "Objection Trend Tracking" | No |
| 35 | Sticky Note2 | Sticky Note | Canvas label: "Daily Summary Dashboard" | No |
| 36 | Sticky Note3 | Sticky Note | Canvas label: "Rep-Wise Performance Analysis" | No |
| 37 | Sticky Note4 | Sticky Note | Canvas label: "Weekly Deal Risk Summary" | No |

## Node-by-Node Description

### Webhook
HTTP entry point, method `POST`, path `/sales-call-analysis`, response mode `lastNode` (the HTTP response body is whatever the final node in the triggered execution outputs, not necessarily an explicit acknowledgment). Expects a JSON body with (at minimum) `transcript`, and optionally `deal_name`, `company_name`, `sales_rep`, `call_date`. No webhook authentication is configured.

### Set (Clean & Normalize Input)
An Edit Fields (Set) node that produces six fields: `transcript_clean` (whitespace-trimmed/collapsed transcript), `deal_name`, `company_name`, `sales_rep` (each defaulted via `||` if missing from the body), `call_date` (defaults to current server time if not supplied), and `analysis_id` (a Unix-timestamp + random string composite, generated fresh per execution). Downstream nodes reference this node by name (`$('Set (Clean & Normalize Input)')`) to recover the original metadata after the AI call.

### If
Single condition: `transcript_clean.length > 8000` (number, greater-than, strict type validation). See Branching Logic and Known Limitations — the "true" (long transcript) output has no downstream connection.

### OpenRouter Chat Model
LangChain Chat Model sub-node feeding the AI Agent's `ai_languageModel` input. Model: `nvidia/nemotron-nano-12b-v2-vl:free` (OpenRouter free-tier model). Uses the `openRouterApi` credential named "n8n-regular". No temperature/other generation options are explicitly overridden (`options: {}`).

### AI Agent (Sales Intelligence Extraction)
LangChain Agent node (`@n8n/n8n-nodes-langchain.agent`, typeVersion 3). Prompt type: `define` (explicit prompt text, not "auto" from chat input). See AI/LLM Integration below for the full prompt text. Output lands on the item's `output` field as a raw text string (expected to be JSON, per the system prompt).

### PARSE OUTPUT
Code node (JavaScript) that:
- Reads `$input.item.json.output`.
- Strips a leading ```` ```json ```` fence and any ```` ``` ```` fences via regex.
- `JSON.parse()`s the cleaned string; **throws an error** ("Failed to parse AI JSON output") if parsing fails — there is no fallback/retry.
- Returns a flattened object: `call_summary`, `intent_level`, `sentiment`, `deal_score`, `recommended_action` as scalars; `buying_signals`, `objections`, `risks`, `follow_up_points` joined into `; `-delimited strings (empty string if the source wasn't an array).

### STORE DATA
Google Sheets Append to document `1fqZX62Ciss-T7mOP3qIsd9OyeWoNfwmaPQL-pffaGM4`, sheet reference `gid=0` (cached display name "Sheet1" in this node's parameters, though the same `gid=0` tab is labeled "Call Intelligence" in other nodes — see Assumptions). Maps 15 columns: `Call Summary`, `Intent Level`, `Buying Signals`, `Objections`, `Risks`, `Sentiment`, `Deal Score`, `Recommended Action`, `Follow-up Points` (from PARSE OUTPUT's output) plus `Analysis ID`, `Call Date`, `Deal Name`, `Company`, `Sales Rep`, `Raw Transcript` (pulled back from the **Set (Clean & Normalize Input)** node by name).

### If1
Combinator `OR` across two conditions: `Deal Score >= 75` (number, gte) and `Risks.length > 0` (number, gt — note this is JavaScript string `.length`, not an array length; see Known Limitations). True output connects to SEND MESSAGE; false output has no connection (silent no-op).

### SEND MESSAGE
Telegram node, hardcoded `chatId: "7350267365"`. Message is a formatted multi-line alert including deal name, score, company, rep, intent, sentiment, buying signals, objections, risks, recommended action, follow-up points, and the analysis ID. Uses the "Telegram account for AI Journal" credential.

### Schedule Trigger / Schedule Trigger1 / Schedule Trigger2 / Schedule Trigger3
Four independent Cron-style triggers — see Scheduling Details below for exact timing.

### Get row(s) in sheet / Get row(s) in sheet1 / Get row(s) in sheet2 / Get row(s) in sheet3
Four near-identical Google Sheets "Read" nodes, each reading the full "Call Intelligence" tab (`gid=0`) with no filters — full-table scans on every scheduled run. All use the "Google Sheets account" credential. (`Get row(s) in sheet` has `alwaysOutputData: false` explicitly set; the others rely on default behavior.)

### Code in JavaScript1 (Objection aggregation)
Iterates all rows, splits each row's `Objections` string on `;`, counts occurrences of each distinct objection (case-sensitive except for filtering out the literal string "none raised"), sorts descending, and keeps the top 10. Emits one item per objection with `objection`, `count`, `last_updated`.

### Clear sheet
Google Sheets "Clear" operation on the "Objection Trends" tab, fixed range `A2:C100` (i.e. up to 99 data rows retained; anything beyond that range is not cleared and could accumulate stale data if trends ever exceed 10 rows carried over incorrectly — in practice only 10 rows are written back each run, well within range).

### Append row in sheet1
Google Sheets Append writing `Objection`, `Count`, `Last Updated` to "Objection Trends".

### Code in JavaScript2 (Rep stats)
Groups all Call Intelligence rows by `Sales Rep` (default `'Unassigned'` if blank), accumulating: total calls, per-call deal scores, high-intent count, positive-sentiment count, and total buying-signal/objection counts (via semicolon-split length). Computes average deal score, high-intent %, and positive-sentiment % per rep, then returns one item per rep sorted by average deal score descending.

### Clear sheet1
Google Sheets "Clear" on "Rep Performance" tab, range `A2:I100`.

### Append row in sheet2
Google Sheets Append writing 9 columns (`Sales Rep`, `Total Calls`, `Avg Deal Score`, `High Intent Calls`, `High Intent %`, `Total Buying Signals`, `Total Objections`, `Positive Sentiment %`, `Last Updated`) to "Rep Performance".

### Code in JavaScript3 (Performance message formatter)
Re-sorts the just-appended rep items by `Avg Deal Score` descending, identifies the top performer, and builds a single formatted text block (`message` field) summarizing the top performer and a numbered team ranking, for Telegram.

### Send a text message1
Telegram node, same hardcoded `chatId: "7350267365"` as SEND MESSAGE, posting the `message` field built by Code in JavaScript3. Uses the "Telegram account for AI Journal" credential.

### Code in JavaScript4 (At-risk filter)
Filters Call Intelligence rows to those with a non-empty `Risks` value (excluding the literal "none identified") **and** a `Call Date` within the last 7 days (`Date` comparison against `now - 7 days`). For each match, computes `days_since_call` and a `status` classification: `Active - Monitor` (score ≥70), `Active - Requires Attention` (50–69), or `At Risk - Urgent Action` (<50). Sorts ascending by score (worst deals first).

### If2
Single condition: `$input.all().length > 0` — i.e., "did the filter step find any at-risk deals this week?" True → continue to reporting; false → the branch ends with no report generated and no email sent.

### Append row in sheet3
Google Sheets Append writing 8 columns (`Report Date`, `Deal Name`, `Company`, `Sales Rep`, `Deal Score`, `Risks`, `Days Since Call`, `Status`) to "Weekly Risk Report".

### Code in JavaScript5 (Risk email formatter)
Buckets the appended items into "urgent" (status contains "urgent") vs. "attention" (status contains "attention" or "monitor") groups, and builds an email `subject` and plain-text `body` listing both groups with deal name, company, rep, score, risks, and days since call.

### Send a message (Gmail)
Sends the formatted weekly risk email. Fixed recipient: `storysnippents@gmail.com`. `emailType: text`. Subject and body come from Code in JavaScript5's output. Uses the "Gmail account" OAuth2 credential.

### Get row(s) in sheet3 (Daily Summary)
Same pattern as the other three "read all Call Intelligence rows" nodes, feeding the daily summary job.

### Code in JavaScript6 (Daily stats)
Determines the "reporting date" as the **latest** `Call Date` present in the sheet (not the literal current date), to be resilient to timezone/timing gaps between when a call was logged and when this job runs. Filters rows to that date, then computes: call count, average deal score, high-intent count, at-risk count (any non-trivial `Risks` value), the most frequent single objection that day, and the sales rep with the highest average score that day.

### Append row in sheet4
Google Sheets Append writing 8 columns (`Date`, `Total Calls Analyzed`, `Avg Deal Score`, `High Intent Count`, `At Risk Deals`, `Top Objection`, `Best Performing Rep`, `Generated At`) to "Daily Summary Stats".

### Sticky Notes (Sticky Note, Sticky Note1–4)
Purely visual canvas group labels — "MAIN AUTOMATION", "Objection Trend Tracking", "Daily Summary Dashboard", "Rep-Wise Performance Analysis", "Weekly Deal Risk Summary". No functional effect; used here to structure this document's grouping.

## Node Configuration

### Set (Clean & Normalize Input) — field expressions
```
transcript_clean = {{ $json.body.transcript.trim().replace(/\s+/g, ' ') }}
deal_name        = {{ $json.body.deal_name || 'Unknown Deal' }}
company_name     = {{ $json.body.company_name || 'Unknown Company' }}
sales_rep        = {{ $json.body.sales_rep || 'Unassigned' }}
call_date        = {{ $json.body.call_date || $now.format('yyyy-MM-dd HH:mm:ss') }}
analysis_id      = {{ $now.toUnixInteger() }}_{{ Math.random().toString(36).substr(2, 9) }}
```

### If — length check
`transcript_clean.length` **greater than** `8000` (number comparison, strict type validation, case-sensitive option set but irrelevant to a numeric comparison).

### If1 — notification trigger
`Deal Score` **≥** `75` **OR** `Risks.length` **>** `0` (combinator: `or`).

### If2 — risk report gate
`$input.all().length` **>** `0`.

### Google Sheets document/tabs (all Google Sheets nodes point at the same spreadsheet)
- Document: `1fqZX62Ciss-T7mOP3qIsd9OyeWoNfwmaPQL-pffaGM4` ("AI Sales Call Analyzer (Text-based)")
- Tabs used: `gid=0` (Call Intelligence / cached as "Sheet1" in the STORE DATA node), `gid=1570396302` (Objection Trends), `gid=685151071` (Rep Performance), `gid=1169302920` (Weekly Risk Report), `gid=355210388` (Daily Summary Stats)

## Branching Logic

| Node | Condition | True branch | False branch |
|---|---|---|---|
| If | `transcript_clean.length > 8000` | *(no connection — execution ends)* | AI Agent (Sales Intelligence Extraction) |
| If1 | `Deal Score >= 75 OR Risks.length > 0` | SEND MESSAGE (Telegram alert) | *(no connection — execution ends silently)* |
| If2 | `count of at-risk deals found > 0` | Append row in sheet3 → ... → Gmail | *(no connection — no report/email that week)* |

Note the **inverted intuition** on the `If` node: the "true" output (long transcript) is the one with *no* downstream path, while the "false" output (transcript within limit) is what continues to analysis. In effect, transcripts over 8000 characters are silently dropped rather than chunked or handled specially.

## Variables & Expressions

Notable expressions beyond the field mappings already shown:
- `{{ $('Set (Clean & Normalize Input)').item.json.analysis_id }}` and siblings in **STORE DATA** — pulls original request metadata back into scope after the AI Agent/Parse steps, using n8n's named-node reference syntax rather than relying on data passing straight through (since the AI Agent node's output replaces `$json`).
- `{{ $now.toUnixInteger() }}_{{ Math.random().toString(36).substr(2, 9) }}` — generates a semi-unique `analysis_id` (timestamp + random suffix); not guaranteed globally unique but collision risk is very low for this use case.
- Code in JavaScript6's `latestDate` logic effectively substitutes for `{{ $today }}`, deliberately trusting the data over wall-clock time.

## Credentials Used

| Node(s) | Credential Name | Type | Service |
|---|---|---|---|
| OpenRouter Chat Model | n8n-regular | `openRouterApi` | OpenRouter (LLM API) |
| Get row(s) in sheet, Clear sheet, Append row in sheet1, Get row(s) in sheet1, Clear sheet1, Append row in sheet2, Get row(s) in sheet2, Append row in sheet3, Get row(s) in sheet3, Append row in sheet4, STORE DATA | Google Sheets account | `googleSheetsOAuth2Api` | Google Sheets |
| SEND MESSAGE, Send a text message1 | Telegram account for AI Journal | `telegramApi` | Telegram Bot API |
| Send a message | Gmail account | `gmailOAuth2` | Gmail |

No secret values are present in the export (n8n never exports raw credential secrets — only name/ID references, as shown above).

## Input Data

**Webhook (main flow):** JSON body expected at `POST /sales-call-analysis`:
```json
{
  "transcript": "string (required — the call/chat text)",
  "deal_name": "string (optional)",
  "company_name": "string (optional)",
  "sales_rep": "string (optional)",
  "call_date": "string (optional, e.g. 'YYYY-MM-DD HH:mm:ss')"
}
```
**Scheduled flows:** no external input; each reads the full "Call Intelligence" Google Sheet tab as its data source.

## Output Data

- **STORE DATA**: one new row per analyzed call appended to the "Call Intelligence" tab (15 columns, see Node-by-Node Description).
- **SEND MESSAGE / Send a text message1**: Telegram messages posted to chat ID `7350267365`.
- **Append row in sheet1**: up to 10 rows (objection, count, timestamp) appended to "Objection Trends" (after the tab is cleared each run).
- **Append row in sheet2**: one row per active sales rep appended to "Rep Performance" (after the tab is cleared each run).
- **Append row in sheet3**: one row per at-risk deal found that week, appended to "Weekly Risk Report" (not cleared — accumulates over time).
- **Send a message (Gmail)**: one weekly plain-text email to `storysnippents@gmail.com`.
- **Append row in sheet4**: one row per day appended to "Daily Summary Stats" (not cleared — accumulates over time).

## Data Transformation

The workflow performs several notable reshaping steps:
- Free-text transcript → structured JSON (via the AI Agent) → flattened spreadsheet row (arrays joined with `; `).
- Row-level Call Intelligence data → aggregated objection frequency table (Code in JavaScript1).
- Row-level Call Intelligence data → per-rep aggregate statistics (Code in JavaScript2).
- Row-level Call Intelligence data → filtered/classified subset of at-risk deals (Code in JavaScript4).
- Row-level Call Intelligence data → single-day aggregate snapshot (Code in JavaScript6).
- Two sheets ("Objection Trends", "Rep Performance") are **replaced** on each run (clear + append), while "Weekly Risk Report", "Daily Summary Stats", and "Call Intelligence" itself are **append-only logs** that grow indefinitely.

## AI/LLM Integration

- **Provider**: OpenRouter, via the `@n8n/n8n-nodes-langchain.lmChatOpenRouter` node ("OpenRouter Chat Model"), connected to the AI Agent's `ai_languageModel` input.
- **Model**: `nvidia/nemotron-nano-12b-v2-vl:free` (OpenRouter free-tier model).
- **Agent node**: `@n8n/n8n-nodes-langchain.agent` (typeVersion 3), prompt type `define`.
- **User prompt template** (built from the Set node's output):
  ```
  TRANSCRIPT:
  {{ $json.transcript_clean }}

  DEAL CONTEXT:
  - Deal Name: {{ $json.deal_name }}
  - Company: {{ $json.company_name }}
  - Sales Rep: {{ $json.sales_rep }}
  ```
- **System message**:
  ```
  You are an expert sales intelligence analyst. Analyze the following sales conversation transcript and extract structured insights.

  TASK:
  Analyze this sales conversation and return ONLY a valid JSON object with the following structure:

  {
    "call_summary": "Brief 2-3 sentence summary of the call",
    "intent_level": "Low | Medium | High",
    "buying_signals": ["signal1", "signal2"],
    "objections": ["objection1", "objection2"],
    "risks": ["risk1", "risk2"],
    "sentiment": "Positive | Neutral | Negative",
    "deal_score": 75,
    "recommended_action": "Clear next step recommendation",
    "follow_up_points": ["point1", "point2", "point3"]
  }

  ANALYSIS GUIDELINES:
  - Intent Level: Evaluate readiness to buy (Low=research, Medium=evaluating, High=ready to purchase)
  - Buying Signals: Look for budget mentions, timeline urgency, decision-maker involvement, competitor comparisons
  - Objections: Price concerns, feature gaps, timing issues, competitor preferences
  - Risks: Authority unclear, budget constraints, long decision timeline, competitor advantage
  - Sentiment: Overall tone (Positive=enthusiastic, Neutral=informational, Negative=frustrated)
  - Deal Score: 0-100 likelihood of closing (consider intent, objections, timeline, authority)
  - Recommended Action: Specific next step (demo, proposal, follow-up call, trial, etc.)
  - Follow-up Points: 3-5 key topics to address in next interaction

  Return ONLY valid JSON. No markdown, no code blocks, no explanations.
  ```
- **Downstream consumption**: The agent's raw text output (`$json.output`) is passed to **PARSE OUTPUT**, which strips any Markdown code fence and calls `JSON.parse()`. There is no schema/structured-output enforcement at the API level — correctness depends entirely on the model following the system prompt. No memory/conversation history and no tools are attached to this agent (single-shot extraction only).

## Error Handling

- No node in this workflow has `continueOnFail`/`onError` set, and `settings` contains no `errorWorkflow` reference — an unhandled exception anywhere (most notably a `JSON.parse` failure in **PARSE OUTPUT** if the model doesn't return valid JSON) will fail the entire execution with no automatic recovery or alerting.
- **PARSE OUTPUT** explicitly `throw`s a custom error ("Failed to parse AI JSON output") on malformed AI responses rather than falling back to a default/partial record.
- The `If` and `If1`/`If2` "empty" branches are not error paths — they are intentional silent no-ops (long transcripts are dropped; low-priority deals don't get a Telegram alert; weeks with no risky deals don't get an email).

## Retry Strategy

No `retryOnFail`, `maxTries`, or `waitBetweenTries` settings are configured on any node (including the Google Sheets, Telegram, Gmail, or OpenRouter calls). A transient API failure (rate limit, network blip) will fail the run outright with no automatic retry.

## Failure Notifications

None configured. There is no node or error workflow dedicated to notifying anyone when an execution fails.

## Logging

No dedicated logging node exists. The only persistent record of activity is n8n's own execution log (visible in the n8n UI/API) plus the data written to the Google Sheet tabs themselves (which function as an application-level audit trail for successful runs only).

## Execution Order

`settings.executionOrder` is `v1` (connection-based execution order, the modern default). This mainly affects the interleaving of parallel branches within a single execution; since the five trigger flows in this workflow are independent (they don't share a single execution), this setting has limited practical impact here.

## Scheduling Details

| Trigger | Schedule | Plain English |
|---|---|---|
| Schedule Trigger | `triggerAtHour: 10` | Every day at 10:00 |
| Schedule Trigger1 | `field: weeks, triggerAtDay: [1], triggerAtHour: 8` | Every Monday at 08:00 |
| Schedule Trigger2 | `field: weeks, triggerAtDay: [1], triggerAtHour: 21, triggerAtMinute: 59` | Every Monday at 21:59 |
| Schedule Trigger3 | `triggerAtHour: 23, triggerAtMinute: 59` | Every day at 23:59 |

No `timezone` override is set in `settings`, so all schedules run in the n8n instance's configured default timezone. _[Confirm actual instance timezone with the workflow owner.]_

## Dependencies

- A single Google Sheets spreadsheet (`AI Sales Call Analyzer (Text-based)`, ID `1fqZX62...faGM4`) with five specifically-named/gid-referenced tabs must exist and be reachable: Call Intelligence (`gid=0`), Objection Trends (`gid=1570396302`), Rep Performance (`gid=685151071`), Weekly Risk Report (`gid=1169302920`), Daily Summary Stats (`gid=355210388`). If any tab is renamed/deleted, the corresponding node's cached `gid`/name reference will break.
- OpenRouter API must be reachable and the free-tier model `nvidia/nemotron-nano-12b-v2-vl:free` must remain available (OpenRouter periodically retires free models).
- A Telegram bot with access to chat ID `7350267365` must remain valid.
- The Gmail OAuth2 credential must remain authorized to send as the connected account.

## Security Considerations

- The **Webhook** trigger has no authentication configured (no header auth, no Basic Auth, no JWT check) — anyone who discovers the webhook URL can submit arbitrary transcripts for AI analysis and have them written to the company's Google Sheet and potentially trigger Telegram/Gmail notifications. Recommend adding header-based or JWT authentication if this endpoint is internet-reachable.
- The **Gmail recipient** (`storysnippents@gmail.com`) and both **Telegram chat IDs** are hardcoded in node parameters rather than pulled from environment variables/credentials — anyone with workflow edit access can see these; not a secret leak per se, but worth moving to environment variables for portability between environments (dev/staging/prod).
- Raw transcripts (potentially containing customer PII) are stored in plaintext in the "Call Intelligence" Google Sheet indefinitely with no redaction or retention policy.
- _[Further security review/sign-off to be provided by workflow owner.]_

## Performance Considerations

- All four scheduled read nodes (`Get row(s) in sheet` / `...1` / `...2` / `...3`) perform a **full read of the entire "Call Intelligence" tab** with no filtering, pagination, or date-range restriction. As the sheet grows, these reads — and the in-memory JavaScript aggregation that follows each — will get slower and eventually hit Google Sheets API or n8n memory limits.
- The 8,000-character transcript length check in the main flow has no actual chunking implementation behind it (see Known Limitations); long transcripts are simply dropped rather than costing extra processing time.
- No batching (`Split In Batches`) is used anywhere, so aggregation Code nodes process entire datasets in a single execution step.

## Execution Time

_[To be provided by workflow owner — requires execution data from the n8n instance.]_

## Resource Usage

_[To be provided by workflow owner — requires execution data from the n8n instance.]_

## Testing & Validation

_[To be provided by workflow owner.]_

## Troubleshooting

- **No Telegram alert received for a call that seems high-value**: check the **If1** condition — it requires `Deal Score >= 75` OR a non-empty `Risks` string. If the AI returned `risks: []`, the joined string is empty and that half of the OR is false; verify the AI's raw output in **PARSE OUTPUT**'s input.
- **Workflow execution fails immediately after the AI Agent runs**: almost certainly a JSON parsing failure in **PARSE OUTPUT** — the model likely returned prose, a malformed JSON object, or wrapped the JSON in unexpected formatting. Inspect the AI Agent's raw `output` field.
- **A submitted transcript never shows up in the sheet at all**: check its length — if `transcript_clean.length > 8000`, the **If** node's true branch has no downstream connection and the run ends silently with no row written and no error.
- **Weekly risk email doesn't arrive**: check **If2** — if no rows matched the "risky in the last 7 days" filter in **Code in JavaScript4**, the false branch is a dead end and no email is sent that week; this is expected behavior, not a failure.
- **Scheduled jobs return stale/incorrect Objection Trends or Rep Performance data**: these two sheets are fully cleared (`A2:C100` / `A2:I100`) and rewritten every run — if a run fails after the "Clear" step but before "Append", the sheet will be left empty until the next successful run.
- **Google Sheets node errors referencing a `gid` that no longer exists**: someone likely renamed or deleted a tab in the spreadsheet; re-select the correct sheet in the affected node(s).

## Known Limitations

- **Long transcripts (>8000 characters) are silently dropped.** The `If` node was clearly intended to route long transcripts to a chunking/batch-processing path (per the node's presence and naming conventions elsewhere in this workflow family), but no such path exists in this export — the "true" output has zero connections, so oversized transcripts simply vanish with no row written, no error, and no notification.
- **No retry or error-workflow handling anywhere** in the workflow (see Error Handling / Retry Strategy above); a single transient failure (e.g., OpenRouter rate limit, Google Sheets API hiccup) fails the whole execution.
- **`Risks.length` in If1 checks string length, not array length.** Because `Risks` is already flattened to a semicolon-joined string by the time If1 evaluates it, any non-empty risks string (even a single short risk) satisfies `> 0`; this is very likely the intended behavior (alert on any detected risk) but is worth confirming, since it's easy to misread as "count of risks."
- **No webhook authentication** — see Security Considerations.
- **Full-table reads with no pagination** on every scheduled job — see Performance Considerations; this will not scale indefinitely.
- **Hardcoded recipients**: Telegram `chatId` and the Gmail recipient address are hardcoded in node parameters, not parameterized via credentials or environment variables.
- **Weekly Risk Report and Daily Summary Stats tabs are append-only** and are never pruned/archived — they will grow without bound over the life of the workflow.
- **Single point of failure on the shared spreadsheet**: all five flows depend on one Google Sheets document; if it becomes unavailable or a tab is restructured, multiple flows break simultaneously.

## Assumptions

- Assumed the Google Sheets tab referenced as `gid=0` and cached as **"Sheet1"** in the **STORE DATA** node is the same physical tab that other nodes cache as **"Call Intelligence"** (also `gid=0`) — i.e., the tab was likely renamed from "Sheet1" to "Call Intelligence" after STORE DATA's reference was cached, and this display-name mismatch does not affect execution since `gid` (not the name) is the effective identifier.
- Assumed the workflow runs on the n8n instance's default/local timezone since no `settings.timezone` override is present.
- Assumed the two "no-op" IF branches (long transcripts, low-priority deals) are intentional design choices rather than incomplete wiring — flagged the long-transcript case specifically as a likely-unfinished feature under Known Limitations because a length-check node with no purpose otherwise would be unusual to include.
- Assumed `storysnippents@gmail.com` and Telegram chat ID `7350267365` are the intended, currently-correct recipients (cannot verify ownership from the export alone).
- Business Use Case (above) is an inference from workflow structure/sticky-note titles, not a stated requirement.

## Version History / Change Log

_[To be provided by workflow owner]_

## References

_[To be provided by workflow owner]_

## Maintenance Notes

_[To be provided by workflow owner]_
