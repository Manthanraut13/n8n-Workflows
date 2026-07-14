# AI Proposal Generator — Workflow Documentation

| Field | Value |
|---|---|
| Workflow Name | AI Proposal Generator |
| Workflow ID | `lE8loeSthHTSY8sG` |
| Version | Not tracked as a semantic version in the export. Internal n8n revision hash (`versionId`): `d691551a-a000-4583-920c-019b86098691` |
| Owner | _[To be provided by workflow owner]_ |
| Last Updated | Not present in the JSON export (no `updatedAt` field). Document generated: 2026-07-11 |
| Active State | `false` (workflow is currently deactivated in n8n) |

## Purpose

Automatically converts raw client requirements submitted via webhook into a structured, professional B2B proposal using an AI Agent, stores the result in Google Sheets, and notifies the workflow owner via Telegram. Invalid submissions are rejected early and trigger a separate Telegram error notification.

## Business Use Case

Automates first-draft proposal writing for a consulting/agency-style business, reducing manual proposal-drafting time and providing a centralized, trackable log of every proposal generated. _(Inferred from node naming and structure — not explicitly stated in the workflow; confirm with workflow owner.)_

## Workflow Summary

A client (or an internal form/integration) submits project requirements to a webhook endpoint. The data is normalized into consistent fields, validated for completeness, and — if valid — enriched with heuristic metadata (project complexity, project type, industry, urgency) via a Code node. This enriched context is compiled into a single prompt string and sent to an AI Agent (OpenRouter-hosted LLM) with a Structured Output Parser enforcing a fixed JSON schema. The AI's JSON response is parsed and validated by a second Code node; on success, the proposal fields are flattened into a Google Sheets row and appended to a tracking spreadsheet, followed by a Telegram success notification. If input validation fails, a separate branch sends a Telegram error notification instead.

## Workflow Diagram

See attached canvas screenshot (`AI_Proposal_Generator.png`). Shape: primarily **linear** (Webhook → Set → IF → Code → Set → AI Agent → Code → IF → Set → Google Sheets → Telegram), with **one conditional branch** off `IF (Validation)`: the false path drops to a lower row (`Set (Error Message)` → `Telegram (Send Error Notification)`). The AI Agent node has two auxiliary sub-connections feeding into it from below: `OpenRouter Chat Model` (language model) and `Structured Output Parser` (output parser) — both are LangChain-style auxiliary inputs, not part of the main data flow.

Reconstructed high-level flow:

```
Webhook (Input Trigger)
   │
Set (Normalize Input)
   │
IF (Validation) ──true──▶ Function (Enrich Context) ▶ Set (Prepare AI Input) ▶ AI Agent (Generate Proposal)
   │                                                                                │
   │                                                          ┌─────────────────────┴───────────────────┐
   │                                                   OpenRouter Chat Model              Structured Output Parser
   │                                                     (ai_languageModel)                  (ai_outputParser)
   │                                                                                │
   │                                                                Function (Parse AI Response)
   │                                                                                │
   │                                                                IF (Check Parse Success) ──true──▶ Set (Format for Sheets)
   │                                                                     │                                        │
   │                                                                  (false: not connected)          Google Sheets (Store Proposal)
   │                                                                                                                │
   │                                                                                                  Telegram (Send Notification)
   │
   └──false──▶ Set (Error Message) ▶ Telegram (Send Error Notification)
```

## Trigger Details

- **Type**: `Webhook` (`n8n-nodes-base.webhook`, typeVersion 2.1)
- **Node name**: `Webhook (Input Trigger)`
- **HTTP Method**: `POST`
- **Path**: `/generate-proposal`
- **Response Mode**: `lastNode` (the webhook HTTP response returns whatever the last-executed node in the triggered branch outputs)
- **Authentication**: None configured (no auth options set on the node) — see Security Considerations.
- **Expected payload**: JSON body with `client_name`, `company_name`, `project_description`, `budget_range` (optional), `timeline_expectations` (optional), `industry` (optional), `contact_email` (optional).

There is a single entry point; no other trigger nodes exist in this workflow.

## Execution Flow

1. **Webhook (Input Trigger)** receives a `POST` request at `/generate-proposal` with client requirement data in the body.
2. **Set (Normalize Input)** extracts fields from `$json.body`, applies default values for `budget_range` (`"To be discussed"`) and `timeline_expectations` (`"Flexible"`) if missing, and generates `proposal_id` (ISO timestamp) and `created_date` (formatted timestamp).
3. **IF (Validation)** checks that `client_name`, `company_name`, and `project_description` are all non-empty AND that `project_description` is longer than 20 characters (all conditions ANDed).
   - **True** → proceeds to step 4.
   - **False** → jumps to step 12 (error branch).
4. **Function (Enrich Context)** (Code node) analyzes `project_description` text to heuristically derive `complexity` (Low/Medium/High), `project_type`, `estimated_timeline_weeks`, `detected_industry`, and `urgency`, and attaches these as an `enriched_data` object.
5. **Set (Prepare AI Input)** builds a single `ai_context` string combining client, industry, project type, complexity, requirements, budget, and timeline into a block of text for the AI prompt.
6. **AI Agent (Generate Proposal)** sends the compiled context into a defined prompt (see AI/LLM Integration) to the connected **OpenRouter Chat Model**, using **Structured Output Parser** to enforce a specific JSON schema for the response.
7. **Function (Parse AI Response)** (Code node) extracts the AI's output, strips markdown code fences if present, parses it as JSON, and validates that all 9 required proposal fields are present. Sets `status: "success"` or `status: "parse_error"` accordingly.
8. **IF (Check Parse Success)** checks `status == "success"`.
   - **True** → proceeds to step 9.
   - **False** → **no downstream connection exists** for this branch (see Known Limitations) — execution ends silently for that item.
9. **Set (Format for Sheets)** flattens the parsed proposal plus earlier metadata (client, industry, complexity, etc.) into named fields matching the target spreadsheet columns.
10. **Google Sheets (Store Proposal)** appends a new row to the "AI Proposal Generator" spreadsheet (`Sheet1`) with the flattened data.
11. **Telegram (Send Notification)** sends a formatted success message to a fixed Telegram chat, summarizing client, company, proposal ID, executive summary, project type, and complexity.
12. **(Error branch, from step 3 False)** **Set (Error Message)** populates `error_type`, `error_details`, `client_name`, and `timestamp` fields (note: values are static placeholder strings, not the actual validation failure reason — see Known Limitations).
13. **Telegram (Send Error Notification)** sends a Telegram message using the *same template as the success message* (`Client_Name`, `Company_Name`, `Proposal_ID`, `Executive_Summary`, etc.) — see Known Limitations, as these fields don't actually exist on the error-branch payload.

## Node List

| # | Node Name | Type | Purpose | Disabled |
|---|---|---|---|---|
| 1 | Webhook (Input Trigger) | Webhook | Entry point; receives client requirements via POST | No |
| 2 | Set (Normalize Input) | Set (Edit Fields) | Extracts/normalizes input fields, sets defaults, generates IDs/timestamps | No |
| 3 | IF (Validation) | IF | Validates required fields are present and description length | No |
| 4 | Function (Enrich Context) | Code (JavaScript) | Heuristically derives complexity, project type, industry, urgency | No |
| 5 | Set (Prepare AI Input) | Set (Edit Fields, raw mode) | Builds combined `ai_context` string for the AI prompt | No |
| 6 | AI Agent (Generate Proposal) | LangChain Agent | Generates structured proposal JSON via LLM | No |
| 7 | OpenRouter Chat Model | LangChain LM Chat (OpenRouter) | Language model backing the AI Agent | No |
| 8 | Structured Output Parser | LangChain Output Parser (Structured) | Enforces JSON schema on AI Agent output | No |
| 9 | Function (Parse AI Response) | Code (JavaScript) | Parses/validates AI JSON response, sets status | No |
| 10 | IF (Check Parse Success) | IF | Checks whether parsing succeeded | No |
| 11 | Set (Format for Sheets) | Set (Edit Fields) | Flattens proposal + metadata into Sheets column format | No |
| 12 | Google Sheets (Store Proposal) | Google Sheets | Appends proposal row to tracking spreadsheet | No |
| 13 | Telegram (Send Notification) | Telegram | Sends success notification | No |
| 14 | Set (Error Message) | Set (Edit Fields) | Builds error metadata fields (static placeholders) | No |
| 15 | Telegram (Send Error Notification) | Telegram | Sends error notification (reuses success template) | No |

No Sticky Note nodes are present in this workflow.

## Node-by-Node Description

### Webhook (Input Trigger)
`n8n-nodes-base.webhook`, typeVersion 2.1. POST endpoint at path `/generate-proposal`, response mode `lastNode` (returns whatever the final node in the executed branch outputs as the HTTP response). No authentication configured. `webhookId`: `7f687bb4-b82a-4dab-b3ce-587e44e6739f`.

### Set (Normalize Input)
`n8n-nodes-base.set`, typeVersion 3.4. Maps 9 fields from `$json.body.*` into top-level fields: `client_name`, `company_name`, `project_description`, `budget_range` (default `"To be discussed"`), `timeline_expectations` (default `"Flexible"`), `industry` (no default), `contact_email` (no default), `proposal_id` (`{{ $now.toISO() }}`), `created_date` (`{{ $now.format('yyyy-MM-DD HH:mm:ss') }}`). Passes this normalized object downstream.

### IF (Validation)
`n8n-nodes-base.if`, typeVersion 2.2. Four ANDed conditions: `client_name` not empty, `company_name` not empty, `project_description` not empty, and `project_description.length > 20`. True branch → enrichment; false branch → error notification path.

### Function (Enrich Context)
`n8n-nodes-base.code`, typeVersion 2 (JavaScript, "Run Once for All Items" style — operates on `$input.all()[0].json`). Lowercases `project_description` and scans for keywords to classify:
- **Complexity**: High if description mentions "integration"/"api"/"custom"; Low if "simple"/"basic"; otherwise Medium.
- **Project type**: Automation Implementation, AI/ML Solution, Web Development, System Integration, or General Consulting, based on keyword matches.
- **Estimated timeline**: 12 weeks (High complexity), 4 weeks (Low), 8 weeks (Medium, default).
- **Detected industry**: matched against keyword lists for Healthcare, Finance, E-commerce, Education, Real Estate; falls back to the input's `industry` field or `"General"`.
- **Urgency**: `"High"` if `timeline_expectations` contains "urgent", else `"Normal"`.

Appends all of this as an `enriched_data` object alongside the original fields.

### Set (Prepare AI Input)
`n8n-nodes-base.set`, typeVersion 3.4, mode `raw`. Builds a single string field `ai_context` interpolating client name, company, detected industry, project type, complexity, the raw project description, budget range, timeline expectations, and estimated duration in weeks. This is the only field consumed by the next node's prompt.

### AI Agent (Generate Proposal)
`@n8n/n8n-nodes-langchain.agent`, typeVersion 3. See **AI/LLM Integration** section below for full prompt and schema. Uses `promptType: "define"` (fixed prompt template with `hasOutputParser: true`).

### OpenRouter Chat Model
`@n8n/n8n-nodes-langchain.lmChatOpenRouter`, typeVersion 1. Model: `nvidia/nemotron-nano-12b-v2-vl:free`. Connected to `AI Agent (Generate Proposal)` via the `ai_languageModel` connection type. No non-default options set.

### Structured Output Parser
`@n8n/n8n-nodes-langchain.outputParserStructured`, typeVersion 1.3. Defines a JSON schema example (the same 9-field proposal structure) that the AI Agent's output is coerced/validated against. Connected via `ai_outputParser`.

### Function (Parse AI Response)
`n8n-nodes-base.code`, typeVersion 2 (JavaScript). Reads `items[0].json.output` (or `.text`, or the raw json) from the AI Agent's result, strips ```` ```json ```` fences if present, and `JSON.parse()`s it. Validates presence of all 9 required proposal fields (`executive_summary`, `problem_statement`, `proposed_solution`, `scope_of_work`, `deliverables`, `timeline`, `assumptions`, `out_of_scope`, `next_steps`); throws if any are missing. On success, merges the parsed object under a `proposal` key and sets `status: "success"`. On any error (parse failure or missing fields), catches it and instead sets `proposal: null`, `status: "parse_error"`, `parse_error: <message>`.

### IF (Check Parse Success)
`n8n-nodes-base.if`, typeVersion 2.2. Single condition: `$json.status equals "success"`. **Only the true output is wired to a downstream node** — see Known Limitations.

### Set (Format for Sheets)
`n8n-nodes-base.set`, typeVersion 3.4. Flattens 20 fields for the spreadsheet row, pulling from three different upstream nodes by name:
- From `Set (Normalize Input)`: `Proposal_Id`, `Created_Date`, `Client_Name`, `Company_Name`, `Contact_Email`.
- From `Function (Enrich Context)`: `Industry`, `Project_Type`, `Complexity`, `Budget_Range`, `Timeline_Expectations`.
- From `Function (Parse AI Response)` (`.item.json.output.*`): `Executive_Summary`, `Problem_Statement`, `Proposed_Solution`, `Scope_of_Work` (array), `Deliverables` (array), `Timeline`, `Assumptions` (array), `Out_of_Scope` (array), `Next_Steps`.
- Static: `Status = "Generated"`.

**Note**: the array-typed fields (`Scope_of_Work`, `Deliverables`, `Assumptions`, `Out_of_Scope`) reference `Function (Parse AI Response).item.json.output.*`, but Node 9's code stores the parsed proposal under a `proposal` key, not `output` — see Known Limitations for the discrepancy this may cause.

### Google Sheets (Store Proposal)
`n8n-nodes-base.googleSheets`, typeVersion 4.7. Operation: `append`. Target spreadsheet: "AI Proposal Generator" (`documentId`: `1vCwn_2Q0koE3dTLmAzxTQp9G7Vl6lHuenyK3VgqzvLU`), sheet `Sheet1` (`gid=0`). Maps 20 columns 1:1 from `Set (Format for Sheets)` output, joining array fields with newlines (`.join('\n')`).

### Telegram (Send Notification)
`n8n-nodes-base.telegram`, typeVersion 1.2. Sends a Markdown-formatted message to a fixed `chatId` (`7350267365`) summarizing `Client_Name`, `Company_Name`, `Proposal_ID`, `Executive_Summary`, `Project_Type`, `Complexity`, and `Created_Date`.

### Set (Error Message)
`n8n-nodes-base.set`, typeVersion 3.4. Sets `error_type` (static string `"Validation Error or Parse Error"`), `error_details` (static string `"Relevant error message"`), `client_name` (pulled from `Set (Normalize Input)`), and `timestamp` (`{{ $now }}`). Does not carry forward the actual IF (Validation) failure reason.

### Telegram (Send Error Notification)
`n8n-nodes-base.telegram`, typeVersion 1.2. **Uses the same message template as `Telegram (Send Notification)`**, referencing `$json.Client_Name`, `$json.Company_Name`, `$json.Proposal_ID`, `$json.Executive_Summary`, etc. — none of these fields exist on the payload produced by `Set (Error Message)` (which only sets `error_type`, `error_details`, `client_name` (lowercase), `timestamp`). This will render as empty/blank values in the actual Telegram message. See Known Limitations.

## Branching Logic

Two IF nodes control flow:

1. **IF (Validation)** — condition: `client_name` not empty AND `company_name` not empty AND `project_description` not empty AND `project_description.length > 20` (combinator: `and`).
   - **True** → `Function (Enrich Context)` (main proposal generation path).
   - **False** → `Set (Error Message)` → `Telegram (Send Error Notification)`.

2. **IF (Check Parse Success)** — condition: `status equals "success"` (string comparison).
   - **True** → `Set (Format for Sheets)` (continues to storage + notification).
   - **False** → **not connected to any node.** Items that fail this check simply terminate with no notification, no logging, and no error surfaced to the caller beyond whatever the webhook's `lastNode` response mode returns (in this case, likely the raw `parse_error` object from `Function (Parse AI Response)`, since it's the last node to have executed on that branch).

## Variables & Expressions

Notable expressions across the workflow:

- `{{ $json.body.<field> }}` — extracts fields from the incoming webhook POST body (`Set (Normalize Input)`).
- `{{ $json.body.budget_range || "To be discussed" }}` / `{{ $json.body.timeline_expectations || "Flexible" }}` — fallback defaults for two optional input fields. Note `industry` and `contact_email` have **no equivalent fallback**.
- `{{ $now.toISO() }}` and `{{ $now.format('yyyy-MM-DD HH:mm:ss') }}` — timestamp generation (Luxon), used for `proposal_id` and `created_date`.
- `{{ $json.project_description.length }}` — numeric length check used in the IF (Validation) node.
- `{{ $('Set (Normalize Input)').item.json.* }}` and `{{ $('Function (Enrich Context)').item.json.* }}` — cross-node references (by node name) used extensively in `Set (Format for Sheets)` to pull fields from multiple upstream steps rather than only the immediately preceding node.
- `{{ $('Function (Parse AI Response)').item.json.output.* }}` — references the parsed AI proposal fields. As noted above, the Code node in `Function (Parse AI Response)` actually stores the parsed object under the key `proposal`, not `output`; this expression path may not resolve as intended (flag for testing).
- `.join('\n')` (Google Sheets node) / `.join('\\n')` (as escaped in the JSON) — flattens array fields (scope, deliverables, assumptions, out-of-scope) into newline-delimited strings for spreadsheet cells.

## Credentials Used

| Node | Credential Name | Credential Type | Service |
|---|---|---|---|
| OpenRouter Chat Model | n8n-regular | `openRouterApi` | OpenRouter |
| Google Sheets (Store Proposal) | Google Sheets account | `googleSheetsOAuth2Api` | Google Sheets |
| Telegram (Send Notification) | Telegram account for AI Journal | `telegramApi` | Telegram |
| Telegram (Send Error Notification) | Telegram account for AI Journal | `telegramApi` | Telegram |

No credential secrets are present in the export (n8n never exports secret values — only name/ID references, as shown above).

## Input Data

Triggered by an HTTP POST to `/generate-proposal` with a JSON body shaped like:

```json
{
  "client_name": "string",
  "company_name": "string",
  "project_description": "string (must be > 20 characters)",
  "budget_range": "string (optional, defaults to 'To be discussed')",
  "timeline_expectations": "string (optional, defaults to 'Flexible')",
  "industry": "string (optional)",
  "contact_email": "string (optional)"
}
```

## Output Data

- **On success**: one new row appended to the "AI Proposal Generator" Google Sheet (`Sheet1`), with 20 columns (see Google Sheets Schema below), plus a Telegram message sent to chat ID `7350267365` summarizing the proposal. The HTTP response to the original webhook caller returns whatever `Telegram (Send Notification)` outputs (per `responseMode: lastNode`).
- **On validation failure**: a Telegram message is sent via `Telegram (Send Error Notification)` (though populated with blank/undefined values per the template mismatch noted above); no Sheets row is written.
- **On AI parse failure**: no further action occurs; the branch dead-ends after `IF (Check Parse Success)`.

## Data Transformation

Data is reshaped at three points:
1. `Set (Normalize Input)` — unwraps webhook body fields into top-level fields and adds generated ID/timestamp fields.
2. `Function (Enrich Context)` — adds a derived `enriched_data` object (complexity, project type, timeline estimate, industry, urgency) via keyword heuristics on free text.
3. `Set (Format for Sheets)` — flattens the nested AI proposal object and cross-references three different upstream nodes into a single flat row shape matching the Sheets schema, joining array fields into newline-delimited strings.

## AI/LLM Integration

- **Node**: `AI Agent (Generate Proposal)` (`@n8n/n8n-nodes-langchain.agent`, typeVersion 3)
- **Model provider**: OpenRouter, via the connected `OpenRouter Chat Model` node
- **Model**: `nvidia/nemotron-nano-12b-v2-vl:free`
- **Output parsing**: `Structured Output Parser` enforces the JSON schema shown below (`hasOutputParser: true` on the Agent node)
- **Prompt type**: `define` (static template, not a dynamic/chat-driven agent prompt)

**Full prompt text** (as configured in the node, with `{{ $json.ai_context }}` interpolated at runtime):

> You are an expert business proposal writer specializing in B2B consulting and automation services.
>
> Generate a comprehensive, professional proposal based on the following client requirements. You MUST return your response as a valid JSON object ONLY, with no additional text before or after.
>
> CLIENT CONTEXT:
> `{{ $json.ai_context }}`
>
> REQUIRED JSON STRUCTURE: *(executive_summary, problem_statement, proposed_solution, scope_of_work[], deliverables[], timeline, assumptions[], out_of_scope[], next_steps — full schema below)*
>
> WRITING GUIDELINES: professional/consultative tone; specific but not overpromising; "we will"/"our team" language; measurable outcomes where possible; realistic timeline; explicit about in/out of scope; explicit assumptions; clear next steps.
>
> Return ONLY the JSON object, nothing else.

**Expected JSON output schema** (also configured identically in the Structured Output Parser node):

```json
{
  "executive_summary": "string",
  "problem_statement": "string",
  "proposed_solution": "string",
  "scope_of_work": ["string", "..."],
  "deliverables": ["string", "..."],
  "timeline": "string or array",
  "assumptions": ["string", "..."],
  "out_of_scope": ["string", "..."],
  "next_steps": "string"
}
```

**Downstream consumption**: `Function (Parse AI Response)` reads the agent's output, strips markdown fences, parses as JSON, and validates all 9 fields are present before flagging `status: "success"`.

## Error Handling

- **Input validation errors** (missing/short fields): handled explicitly via `IF (Validation)`'s false branch → `Set (Error Message)` → `Telegram (Send Error Notification)`. However, the error Set node populates only generic static placeholder text (`"Validation Error or Parse Error"` / `"Relevant error message"`), not the actual failing condition, and the downstream Telegram node's message template references fields (`Client_Name`, `Proposal_ID`, `Executive_Summary`, etc.) that don't exist on this branch's payload — the resulting Telegram message will likely show blank/undefined values for most fields.
- **AI parsing errors**: caught inside `Function (Parse AI Response)`'s try/catch and flagged via `status: "parse_error"`, but **`IF (Check Parse Success)`'s false output has no downstream connection** — these failures are not logged, notified, or surfaced anywhere beyond the (possibly synchronous) webhook HTTP response.
- No node in this workflow has `continueOnFail`/`onError` explicitly configured (not present in any node's parameters in the JSON).
- No workflow-level `errorWorkflow` is configured in `settings`.

## Retry Strategy

No `retryOnFail`, `maxTries`, or `waitBetweenTries` settings are present on any node in the JSON (including the Google Sheets, Telegram, and AI Agent/OpenRouter nodes, which are the most likely to hit transient failures). **No retry logic is configured anywhere in this workflow.**

## Failure Notifications

`Telegram (Send Error Notification)` fires only on the `IF (Validation)` false path. There is no equivalent failure notification for AI parse failures (see Error Handling above).

## Execution Order

`settings.executionOrder: "v1"` — connection-based execution order (the modern default), meaning nodes execute based on when their inputs are ready rather than strict top-to-bottom canvas order.

## Security Considerations

- The `Webhook (Input Trigger)` node has **no authentication configured** (no header auth, basic auth, or JWT). The `/generate-proposal` endpoint is publicly callable by anyone who knows (or guesses) the URL. Recommend adding webhook authentication if this is exposed beyond trusted internal use.
- Credentials (OpenRouter, Google Sheets OAuth2, Telegram Bot API) are referenced by ID only; no secrets are present in this export.
- The Telegram `chatId` (`7350267365`) is hardcoded in both Telegram nodes rather than parameterized.
- _[Further security review to be provided by workflow owner/security reviewer.]_

## Performance Considerations

No batch sizing, pagination, or rate-limiting configuration is present (none of the nodes process arrays of items in batches — the workflow is designed for single-item execution per webhook call). The free-tier OpenRouter model (`nvidia/nemotron-nano-12b-v2-vl:free`) may be subject to provider-side rate limits or slower response times typical of free-tier LLM endpoints; no timeout override is configured on the AI Agent or OpenRouter Chat Model node.

## Execution Time

_[To be provided by workflow owner — requires execution data]_

## Resource Usage

_[To be provided by workflow owner — requires execution data]_

## Testing & Validation

_[To be provided by workflow owner]_

## Troubleshooting

- **No Telegram success message arrives**: Check `IF (Validation)` — confirm all four conditions pass (client_name, company_name, project_description non-empty, description > 20 chars). Also check `IF (Check Parse Success)`; if the AI response failed to parse, execution dead-ends silently with no notification (see Known Limitations).
- **Telegram error message shows blank fields**: Expected given the current mismatch between `Set (Error Message)`'s output fields and the Telegram error template's expected fields (`Client_Name`, `Proposal_ID`, `Executive_Summary`, etc. are never set on that branch).
- **Google Sheets row has blank scope/deliverables/assumptions/out-of-scope columns**: Check whether `Function (Parse AI Response)` outputs the parsed data under `proposal` vs. what `Set (Format for Sheets)` expects (`output`) — this key mismatch (see Variables & Expressions) may cause these expressions to resolve to `undefined`.
- **Webhook returns unexpected/raw JSON instead of a clean confirmation**: `responseMode` is `lastNode`, so the HTTP response body is whatever the last-executed node in that specific branch returns (e.g., raw Telegram API response, or a raw `parse_error` object on the unhandled parse-failure branch). Consider adding a dedicated "Respond to Webhook" node with a defined response shape if callers depend on a stable response contract.

## Known Limitations

- `IF (Check Parse Success)` false branch has no downstream connection — AI/parsing failures are silently dropped with no logging or notification.
- `Set (Error Message)` populates static placeholder text (`"Validation Error or Parse Error"`, `"Relevant error message"`) instead of the actual failed condition/error detail.
- `Telegram (Send Error Notification)` reuses the success-path message template, referencing fields (`Client_Name`, `Company_Name`, `Proposal_ID`, `Executive_Summary`, `Project_Type`, `Complexity`, `Created_Date`) that are not present on the error branch's payload.
- Possible key mismatch: `Function (Parse AI Response)` stores parsed AI output under `proposal`, but `Set (Format for Sheets)` reads from `.output.*` for the array fields — needs verification against live execution data.
- Google Sheets column `Timeline_Weeks` is actually populated from `Timeline_Expectations` (the client's free-text input, e.g. "Flexible") rather than the numeric `enriched_data.estimated_timeline_weeks` computed in `Function (Enrich Context)` — likely not the intended value for that column.
- No retry logic configured on any external-call node (OpenRouter/AI Agent, Google Sheets, Telegram).
- No webhook authentication configured — open endpoint.
- `industry` and `contact_email` input fields have no fallback default (unlike `budget_range`/`timeline_expectations`), so they may be `undefined` if omitted by the caller (the enrichment Code node does independently default `industry` to `"General"` internally).
- Telegram `chatId` and Google Sheets `documentId` are hardcoded, not environment/parameter-driven — moving this workflow between environments (dev/prod) requires manual node edits.

## Assumptions

- Assumed the workflow runs in the `Asia/Kolkata` timezone as set in `settings.timezone`, applying to all `$now` expressions.
- Assumed `Function (Parse AI Response)`'s output key (`proposal`) vs. `Set (Format for Sheets)`'s referenced key (`output`) is a genuine discrepancy rather than something resolved implicitly by n8n's LangChain Agent output wrapping — flagged for the owner to verify against an actual execution.
- Assumed the two "Chat Model" / "Output Parser" sub-nodes visible under the AI Agent in the screenshot correspond exactly to `OpenRouter Chat Model` and `Structured Output Parser` per the JSON's `ai_languageModel`/`ai_outputParser` connections; no memory node is present in the JSON despite the AI Agent supporting one.
- Assumed `responseMode: lastNode` means the webhook caller receives the raw output of whichever node last executes on the taken branch, since no dedicated "Respond to Webhook" node exists.

## Version History / Change Log

_[To be provided by workflow owner]_

## References

_[To be provided by workflow owner]_

## Maintenance Notes

_[To be provided by workflow owner]_
