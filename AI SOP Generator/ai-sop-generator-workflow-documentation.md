# AI SOP Generator — Workflow Documentation

---

## Workflow Overview

| Field | Value |
|---|---|
| **Workflow Name** | AI SOP Generator |
| **Workflow ID** | `S6Mm4KM0AGvOUwlz` |
| **Version ID** | `c253d1c6-087f-4dbd-ad0c-e83cd0361fdf` |
| **Status** | Inactive (not yet activated) |
| **Execution Order** | v1 (connection-based) |
| **Owner** | _[To be provided by workflow owner]_ |
| **Last Updated** | _[To be provided by workflow owner]_ |

---

## Purpose

This workflow converts unstructured process notes submitted via HTTP POST into fully structured, 10-section Standard Operating Procedures (SOPs) using an AI language model. The generated SOP is stored in Google Sheets, and the submitter/team is notified via Telegram. A JSON response is returned to the caller confirming the SOP was successfully generated.

**Primary use case:** Internal operations teams, documentation managers, or department leads who need to transform rough process notes into consistent, professional SOPs without manual writing effort.

---

## Workflow Summary

```
Webhook (POST)
  → Set (Normalize Inputs)
    → If (Validate Input)
        → [TRUE] Function (Build AI Prompt Context)
                  → AI Agent (Generate SOP)   ← powered by OpenRouter Chat Model
                    → Function (Parse & Validate AI Output)
                      → Function (Convert to Markdown)
                        → Google Sheets (Store SOP)
                          → Telegram (Notify Owner)
                            → Webhook Response (Return Result)
        → [FALSE] ← execution stops (no false-branch node connected)
```

The workflow is entirely linear on the happy path. The IF node acts as a guard; invalid submissions are silently dropped (no false-branch notification node is wired in the current JSON — see [Assumptions](#assumptions)).

---

## Trigger Details

| Field | Value |
|---|---|
| **Node** | `Webhook` |
| **Type** | `n8n-nodes-base.webhook` v2.1 |
| **HTTP Method** | `POST` |
| **Path** | `sop-generator` |
| **Full URL (prod)** | `https://<your-n8n-host>/webhook/sop-generator` |
| **Full URL (test)** | `https://<your-n8n-host>/webhook-test/sop-generator` |
| **Response Mode** | `lastNode` — response is held until the final node executes |
| **Webhook ID** | `9f71be17-a3f8-4691-ab9f-ec8e072d1c7d` |

**Expected POST body (JSON):**

```json
{
  "process_name": "Monthly Vendor Invoice Review",
  "process_notes": "Every month finance team checks all vendor invoices...",
  "department": "Finance",
  "tools_involved": "QuickBooks, Google Sheets, Email",
  "frequency": "monthly"
}
```

`process_name` and `process_notes` are mandatory (enforced by the IF node). `department`, `tools_involved`, and `frequency` are optional.

---

## Node List

| # | Node Name | Type | Purpose |
|---|---|---|---|
| 1 | `Webhook` | `n8n-nodes-base.webhook` | Receives POST request with process data |
| 2 | `Set (Normalize Inputs)` | `n8n-nodes-base.set` | Extracts + standardizes all input fields; generates submission metadata |
| 3 | `If` | `n8n-nodes-base.if` | Validates that mandatory fields meet minimum length requirements |
| 4 | `Function (Build AI Prompt Context)` | `n8n-nodes-base.code` | Cleans process notes; assembles a structured context block for the AI |
| 5 | `AI Agent (Generate SOP)` | `@n8n/n8n-nodes-langchain.agent` | Calls the LLM to generate the 10-section SOP as JSON |
| 6 | `OpenRouter Chat Model` | `@n8n/n8n-nodes-langchain.lmChatOpenRouter` | Supplies the LLM (nvidia/nemotron-nano-12b-v2-vl:free) to the AI Agent |
| 7 | `Function (Parse & Validate AI Output)` | `n8n-nodes-base.code` | Parses AI JSON output; strips code fences; applies field-level fallbacks |
| 8 | `Function (Convert to Markdown)` | `n8n-nodes-base.code` | Converts structured JSON SOP into a formatted Markdown document |
| 9 | `Google Sheets (Store SOP)` | `n8n-nodes-base.googleSheets` | Appends one row per SOP to the SOP Registry spreadsheet |
| 10 | `Telegram (Notify Owner)` | `n8n-nodes-base.telegram` | Sends a Telegram message with SOP title, purpose, and 3-step preview |
| 11 | `Webhook Response (Return Result)` | `n8n-nodes-base.respondToWebhook` | Returns success JSON to the original caller |

---

## Node-by-Node Description

---

### Node 1 — `Webhook`

**Type:** `n8n-nodes-base.webhook` v2.1

The entry point of the workflow. Listens on the path `sop-generator` for HTTP POST requests. The `responseMode` is set to `lastNode`, meaning the HTTP connection is held open until the `Webhook Response (Return Result)` node executes, at which point the response is sent back to the caller. This gives the caller confirmation that the full SOP pipeline has completed.

The node has pinned test data from a Postman request simulating a Finance department submission (`Monthly Vendor Invoice Review`), which was used during development.

**Connects to:** `Set (Normalize Inputs)`

---

### Node 2 — `Set (Normalize Inputs)`

**Type:** `n8n-nodes-base.set` v3.4

Extracts all five input fields from `$json.body.*` (the webhook data wrapper) and maps them to clean top-level fields. Also computes two metadata fields at this step:

| Output Field | Source / Expression |
|---|---|
| `process_name` | `$json.body.process_name` |
| `process_notes` | `$json.body.process_notes` |
| `department` | `$json.body.department` |
| `tools_involved` | `$json.body.tools_involved` |
| `frequency` | `$json.body.frequency` |
| `submitted_at` | `$now.toISO()` — ISO 8601 timestamp at execution time |
| `submission_id` | `$now.toMillis()` — Unix epoch in milliseconds, used as unique ID |

> **Note:** No default values are set here for optional fields. If `department`, `tools_involved`, or `frequency` are omitted from the POST body, they will be `undefined` downstream. The AI prompt is robust enough to handle this gracefully, but the Google Sheets row will have empty cells for those columns.

**Connects to:** `If`

---

### Node 3 — `If`

**Type:** `n8n-nodes-base.if` v2.2

Validates the two mandatory fields before allowing the workflow to proceed to the AI step. Uses strict type validation with AND logic.

| Condition | Check |
|---|---|
| `process_notes` length | `> 20` characters |
| `process_name` length | `> 2` characters |

Both conditions must be true for execution to proceed on the **TRUE** branch. If either fails, the **FALSE** branch is taken. **Currently, no node is connected to the FALSE branch** — invalid submissions are silently dropped and the webhook caller receives no response (the connection will time out). See [Assumptions](#assumptions) for a recommended fix.

**TRUE connects to:** `Function (Build AI Prompt Context)`
**FALSE connects to:** _(nothing — dead end)_

---

### Node 4 — `Function (Build AI Prompt Context)`

**Type:** `n8n-nodes-base.code` v2 (JavaScript)

Performs two jobs:

**1. Text cleaning:** Strips leading/trailing whitespace, collapses multiple spaces into one, and removes non-printable ASCII characters from `process_notes` to prevent prompt injection artifacts.

**2. Context block assembly:** Constructs a structured plain-text block combining all input fields:

```
PROCESS NAME: <value>
DEPARTMENT: <value>
FREQUENCY: <value>
TOOLS INVOLVED: <value>

RAW PROCESS NOTES:
<cleaned notes>
```

Also adds two static fields to the item: `sop_version: "1.0"` and `status: "draft"`.

The `context_block` output is the string injected directly into the AI Agent's user prompt.

**Connects to:** `AI Agent (Generate SOP)`

---

### Node 5 — `AI Agent (Generate SOP)`

**Type:** `@n8n/n8n-nodes-langchain.agent` v3

The core AI processing node. Uses a defined prompt (not conversational chat history) via `promptType: "define"`.

**System prompt instructs the model to:**
- Act as a senior operations manager and technical writer
- Write in clear, simple, non-technical language
- Return **only** valid JSON with no markdown fences or commentary
- Generate 6–15 steps
- Generate at least 3 items in every array field

**User prompt (injected per execution):**
```
Here is the process information to convert into an SOP:

<context_block from Node 4>

Generate the complete SOP JSON now.
```

**Expected JSON output schema:**
```json
{
  "sop_title": "string",
  "purpose": "string",
  "scope": "string",
  "prerequisites": ["string"],
  "steps": ["string"],
  "roles_and_responsibilities": ["string"],
  "tools_used": ["string"],
  "common_mistakes": ["string"],
  "quality_checks": ["string"],
  "revision_notes": "string"
}
```

The `hasOutputParser: true` flag is set, connecting to the output parser sub-node (visible in the canvas screenshot as `TooOutput Parser`). This sub-node is the default structured output parser.

**Sub-nodes (LangChain connections):**
- `ai_languageModel` ← `OpenRouter Chat Model`
- `ai_memory` ← `Chat MemoMemory` (visible in canvas; not present in JSON as a separate node — this is an auto-added default window buffer memory)

**Connects to:** `Function (Parse & Validate AI Output)`

---

### Node 6 — `OpenRouter Chat Model`

**Type:** `@n8n/n8n-nodes-langchain.lmChatOpenRouter` v1

Provides the language model to the AI Agent via the `ai_languageModel` connection type.

| Setting | Value |
|---|---|
| **Model** | `nvidia/nemotron-nano-12b-v2-vl:free` |
| **Provider** | OpenRouter (free tier) |
| **Credential** | `OpenRouter account for AI Journal` (ID: `YCawzcwbZd7VllSz`) |

This is a free-tier OpenRouter model. No additional options are configured (temperature, max tokens, etc. all use provider defaults).

**Connects to:** `AI Agent (Generate SOP)` via `ai_languageModel`

---

### Node 7 — `Function (Parse & Validate AI Output)`

**Type:** `n8n-nodes-base.code` v2 (JavaScript)

Defensive post-processing of the AI's raw output. Handles three scenarios:

**1. Markdown code fence stripping:** If the model returns JSON wrapped in ` ```json ... ``` ` despite the prompt instruction, the fences are stripped before parsing.

**2. JSON parsing with error throwing:** If the cleaned string cannot be parsed as valid JSON, the node throws a descriptive error containing the first 300 characters of the raw output to aid debugging.

**3. Field-level fallbacks:** Each field of the parsed SOP is individually validated. Arrays that are not arrays default to `[]`. Strings that are empty or missing default to `''`. The `sop_title` falls back to `<process_name> — SOP` if the AI omits it.

The node also carries forward all metadata fields (`process_name`, `department`, `frequency`, `tools_involved`, `submitted_at`, `submission_id`, `sop_version`, `status`) by pulling them from the previous node's input, ensuring nothing is lost across the chain.

**Connects to:** `Function (Convert to Markdown)`

---

### Node 8 — `Function (Convert to Markdown)`

**Type:** `n8n-nodes-base.code` v2 (JavaScript)

Renders the validated JSON SOP into a clean Markdown document string, stored in the `markdown_sop` field. The Markdown structure:

```
# <sop_title>
**Version:** | **Department:** | **Frequency:**
---
## Purpose & Objective
## Scope
## Prerequisites        (bullet list)
## Step-by-Step Procedure  (numbered list)
## Roles & Responsibilities  (bullet list)
## Tools & Systems Used      (bullet list)
## Common Mistakes & Edge Cases  (bullet list)
## Quality Checks            (bullet list)
## Revision Notes
---
*Generated on: <submitted_at> | Submission ID: <submission_id>*
```

This `markdown_sop` field is what gets written to the Google Sheets `Markdown SOP` column, making it directly pasteable into Notion, Confluence, or any Markdown-aware tool.

**Connects to:** `Google Sheets (Store SOP)`

---

### Node 9 — `Google Sheets (Store SOP)`

**Type:** `n8n-nodes-base.googleSheets` v4.7

**Operation:** `append` — adds one new row per SOP to the sheet.

**Target spreadsheet:**
- **Document ID:** `1Kk7hXmBtfyejqTJguzyiazpRRuhe1ZmMcKdMUdhLOeY`
- **Document name:** `AI SOP Generator`
- **Sheet:** `Sheet1` (gid=0)
- **URL:** `https://docs.google.com/spreadsheets/d/1Kk7hXmBtfyejqTJguzyiazpRRuhe1ZmMcKdMUdhLOeY/`

**Column mapping:**

| Sheet Column | Source Expression | Notes |
|---|---|---|
| `Submission ID` | `$('Set (Normalize Inputs)').item.json.submission_id` | Epoch ms timestamp |
| `SOP Title` | `$json.sop_title` | AI-generated |
| `Process Name` | `$('Set (Normalize Inputs)').item.json.process_name` | From original input |
| `Department` | `$('Set (Normalize Inputs)').item.json.department` | From original input |
| `Frequency` | `$('Set (Normalize Inputs)').item.json.frequency` | From original input |
| `Version` | `$('Function (Parse & Validate AI Output)').item.json.revision_notes` | Contains full revision note string, not just version number |
| `Status` | `"pending"` (hardcoded) | Always set to pending on creation |
| `Purpose` | `$json.purpose` | AI-generated |
| `Scope` | `$json.scope` | AI-generated |
| `Steps Count` | `$json.steps.length` | Number of steps generated |
| `Markdown SOP` | `$json.markdown_sop` | Full formatted Markdown document |
| `Raw JSON` | Multi-line expression (see below) | Human-readable key-value dump of original inputs |
| `Submitted At` | `$('Set (Normalize Inputs)').item.json.submitted_at` | ISO 8601 timestamp |

> **Note on `Version` column:** This is mapped to `revision_notes` (the full AI-generated text string like `"Version 1.0 — Initial SOP generated from process notes on…"`), not to the `sop_version` field (`"1.0"`). Consider remapping to `sop_version` if you want a sortable/filterable version number.

> **Note on `Raw JSON` column:** This is not a serialized JSON object but a manually formatted multi-line key-value string of the original inputs. This is human-readable but not machine-parseable. If re-processing is needed, consider using `JSON.stringify($json)` instead.

**Credential:** `Google Sheets account` (OAuth2, ID: `twmxeUbOfErGsbbM`)

**Connects to:** `Telegram (Notify Owner)`

---

### Node 10 — `Telegram (Notify Owner)`

**Type:** `n8n-nodes-base.telegram` v1.2

Sends a Telegram message to a hardcoded chat ID when an SOP is successfully stored.

**Target Chat ID:** `7350267365` (hardcoded — personal or bot chat)

**Message format:**
```
✅ New SOP Generated

📄 Title: <SOP Title>
🏢 Department: <Department>
🔁 Frequency: <Frequency>
🆔 Submission ID: <Submission ID>

Purpose: <Purpose>

First 3 Steps Preview:
1. <steps[0]>
2. <steps[1]>
3. <steps[2]>

📊 Total Steps: <Steps Count>
🗓️ Generated: <Submitted At>
```

Steps 1–3 are pulled from `$('Function (Convert to Markdown)').item.json.steps` (back-referencing the pre-Sheets node). All other fields come from the Google Sheets node's output (which mirrors the sheet row data).

**Credential:** `Telegram account for AI Journal` (ID: `WM4UJf38iPD545XB`)

**Connects to:** `Webhook Response (Return Result)`

---

### Node 11 — `Webhook Response (Return Result)`

**Type:** `n8n-nodes-base.respondToWebhook` v1.4

Closes the HTTP connection opened by the Webhook trigger and returns a JSON success payload to the caller.

**Response body:**
```json
{
  "success": true,
  "sop_title": "<SOP Title from Google Sheets node>",
  "submission_id": <Submission ID>,
  "steps_generated": "<Steps Count>",
  "message": "SOP generated and stored successfully."
}
```

This is the terminal node. Once it executes, the workflow execution is complete.

---

## Branching Logic

### IF Node — Input Validation

| Branch | Condition | Next Node |
|---|---|---|
| TRUE | `process_notes.length > 20` AND `process_name.length > 2` | `Function (Build AI Prompt Context)` |
| FALSE | Either condition fails | _(no node connected — execution halts silently)_ |

**Current gap:** The FALSE branch has no connected node. A caller submitting an invalid payload will receive no HTTP response (the `lastNode` response mode will time out). Recommended fix: connect a `Respond to Webhook` node on the FALSE branch returning a `400` error message.

---

## AI / LLM Integration

| Property | Value |
|---|---|
| **Framework** | LangChain via n8n native langchain nodes |
| **Agent type** | `@n8n/n8n-nodes-langchain.agent` (ReAct-style agent) |
| **Agent version** | v3 |
| **LLM Provider** | OpenRouter |
| **Model** | `nvidia/nemotron-nano-12b-v2-vl:free` |
| **Prompt mode** | `define` (fixed prompt per execution, not chat history) |
| **Output parser** | Structured output parser (auto-connected) |
| **Memory** | Window buffer memory (auto-connected, default settings) |
| **Temperature** | Provider default (not configured) |
| **Max tokens** | Provider default (not configured) |

The AI is instructed via system prompt to return **only** valid JSON. The downstream parser node (`Function (Parse & Validate AI Output)`) handles cases where the model ignores this instruction and wraps output in markdown fences.

---

## Credentials Used

| Credential Name | Type | Used In Node | Scope |
|---|---|---|---|
| `OpenRouter account for AI Journal` | `openRouterApi` | `OpenRouter Chat Model` | LLM API calls to OpenRouter |
| `Google Sheets account` | `googleSheetsOAuth2Api` | `Google Sheets (Store SOP)` | Append rows to SOP Registry spreadsheet |
| `Telegram account for AI Journal` | `telegramApi` | `Telegram (Notify Owner)` | Send messages to Telegram chat |

> **Security note:** No credential secrets are stored in the workflow JSON export (n8n never exports secret values). All three credentials must be configured in the n8n instance's credential store before the workflow can execute.

---

## Variables & Expressions

Key expressions used across nodes:

| Expression | Location | Purpose |
|---|---|---|
| `$json.body.process_name` | `Set (Normalize Inputs)` | Extracts field from webhook body wrapper |
| `$now.toISO()` | `Set (Normalize Inputs)` | ISO 8601 submission timestamp |
| `$now.toMillis()` | `Set (Normalize Inputs)` | Epoch ms as unique submission ID |
| `$json.process_notes.length` | `If` | Guards minimum notes length |
| `$json.context_block` | `AI Agent (Generate SOP)` | Injects assembled prompt context |
| `$input.first().json.output` | `Function (Parse & Validate AI Output)` | Reads AI Agent raw output |
| `$('Set (Normalize Inputs)').item.json.*` | `Google Sheets (Store SOP)` | Back-references original inputs after AI processing |
| `$('Function (Convert to Markdown)').item.json.steps[0..2]` | `Telegram (Notify Owner)` | Pulls first 3 steps for preview |
| `$('Google Sheets (Store SOP)').item.json['SOP Title']` | `Webhook Response` | Includes generated title in API response |

---

## API Integrations & External Services

| Service | Node | Operation | Endpoint / Target |
|---|---|---|---|
| **OpenRouter** | `OpenRouter Chat Model` | LLM inference | `https://openrouter.ai/api/v1` |
| **Google Sheets** | `Google Sheets (Store SOP)` | Append row | Spreadsheet ID `1Kk7hXmBtfyejqTJguzyiazpRRuhe1ZmMcKdMUdhLOeY`, Sheet1 |
| **Telegram Bot API** | `Telegram (Notify Owner)` | `sendMessage` | Chat ID `7350267365` |

---

## Error Handling

| Scenario | Current Behaviour | Recommendation |
|---|---|---|
| Invalid/short input | IF FALSE branch — execution silently halts, caller times out | Connect `Respond to Webhook` on FALSE branch with HTTP 400 and error message |
| AI returns non-JSON | `Function (Parse & Validate AI Output)` throws a descriptive error, execution fails | Add a try/catch wrapper that returns a fallback SOP stub or notifies the owner via Telegram |
| Google Sheets write fails | Execution fails, no notification sent, no webhook response | Enable `continueOnFail` on the Sheets node + add error branch |
| Telegram send fails | Execution fails, webhook response never sent | Enable `continueOnFail` on Telegram node |
| OpenRouter rate limit / model unavailable | AI Agent node fails, execution stops | No retry logic configured — consider enabling `retryOnFail` on AI Agent node |

No global error workflow is configured (`settings.errorWorkflow` is absent from the JSON).

---

## Google Sheets Schema

**Sheet name:** `Sheet1`

| Column | Data Type | Source | Notes |
|---|---|---|---|
| `Submission ID` | String (number) | `submission_id` (epoch ms) | Acts as unique row key |
| `SOP Title` | String | AI-generated | Short descriptive title |
| `Process Name` | String | Original input | Submitted by caller |
| `Department` | String | Original input | May be empty if not provided |
| `Frequency` | String | Original input | e.g. daily, weekly, monthly, ad-hoc |
| `Version` | String | `revision_notes` from AI output | Full AI-generated revision note string |
| `Status` | String | Hardcoded `"pending"` | Manual update required to change to approved/archived |
| `Purpose` | String | AI-generated | 1–2 sentence objective statement |
| `Scope` | String | AI-generated | Who/what the SOP applies to |
| `Steps Count` | Number | `steps.length` | Total number of procedure steps |
| `Markdown SOP` | Long string | Rendered by Node 8 | Full Markdown document, pasteable into docs tools |
| `Raw JSON` | Long string | Key-value dump | Human-readable original input summary |
| `Submitted At` | String (ISO 8601) | `submitted_at` | Timestamp of submission |

---

## Pinned Test Data

The workflow includes pinned (frozen) test data on two nodes for development/testing:

**`Webhook` — pinned payload:**
```json
{
  "process_name": "Monthly Vendor Invoice Review",
  "process_notes": "Every month finance team checks all vendor invoices...",
  "department": "Finance",
  "tools_involved": "QuickBooks, Google Sheets, Email",
  "frequency": "monthly"
}
```

**`Set (Normalize Inputs)` — pinned output:**
The normalized version of the above, with `submitted_at: "2026-02-17T23:05:55.055-05:00"` and `submission_id: "1771387555062"`.

> When testing, n8n will use this pinned data instead of live execution data for these two nodes. Remember to unpin or run with live data before validating the full pipeline end-to-end.

---

## Assumptions

1. **FALSE branch of IF node** — The current JSON has no node connected to the IF node's FALSE output. It is assumed this was intentionally left incomplete during development and that an error response node should be added before production activation.

2. **Optional fields not defaulted** — `department`, `tools_involved`, and `frequency` have no fallback values. If the caller omits them, the AI prompt will contain blank/undefined values for those fields. The AI prompt does not explicitly handle missing optional fields. A `Set` node or inline default expressions (e.g. `{{ $json.body.department ?? 'General Operations' }}`) are recommended.

3. **Hardcoded Chat ID** — The Telegram Chat ID (`7350267365`) is hardcoded in the node parameters. This limits the workflow to notifying a single recipient. If multi-recipient or dynamic routing is needed, this should be parameterized.

4. **Model selection** — `nvidia/nemotron-nano-12b-v2-vl:free` is a free-tier multimodal model on OpenRouter. Its JSON compliance and instruction-following may vary. The parser node in Node 7 provides a safety net, but for production use, consider switching to a model with stronger JSON output reliability (e.g. `mistralai/mistral-7b-instruct:free` or a non-free model with JSON mode).

5. **No deduplication** — The Google Sheets append always adds a new row. There is no check for duplicate submissions (same process name re-submitted). Duplicate detection would require a Sheets read + IF check before the append.

6. **Workflow is inactive** — `"active": false` in the JSON. The workflow must be manually activated in the n8n UI before it can receive live webhook requests.

---

## Maintenance Notes

| Task | How |
|---|---|
| Change the LLM model | Update the `model` parameter in `OpenRouter Chat Model` node |
| Change the notification recipient | Update `chatId` in `Telegram (Notify Owner)` node |
| Change the target spreadsheet | Update `documentId` and `sheetName` in `Google Sheets (Store SOP)` node |
| Add error handling on FALSE branch | Connect a `Respond to Webhook` node to IF node's FALSE output |
| Add default values for optional fields | Add `?? 'default'` fallbacks in `Set (Normalize Inputs)` expressions |
| Update the SOP prompt | Edit the `systemMessage` in `AI Agent (Generate SOP)` options |
| Activate the workflow | Toggle Active switch in n8n UI, or use n8n API `PATCH /workflows/{id}/activate` |

---

## Sections Not Applicable

The following optional documentation sections were omitted because they do not apply to this workflow:

- **Loop Logic** — No `SplitInBatches` or loop-over-items nodes
- **Database Operations** — No database nodes (Postgres, MySQL, etc.)
- **File Operations** — No binary file read/write nodes
- **Scheduling Details** — Trigger is webhook, not cron/schedule
- **Environment Variables** — No `$env` references used
- **Sub-workflow Dependencies** — No `Execute Workflow` calls

---

## Sections Requiring Owner Input

The following fields are left as placeholders and require input from the workflow owner to complete this documentation:

- `Owner / Maintainer` — Who is responsible for this workflow?
- `Last Updated` — When was the workflow last modified?
- `Business Use Case / Justification` — Why was this built? What team problem does it solve?
- `Version History / Change Log` — What changes have been made since initial creation?
- `Security Review` — Has the hardcoded Chat ID, Spreadsheet ID, and OpenRouter model been reviewed for compliance?
- `Testing & Validation Results` — Has end-to-end testing been performed with real data?
- `Known Limitations` — Are there any known edge cases beyond those identified above?
- `SLA / Expected Execution Time` — What is the acceptable response time for SOP generation?
