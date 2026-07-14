# AI Financial Anomaly Detection & Expense Reconciliation — Workflow Documentation

| Field | Value |
|---|---|
| Workflow Name | AI Financial Anomaly Detection & Expense Reconciliation |
| Workflow ID | `olTTPupSTvcupj1F` |
| Internal Version ID | `7c0d30f4-0aef-4e47-9450-d4dfa0abddd3` (n8n internal revision hash — not a human version number) |
| Version | v1.0 (initial documentation) |
| Owner | _[To be provided by workflow owner]_ |
| Workflow Last Saved | _[Not present in export — `updatedAt` field absent]_ |
| Document Generated | 2026-07-14 |
| Active Status | ⚠️ **INACTIVE** — `"active": false` in export. Must be toggled ON before production use. |

---

## Purpose

This workflow automates daily financial anomaly detection across corporate expense transactions. It reads all transaction records from a Google Sheet, runs a JavaScript-based rule engine to pre-flag duplicates, budget overruns, and policy violations, then forwards each enriched transaction to a free LLM (via OpenRouter) for deeper contextual fraud analysis. Based on the AI's assessed severity, it routes alerts to Gmail (high) or Slack (medium), suppresses noise for low-severity findings, and writes every detected anomaly to a permanent audit log in Google Sheets — all without any paid third-party services beyond the n8n instance itself.

---

## Business Use Case

> ⚠️ _Inferred from workflow structure — confirm or replace with the team's actual use case._

This automation is intended to reduce manual financial review overhead by surfacing duplicate payments, policy-violating expenses, and budget overruns before they are processed or approved. It serves as an early-warning system for finance teams, compliance auditors, or engineering leads who manage shared expense budgets.

---

## Workflow Summary

The workflow is triggered automatically every day at 08:00. It reads the entire `Transactions` sheet from a designated Google Spreadsheet and processes each row through a JavaScript Code node that computes fingerprint-based duplicates, compares category spend against hard-coded budget limits, and flags weekend purchases, round-number amounts, and high-value transactions. Each enriched transaction is then sent individually to a LangChain AI Agent node backed by the `nvidia/nemotron-nano-12b-v2-vl:free` model on OpenRouter, which analyses it against a detailed financial fraud detection system prompt and returns a structured JSON report of anomalies. A second Code node parses and normalises the LLM's output into discrete anomaly items, each carrying a severity level. A Switch node routes these items: high-severity findings trigger an HTML email to the finance team via Gmail; medium-severity findings post a message to Slack; low-severity findings pass through a Set node that marks them as log-only. All three branches converge at a Merge node, and every anomaly — regardless of severity — is appended as a new row to the `AuditLog` sheet, creating an immutable audit trail.

---

## Workflow Diagram

> See the attached canvas screenshot (`AI_Financial_Anomaly_Detection___Expense_Reconciliation.webp`).

**Layout shape:** Linear pipeline with a three-way fan-out alert stage that re-converges before the terminal write node.

```
Schedule Trigger
      │
      ▼
'Fetch Transactions' (Google Sheets)  ←── reads Transactions sheet
      │
      ▼
Pre-Process & Deduplicate  ←── JS rule engine; 1 enriched item out per transaction in
      │
      ▼
AI Anomaly Detection (LangChain Agent)
      │  └── [sub-node] OpenRouter Chat Model (nvidia/nemotron-nano-12b-v2-vl:free)
      │
      ▼
Parse AI Response  ←── normalises LLM JSON; emits 1 item per anomaly + 1 summary item
      │
      ▼
Switch (routes on $json.severity)
      ├── Output 0: "high"   ──► Gmail Alert — High Severity
      ├── Output 1: "medium" ──► Slack — Medium Severity
      └── Output 2: "low"    ──► Set Node — Low Severity
              │                       │                    │
              └───────────────────────┴────────────────────┘
                                      │
                                      ▼
                               Merge (3 inputs)
                                      │
                                      ▼
                          Write Audit Log (Google Sheets)  ←── appends to AuditLog sheet
```

---

## Trigger Details

| Field | Value |
|---|---|
| Trigger Node | `Schedule Trigger` |
| Trigger Type | Schedule (`n8n-nodes-base.scheduleTrigger` v1.3) |
| Frequency | Daily |
| Time | 08:00 (hour 8) |
| Timezone | _Not explicitly set in `settings` — defaults to the n8n instance timezone_ |
| Manual override | Workflow can be run manually via the **Execute workflow** button visible in the canvas |

The trigger carries no input payload — it simply fires and passes an empty execution context to the next node.

---

## Execution Flow

1. **Schedule Trigger** fires at 08:00 every day, initiating a new execution.

2. **`'Fetch Transactions' (Google Sheets)`** reads all rows from the `Transactions` tab (gid `0`) of spreadsheet `1Rt8ENolpb0rQ43qCGf-PLdIoN3ZxNve3syCyNDaGY24`. Each row becomes one n8n item. No row filtering is applied — all rows are fetched on every run.

3. **`Pre-Process & Deduplicate`** (Code node) receives all transaction items and runs four operations across the full dataset in a single execution:
   - **Duplicate fingerprinting**: builds a `vendor|amount|date|employee` key per transaction; any collision marks both the original and the duplicate with a `duplicate_info` object.
   - **Budget comparison**: sums amounts per `category` field and compares against hard-coded limits (`Cloud Infrastructure: 10000`, `Travel: 5000`, `Meals & Entertainment: 2000`, `Software: 3000`, `Office Supplies: 1000`, `Marketing: 8000`). Over-budget categories are collected in a Set for O(1) lookup.
   - **Policy violation tagging**: flags `weekend_transaction` (non-Travel purchase on Saturday/Sunday), `suspicious_round_number` (amount ≥ 5000 and divisible by 1000), and `high_value` (amount > 10000) into a `violations[]` array per transaction.
   - **Output**: returns one enriched item per input transaction (not a single aggregated object), carrying the original fields plus `duplicate_info`, `category_total`, `category_budget`, `is_over_budget`, `violations`, `violation_count`, `is_weekend`, `is_high_risk`, `total_transactions`, `total_spend`, and `processed_at`.

4. **`AI Anomaly Detection`** (LangChain Agent, `@n8n/n8n-nodes-langchain.agent` v3) receives each enriched transaction item. It injects the full item JSON into a user prompt and sends it, along with a detailed system prompt, to the **`OpenRouter Chat Model`** sub-node. The model (`nvidia/nemotron-nano-12b-v2-vl:free`) responds with a JSON object containing an `anomalies[]` array and a `summary{}` object.

   > ⚠️ **Pinned data active**: The `AI Anomaly Detection` node has 10 pinned output items in the workflow export (transactions TXN001–TXN010). While pinned data is present, n8n will **not** call the OpenRouter API — it will replay the pinned responses. This must be cleared before live execution. See [Known Limitations](#known-limitations).

5. **`Parse AI Response`** (Code node) iterates over the agent's output items:
   - Extracts `item.json.output` and strips any markdown code fences.
   - Parses the JSON with a try/catch guard — malformed items are silently skipped (no error thrown).
   - Normalises `severity` and `anomaly_type` to lowercase.
   - Enriches each anomaly with `detected_at` (ISO timestamp), `workflow_run_id` (`$execution.id`), `status: "open"`, and computed flags `is_high_risk` and `needs_review`.
   - Returns one item per anomaly plus one `type: "summary"` item per agent response.

6. **`Switch`** evaluates `$json.severity` using strict, case-sensitive string equality against three rules:
   - **Output 0** (`severity == "high"`) → `Gmail Alert — High Severity`
   - **Output 1** (`severity == "medium"`) → `Slack — Medium Severity`
   - **Output 2** (`severity == "low"`) → `Set Node — Low Severity`

   > ⚠️ The `type: "summary"` items emitted by Parse AI Response carry no `severity` field — they will not match any Switch rule and will be silently dropped. This is a known gap; see [Known Limitations](#known-limitations).

7. **`Gmail Alert — High Severity`** (Gmail v2.1, OAuth2): sends an HTML email to `storysnippents@gmail.com` with a red-headed branded template listing each high-severity finding's transaction ID, anomaly type, vendor, amount, employee, reason, recommendation, and fraud score.

8. **`Slack — Medium Severity`** (Slack v2.3): posts a text-formatted alert to the Slack user `USLACKBOT` (the Slackbot DM), reporting severity, count, detection timestamp, and workflow run ID. _(Note: this targets a bot DM, not a shared channel — see [Known Limitations](#known-limitations).)_

9. **`Set Node — Low Severity`** (Set v3.4): appends three fields to the low-severity item — `alert_sent: "false"`, `log_only: "true"`, `reason: "Low severity — logged for weekly review"` — and passes the item downstream without sending any external alert.

10. **`Merge`** (Merge v3.2, `numberInputs: 3`) waits for items from all three branches and combines them into a single stream.

11. **`Write Audit Log`** (Google Sheets v4.7, append operation): appends one row per incoming anomaly item to the `AuditLog` sheet (gid `1762919534`) of the same spreadsheet, mapping columns as defined in the [Node Configuration](#node-configuration) section below.

---

## Node List

| # | Node Name | Type | Role | Disabled |
|---|---|---|---|---|
| 1 | `Schedule Trigger` | Schedule Trigger | Fires workflow daily at 08:00 | No |
| 2 | `'Fetch Transactions' (Google Sheets)` | Google Sheets | Reads all rows from `Transactions` sheet | No |
| 3 | `Pre-Process & Deduplicate` | Code (JavaScript) | Rule-based dedup, budget check, policy flagging | No |
| 4 | `AI Anomaly Detection` | LangChain Agent | Sends enriched transaction to LLM for anomaly analysis | No |
| 5 | `OpenRouter Chat Model` | lmChatOpenRouter (sub-node) | Provides `nvidia/nemotron-nano-12b-v2-vl:free` as the LLM backend | No |
| 6 | `Parse AI Response` | Code (JavaScript) | Normalises and enriches LLM JSON output into discrete anomaly items | No |
| 7 | `Switch` | Switch | Routes items by `$json.severity` to three output branches | No |
| 8 | `Gmail Alert — High Severity` | Gmail | Sends HTML alert email for high-severity anomalies | No |
| 9 | `Slack — Medium Severity` | Slack | Posts text alert to Slack for medium-severity anomalies | No |
| 10 | `Set Node — Low Severity` | Set | Tags low-severity items as log-only; no external alert | No |
| 11 | `Merge` | Merge | Reunites all three severity branches into one stream | No |
| 12 | `Write Audit Log` | Google Sheets | Appends every anomaly to the `AuditLog` sheet | No |

---

## Node-by-Node Description

### `Schedule Trigger`
Fires once per day at hour 8 (08:00) using n8n's built-in schedule trigger. No credentials or authentication required. Produces a single empty item that triggers the downstream chain. The timezone used is the instance default — no override is configured in the workflow settings.

---

### `'Fetch Transactions' (Google Sheets)`
Performs a **Read** operation on the `Transactions` sheet (gid `0`) of spreadsheet `1Rt8ENolpb0rQ43qCGf-PLdIoN3ZxNve3syCyNDaGY24`. All rows are returned without any filter predicate — every row in the sheet is fetched on every run. Each row becomes one n8n item with the sheet's column headers as JSON keys. Authenticates via `googleSheetsOAuth2Api`.

---

### `Pre-Process & Deduplicate`
A Code node running in **Run Once for All Items** mode. It processes the full transaction array in a single JavaScript execution, performing four operations:

1. **Fingerprint dedup**: maps `vendor|amount|date|employee` to the first-seen `transaction_id`. Subsequent collisions populate a `duplicatesMap` that both the duplicate and the original are tagged against.
2. **Budget aggregation**: iterates all items to build `categoryTotals{}` and compares against a hard-coded `budgetLimits{}` object. Over-budget categories are stored in a `Set`.
3. **Policy violation detection**: per-transaction checks for weekend purchases outside Travel, round numbers ≥ 5000, and amounts > 10000.
4. **Return**: maps the original transactions array to one enriched item each, preserving all original fields and adding the enrichment objects.

The code does **not** produce a single aggregated output — it maintains the one-item-per-transaction cardinality required for the downstream LangChain agent to process each transaction independently.

---

### `AI Anomaly Detection`
A LangChain Agent node (`promptType: define`) that wraps each enriched transaction in a structured prompt and submits it to the connected LLM. The `hasOutputParser: true` flag activates the ToOutputParser sub-node to help enforce JSON output. The agent's `output` field in the response contains the LLM's raw text response (which may or may not include markdown fences despite the JSON-only system prompt instruction). Connected to `OpenRouter Chat Model` via the `ai_languageModel` connection type.

---

### `OpenRouter Chat Model`
A sub-node of the AI Agent, providing the LLM backend. Uses the model `nvidia/nemotron-nano-12b-v2-vl:free` — a free-tier, 12-billion-parameter multimodal model available through OpenRouter's free API tier. No custom temperature or sampling parameters are configured (defaults apply). Authenticates via `openRouterApi`.

---

### `Parse AI Response`
A Code node running in **Run Once for All Items** mode. Iterates over all agent output items, extracts `item.json.output`, strips markdown code fences via regex, and JSON-parses the content inside a try/catch block — items that fail to parse are silently skipped (no exception propagates). Valid anomaly arrays are flattened: each anomaly within an `anomalies[]` array becomes its own output item (`type: "anomaly"`), and each `summary{}` object becomes a separate output item (`type: "summary"`). Enrichment fields `detected_at`, `workflow_run_id`, `status`, `is_high_risk`, and `needs_review` are computed and attached to every anomaly item.

---

### `Switch`
Routes items based on the string value of `$json.severity`. Uses Rules mode with three outputs — `"high"`, `"medium"`, and `"low"` — all evaluated with `caseSensitive: true` and `typeValidation: "strict"`. Items that match no rule (e.g. `type: "summary"` items with no `severity` field) are dropped silently. The Parse AI Response node normalises severity to lowercase before this node runs, ensuring the case-sensitive match works correctly.

---

### `Gmail Alert — High Severity`
Sends an HTML email via the Gmail node (OAuth2) to `storysnippents@gmail.com`. The subject line is dynamic: `URGENT: {{ $json.high_severity.length }} Critical Financial Anomaly(ies) Detected`. The body is a full inline HTML document — a red-branded header block followed by a per-anomaly card section listing transaction ID, anomaly type, vendor, amount, employee, reason, recommended action, fraud score, and confidence percentage. Authenticates via `gmailOAuth2`.

---

### `Slack — Medium Severity`
Posts a plain-text message to the Slack user `USLACKBOT` (the Slackbot) via the Slack node (API token). Message includes severity label, anomaly count derived from `$json.severity.length`, a formatted detection timestamp using `$now.toFormat('dd MMM HH:mm')`, and the `workflow_run_id`. Authenticates via `slackApi`.

> ⚠️ The current target (`USLACKBOT`) is the Slackbot personal DM, not a shared team channel. Alerts will not be visible to the broader team unless the target is changed to a real channel ID.

---

### `Set Node — Low Severity`
Attaches three static string fields to each passing item: `alert_sent: "false"`, `log_only: "true"`, and `reason: "Low severity — logged for weekly review"`. Does not send any notification. All original item fields are preserved alongside the new fields (Set v3.4 default). Items then flow to the Merge node.

---

### `Merge`
Configured with `numberInputs: 3`, this node collects items arriving from the Gmail branch (Input 0), the Slack branch (Input 1), and the Set Node branch (Input 2), then passes them all downstream as a unified stream. Uses the default Append merge behaviour — items from all branches are concatenated, not combined by key.

---

### `Write Audit Log`
Performs an **Append** operation on the `AuditLog` sheet (gid `1762919534`) of the same spreadsheet. Uses `mappingMode: defineBelow` with explicit column-to-expression mappings. The `valueInputOption` is not set (defaults to `USER_ENTERED`). Authenticates via the same `googleSheetsOAuth2Api` credential as the Fetch node.

---

## Node Configuration

### `'Fetch Transactions' (Google Sheets)` — Key Parameters

| Parameter | Value |
|---|---|
| Spreadsheet | `AI Financial Anomaly Detection & Expense Reconciliation` |
| Spreadsheet ID | `1Rt8ENolpb0rQ43qCGf-PLdIoN3ZxNve3syCyNDaGY24` |
| Sheet | `Transactions` (gid `0`) |
| Operation | Read (default — all rows) |
| Filters | None |

---

### `Pre-Process & Deduplicate` — Budget Limits (Hard-Coded)

```javascript
const budgetLimits = {
  "Cloud Infrastructure": 10000,
  "Travel": 5000,
  "Meals & Entertainment": 2000,
  "Software": 3000,
  "Office Supplies": 1000,
  "Marketing": 8000
};
```

Policy violation thresholds:

| Rule | Threshold |
|---|---|
| Round-number flag | Amount ≥ 5,000 AND divisible by 1,000 |
| High-value flag | Amount > 10,000 |
| Weekend flag | Day-of-week is Saturday (6) or Sunday (0), and category is not `"Travel"` |

---

### `AI Anomaly Detection` — System Prompt (summary)

The system prompt instructs the model to act as a senior financial fraud detection expert with 15 years of Big 4 experience, to return **only** valid JSON (no markdown, no preamble), and to follow a strict output schema:

```json
{
  "anomalies": [{
    "transaction_id", "anomaly_type", "severity", "fraud_score",
    "confidence", "reason", "recommendation", "related_transactions",
    "amount", "vendor", "employee", "weekend_flag", "round_number_flag"
  }],
  "summary": {
    "total_anomalies", "high", "medium", "low",
    "total_suspicious_amount", "top_risk_vendor",
    "top_risk_employee", "overall_risk_level"
  }
}
```

Detection rules encoded in the prompt:

| Rule | Definition |
|---|---|
| **DUPLICATES** | Same vendor + amount + employee within 48 hours → high severity |
| **FRAUD PATTERNS** | Round numbers > ₹5000 with no receipt; unusual-hour transactions; 1-character vendor name typos; split transactions below approval threshold |
| **BUDGET OVERRUNS** | Category total exceeds budget from `budget_alerts` field → minimum medium severity |
| **POLICY VIOLATIONS** | Weekend non-travel purchase; single vendor > 40% of department spend; amount > ₹10,000 |
| **SUSPICIOUS PATTERNS** | 3+ transactions in one day same employee; new vendor with first transaction > ₹8,000 |

Severity thresholds: `HIGH` = confirmed dup / likely fraud / amount > ₹10k | `MEDIUM` = policy violation / budget overrun > 20% | `LOW` = minor deviation / budget overrun < 20%.

---

### `AI Anomaly Detection` — User Prompt Template

```
Analyze the following corporate transaction data and return your findings as JSON.

DATASET:
{{ JSON.stringify($json, null, 2) }}

Cross-reference the pre_detected_duplicates and policy_violations with your own analysis.
Confirm, enrich, or correct them with your reasoning.
Add any additional anomalies you detect that the rule engine missed.
```

---

### `Write Audit Log` — Column Mapping

| AuditLog Column | Expression |
|---|---|
| `transaction_id` | `={{ $('Parse AI Response').item.json.transaction_id }}` |
| `anomaly_type` | `={{ $('Parse AI Response').item.json.anomaly_type }}` |
| `severity` | `={{ $('Parse AI Response').item.json.severity }}` |
| `fraud_score` | `={{ $('Parse AI Response').item.json.fraud_score }}` |
| `reason` | `={{ $('Parse AI Response').item.json.reason }}` |
| `recommendation` | `={{ $('Parse AI Response').item.json.recommendation }}` |
| `workflow_run_id` | `={{ $('Parse AI Response').item.json.workflow_run_id }}` |
| `detected_at` | `={{ $('Parse AI Response').item.json.detected_at }}` |
| `alert_sent` | `={{ $('Parse AI Response').item.json.weekend_flag }}` ⚠️ **Bug: mapped to `weekend_flag` instead of `alert_sent`** |

---

## Branching Logic

The **`Switch`** node implements a three-way severity router.

| Output | Condition | Destination | Behaviour |
|---|---|---|---|
| 0 | `$json.severity === "high"` (strict, case-sensitive) | `Gmail Alert — High Severity` | HTML email sent immediately |
| 1 | `$json.severity === "medium"` (strict, case-sensitive) | `Slack — Medium Severity` | Slack DM to Slackbot |
| 2 | `$json.severity === "low"` (strict, case-sensitive) | `Set Node — Low Severity` | No alert; log-only flag set |
| _(none)_ | Items with no `severity` field (e.g. `type: "summary"` items) | _(dropped)_ | Silently discarded |

The `Parse AI Response` node normalises severity strings to lowercase before items reach the Switch, ensuring `"HIGH"`, `"Medium"`, or `"MEDIUM"` from the LLM are correctly matched.

---

## Variables & Expressions

Notable dynamic expressions used across the workflow:

| Location | Expression | Purpose |
|---|---|---|
| AI Anomaly Detection — user prompt | `{{ JSON.stringify($json, null, 2) }}` | Serialises the full enriched transaction item as the LLM user input |
| Gmail Alert — subject | `{{ $json.high_severity.length }}` | Count of high-severity items in subject line |
| Gmail Alert — body | `{{ $now.toFormat('dd MMM yyyy HH:mm') }}` | Human-readable detection timestamp in email header |
| Gmail Alert — body | `$json.high_severity.map(a => ...)` | Generates per-anomaly HTML cards by iterating the anomaly array |
| Slack — text | `{{ $json.severity.length }}` | ⚠️ **Bug**: `$json.severity` is a string (e.g. `"medium"`), not an array — `.length` returns the character count of the string, not the anomaly count |
| Slack — text | `{{ $now.toFormat('dd MMM HH:mm') }}` | Formatted detection timestamp |
| Slack — text | `{{ $json.workflow_run_id }}` | n8n execution ID for traceability |
| Write Audit Log — all columns | `$('Parse AI Response').item.json.*` | Back-references the Parse AI Response output by node name |

---

## Credentials Used

| Node | Credential Name | Credential Type | Service |
|---|---|---|---|
| `'Fetch Transactions' (Google Sheets)` | Google Sheets account | `googleSheetsOAuth2Api` | Google Sheets |
| `Write Audit Log` | Google Sheets account | `googleSheetsOAuth2Api` | Google Sheets |
| `OpenRouter Chat Model` | n8n-regular | `openRouterApi` | OpenRouter (LLM gateway) |
| `Gmail Alert — High Severity` | Gmail account | `gmailOAuth2` | Gmail |
| `Slack — Medium Severity` | Slack account | `slackApi` | Slack |

All credentials are referenced by name and internal ID only — no secret values are present in the workflow export.

---

## Input Data

The workflow ingests data from the `Transactions` sheet. Expected column schema:

| Column | Type | Notes |
|---|---|---|
| `transaction_id` | String | Primary key, e.g. `TXN001` |
| `date` | Date string | `YYYY-MM-DD` — used for weekend detection via `new Date(tx.date).getDay()` |
| `vendor` | String | Used in fingerprint dedup |
| `amount` | Number | No currency symbol — used for budget and round-number checks |
| `category` | String | Must match budget limit keys exactly for budget comparisons |
| `employee` | String | Full name — used in fingerprint dedup |
| `department` | String | Referenced in AI prompt context |
| `receipt_attached` | Boolean | Referenced by the AI prompt |
| `notes` | String | Optional — passed to AI for context |

---

## Output Data

The workflow produces two output side-effects:

**1. Alert notifications** (conditional on severity):
- **Gmail** (high): HTML email to `storysnippents@gmail.com`
- **Slack** (medium): text DM to `USLACKBOT`
- **No alert** (low): item passes through silently

**2. AuditLog rows** (every anomaly, all severities):

| AuditLog Column | Description |
|---|---|
| `transaction_id` | Source transaction |
| `anomaly_type` | e.g. `duplicate`, `policy_violation`, `budget_overrun` |
| `severity` | `low`, `medium`, or `high` |
| `fraud_score` | 0–100 integer |
| `reason` | AI-generated explanation |
| `recommendation` | AI-generated action |
| `workflow_run_id` | n8n execution ID for cross-referencing |
| `detected_at` | ISO 8601 timestamp of detection |
| `alert_sent` | ⚠️ Currently records `weekend_flag` value — see Known Limitations |

---

## Data Transformation

The workflow performs significant data enrichment across two Code nodes:

**Pre-Process & Deduplicate** adds to each transaction:
- `duplicate_info{}` — whether the transaction is a fingerprint duplicate and which original it duplicates
- `category_total` — running sum of spend in the same category across today's transactions
- `category_budget` — the configured limit for the category
- `is_over_budget` — boolean
- `violations[]` — array of string codes for policy rules triggered
- `is_high_risk` — true if any violations exist or if a duplicate was detected
- `total_transactions` / `total_spend` — global context values attached to every item

**Parse AI Response** transforms the LLM's raw text output into normalised structured items:
- Strips markdown fences; parses JSON
- Splits each `anomalies[]` array element into its own n8n item
- Normalises string casing on `severity` and `anomaly_type`
- Adds `detected_at`, `workflow_run_id`, `status: "open"`, `is_high_risk`, `needs_review`

---

## API Integrations / External Services

| Service | Node | Endpoint / Method | Auth | Purpose |
|---|---|---|---|---|
| Google Sheets API | `'Fetch Transactions' (Google Sheets)` | Sheets Read (all rows) | OAuth2 | Fetch transaction input data |
| OpenRouter API | `OpenRouter Chat Model` | `POST https://openrouter.ai/api/v1/chat/completions` (via n8n integration) | API Key | LLM inference — `nvidia/nemotron-nano-12b-v2-vl:free` |
| Gmail API | `Gmail Alert — High Severity` | `gmail.users.messages.send` | OAuth2 | Send high-severity alert emails |
| Slack API | `Slack — Medium Severity` | `chat.postMessage` (to user `USLACKBOT`) | Bot Token | Post medium-severity Slack alerts |
| Google Sheets API | `Write Audit Log` | Sheets Append | OAuth2 | Write anomaly rows to AuditLog |

---

## AI/LLM Integration

**Model**: `nvidia/nemotron-nano-12b-v2-vl:free`
**Provider**: OpenRouter (free tier — no usage cost, no credit card required)
**Node type**: `@n8n/n8n-nodes-langchain.agent` (LangChain Agent) with `@n8n/n8n-nodes-langchain.lmChatOpenRouter` sub-node
**Prompt mode**: `define` (prompt written directly in node parameters, not from input)

The agent is given a detailed system prompt encoding five fraud detection rule families (duplicates, fraud patterns, budget overruns, policy violations, and suspicious behavioural patterns) and a strict JSON-only output schema. The user prompt serialises the full enriched transaction JSON via `JSON.stringify($json, null, 2)`.

The agent's output appears in `item.json.output` as a raw string. The downstream `Parse AI Response` Code node handles JSON extraction, markdown stripping, and error recovery.

**Pinned test data**: The workflow JSON contains 10 pinned responses for TXN001–TXN010 covering real AI output for: budget overruns, policy violations (weekend purchases by Zomato, Swiggy, Flipkart, Adobe), a duplicate detection (TXN006/TXN007 — Amazon Business, Rahul Sharma), and a suspicious merchant name flag (MakeMyTrip). These responses demonstrate the exact JSON format the real model produces but bypass the live API call. **Remove `pinData` from the JSON export before production use.**

---

## Error Handling

| Aspect | Configuration |
|---|---|
| `errorWorkflow` in settings | Not configured — no dedicated error-handling workflow |
| `continueOnFail` on any node | Not set on any node (defaults to `false` — a node failure will halt the execution) |
| Code node error recovery | `Parse AI Response` wraps JSON parsing in a `try/catch` and silently skips malformed LLM responses (no error item emitted) |
| LLM output failures | If the LLM returns non-JSON text (e.g. rate limit HTML), the Code node's try/catch will swallow it — no alert will be sent and no audit row will be written |
| Merge with missing branches | If a branch (e.g. no high-severity items) produces no items, the Merge node will still pass through items from the other branches — behaviour is correct |

---

## Retry Strategy

No retry logic is configured on any node. HTTP/API nodes (`Gmail`, `Slack`, `OpenRouter`) do not have `retryOnFail`, `maxTries`, or `waitBetweenTries` set. A transient API failure (rate limit, network error, OAuth token expiry) will cause the entire execution to fail at that node with no automatic retry.

---

## Scheduling Details

| Field | Value |
|---|---|
| Type | Interval-based schedule |
| Trigger hour | `8` (08:00) |
| Frequency | Daily (every day — no day-of-week filter) |
| Timezone | Not set in `settings` — inherits from n8n instance configuration |
| Cron equivalent | `0 8 * * *` |

The workflow has **no built-in deduplication guard** to prevent re-processing the same transaction rows if they persist in the sheet across runs. Every row in the `Transactions` sheet is read and sent to the AI on every execution.

---

## Execution Order

`settings.executionOrder` is set to **`"v1"`** — n8n's connection-based execution order where nodes execute in the order determined by their connections, not their canvas position. This is the recommended setting for workflows created in n8n v1.x and later.

---

## Security Considerations

- **No webhook trigger** — this workflow is schedule-driven, so there is no inbound HTTP endpoint to secure.
- **Credentials**: four separate credential sets are used (Google Sheets OAuth2, Gmail OAuth2, Slack Bot Token, OpenRouter API key). All are stored in n8n's encrypted credential store and not exposed in the workflow export.
- **Recipient email hardcoded**: `storysnippents@gmail.com` is embedded directly in the Gmail node parameters. Changing the alert recipient requires editing the node, not a credential or variable.
- **Google Spreadsheet access**: the OAuth2 account must have at least Editor access to the spreadsheet. Over-permissioning to the whole Google Drive should be avoided.
- **LLM data exposure**: the full transaction JSON (including vendor names, employee names, and amounts) is sent to OpenRouter's API, which forwards it to NVIDIA's Nemotron model infrastructure. Verify this is acceptable under your organisation's data classification policy before production deployment.
- _Further security review:_ _[To be provided by workflow owner / security reviewer]_

---

## Performance Considerations

- **No batching**: all transactions are read in one Sheets API call and processed individually through the LangChain agent. For large transaction volumes, this may result in many sequential LLM API calls.
- **OpenRouter free tier rate limits**: the `nvidia/nemotron-nano-12b-v2-vl:free` model is subject to OpenRouter's free-tier rate limits. No wait/throttle node is present. Under high transaction volume, the agent may receive 429 (rate limit) errors that halt the workflow.
- **No pagination on Sheets read**: if the Transactions sheet grows to thousands of rows, the Google Sheets node will attempt to read them all in a single request.

---

## Execution Time

_[To be provided by workflow owner — requires n8n execution log data]_

---

## Resource Usage

_[To be provided by workflow owner — requires execution data]_

---

## Testing & Validation

The workflow includes **10 pinned AI outputs** (TXN001–TXN010) in the exported JSON, representing real LLM responses captured during development testing. These cover the following test scenarios:

| Transaction | Anomaly Detected | Severity | Fraud Score |
|---|---|---|---|
| TXN002 | Budget overrun — Travel category | Medium | 45 |
| TXN003 | Low-severity budget overrun — Meals | Low | 12 |
| TXN004 | Policy violation — weekend Equipment purchase (Flipkart, Sneha Joshi) | Low | 30 |
| TXN005 | Policy violation — weekend Meals purchase (Zomato, Rahul Sharma) | Medium | 75 |
| TXN006 | Duplicate transaction + weekend Office Supplies (Amazon Business, Rahul Sharma) | Medium | 65 / 75 |
| TXN007 | Suspicious merchant name (MakeMyTrip — low flag) | Low | 22 |
| TXN008 | Policy violation — weekend Meals purchase (Swiggy, Amit Patel) | Medium | 35 |
| TXN009 | Weekend transaction — Software category (Adobe, Sneha Joshi) | Medium | 35 |
| TXN010 | Policy violation — weekend Equipment + single-vendor concentration (Local Electronics) | Medium | 70 |

End-to-end live test results with real Google Sheets data and live OpenRouter API calls: _[To be provided by workflow owner]_

---

## Troubleshooting

| Symptom | Likely Cause | Resolution |
|---|---|---|
| Workflow never runs automatically | `active: false` in export | Toggle the **Active** switch ON in the n8n workflow editor |
| AI node always returns TXN001–TXN010 data | `pinData` is present in the workflow JSON | Open the AI Anomaly Detection node, right-click the pinned output, and clear it; or remove `pinData` from the raw JSON export |
| No rows written to AuditLog | Parse AI Response produces only `type: "summary"` items with no `severity` field; Switch drops them all | Verify the LLM is returning anomalies. Check the execution log at the Parse AI Response node to see what was parsed |
| Slack message shows wrong anomaly count | `$json.severity.length` counts string characters, not anomaly count | Fix the Slack message expression to use the actual count (e.g. pass anomaly count explicitly from Parse AI Response or use `$json.total_anomalies`) |
| `alert_sent` column in AuditLog shows `true`/`false` for weekend flag | Column is mapped to `weekend_flag` instead of `alert_sent` | Update the Write Audit Log column mapping for `alert_sent` to `={{ $json.alert_sent }}` |
| OpenRouter returns a 429 error | Free-tier rate limit hit when processing many transactions in quick succession | Add a **Wait** node (5–10 seconds) between Pre-Process & Deduplicate and AI Anomaly Detection |
| Gmail auth error (401) | OAuth2 token expired | Re-authenticate the Gmail credential in **Settings → Credentials → Gmail account** |
| Slack alert not visible to team | Target is `USLACKBOT` (personal bot DM) | Change the Slack node's `user` field to a real channel ID (e.g. `C08XXXXXXXX`) |
| New transaction categories not flagged for budget | Category name not in `budgetLimits` in the Code node | Edit the `Pre-Process & Deduplicate` Code node to add the new category and its limit |

---

## Known Limitations

1. **Workflow is currently inactive** — `"active": false` in the export. The daily schedule will not fire until this is explicitly enabled.

2. **Pinned data bypasses live AI** — The `pinData` block for `AI Anomaly Detection` covers 10 items (TXN001–TXN010). While present, the workflow replays these static responses instead of calling OpenRouter. Remove before production.

3. **`alert_sent` column mapping is incorrect** — The Write Audit Log node maps `alert_sent` to `$('Parse AI Response').item.json.weekend_flag`. The `alert_sent` column will contain the boolean weekend flag value, not whether an alert was dispatched.

4. **Slack anomaly count expression is incorrect** — `$json.severity.length` in the Slack message counts the number of characters in the string `"medium"` (6), not the number of anomalies at that severity level.

5. **Summary items dropped at Switch** — `Parse AI Response` emits `type: "summary"` items with no `severity` field. These do not match any Switch rule and are silently discarded — they never reach the Audit Log.

6. **No row deduplication across runs** — The Transactions sheet is read in full on every daily run. If rows are not removed or marked as processed after each run, previously analysed transactions will be re-sent to the AI on subsequent runs, generating duplicate AuditLog entries.

7. **No retry logic** — Any transient failure (OpenRouter rate limit, Gmail token expiry, Sheets API outage) halts the workflow permanently for that execution with no automatic retry or operator notification.

8. **Slack target is bot DM, not a team channel** — Alerts go to `USLACKBOT` and are not visible to human team members.

9. **Budget limits and policy thresholds are hard-coded** — Changes to budget limits or policy rules require direct edits to the `Pre-Process & Deduplicate` Code node. There is no external configuration sheet or environment variable.

10. **All transactions re-processed every run** — There is no read-cursor, date filter, or processed-flag mechanism to limit the Transactions sheet read to only new rows since the last run.

---

## Assumptions

- The `Transactions` sheet columns are named exactly as expected by the Code node (`transaction_id`, `date`, `vendor`, `amount`, `category`, `employee`). Misspelled or missing columns will produce `undefined` values in fingerprints and budget calculations without throwing an error.
- `amount` values in the Transactions sheet are plain numbers (no currency symbol). The Code node casts them with `Number(tx.amount)` — non-numeric strings will produce `NaN` and silently fail budget comparisons.
- The `date` column contains values parseable by `new Date(tx.date)` — ISO `YYYY-MM-DD` strings work; locale-specific formats (e.g. `DD/MM/YYYY`) will produce `Invalid Date` and incorrect weekend detection.
- The n8n instance timezone is configured to the desired local timezone (e.g. IST). The Schedule Trigger fires at hour 8 in the instance timezone — the document cannot confirm this without access to the instance settings.
- OpenRouter's free tier is sufficient for the transaction volume this workflow processes. High-volume organisations (hundreds of transactions per day) may hit rate limits.
- The `AuditLog` sheet exists in the spreadsheet with column headers matching the Write Audit Log mapping. If the sheet or headers are missing, the append operation will fail.

---

## Version History / Change Log

_[To be provided by workflow owner]_

---

## References

- OpenRouter API docs: https://openrouter.ai/docs
- OpenRouter free models: https://openrouter.ai/models?q=free
- n8n LangChain Agent node docs: https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent/
- n8n Google Sheets node docs: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlesheets/
- n8n Schedule Trigger docs: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.scheduletrigger/
- _Additional references: [To be provided by workflow owner]_

---

## Maintenance Notes

_[To be provided by workflow owner]_

Suggested maintenance tasks based on workflow structure:

- **Monthly**: Review AuditLog sheet for unresolved anomalies; purge or archive processed Transactions rows to prevent re-analysis on subsequent runs.
- **Quarterly**: Review hard-coded `budgetLimits` in the `Pre-Process & Deduplicate` Code node against actual approved budgets.
- **On OAuth2 expiry**: Re-authenticate `Google Sheets account` and `Gmail account` credentials in n8n Credentials panel.
- **On OpenRouter model deprecation**: Update the model ID in the `OpenRouter Chat Model` sub-node if `nvidia/nemotron-nano-12b-v2-vl:free` is retired from the free tier.
