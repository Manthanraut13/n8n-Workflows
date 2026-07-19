# AI Churn Prediction & Customer Retention Engine — Workflow Documentation

| Field | Value |
|---|---|
| **Workflow Name** | AI Churn Prediction & Customer Retention Engine |
| **Workflow ID** | ZQx9F60T9pzbme62 |
| **Internal Version ID** | d74824dd-b842-49ac-9fc7-d017f7958f23 |
| **Version** | v1.0 (initial documentation) |
| **Owner** | _[To be provided by workflow owner]_ |
| **Workflow Last Saved** | _[Not present in export — check n8n instance]_ |
| **Document Date** | July 18, 2026 |
| **Status** | Inactive (not yet activated in n8n) |

---

## Purpose

This workflow automates the daily identification of SaaS customers who are at risk of churning. It ingests customer and support-ticket data from Google Sheets, computes four weighted engagement signals per customer, feeds a structured prompt to an AI language model via OpenRouter, classifies each customer into a Low / Medium / High risk tier, and then — for customers in the Medium or High tier — sends a personalised HTML retention email via Gmail and logs the action to a tracking sheet. All predictions are persisted to a dedicated Google Sheets tab for historical analysis and reporting.

---

## Business Use Case

_Inferred from workflow logic — please confirm or replace._

SaaS companies typically lose 10–30 % of customers annually due to undetected churn. This workflow aims to surface at-risk customers proactively — before they cancel — so the customer-success team can intervene with targeted outreach, offers, and support follow-up. The automated daily run eliminates the need for manual cohort analysis and ensures no high-risk account is missed.

---

## Workflow Summary

The workflow is triggered once a day at 09:00 via a Schedule Trigger. It fetches all rows from the **Customers** sheet and all rows from the **Support\_Tickets** sheet in parallel, then feeds both into an n8n Merge node before a Code node joins them — grouping tickets by `customer_id` and computing derived temporal fields (days since last login, days since last payment, tenure in months). A second Code node validates every merged record for data quality, and a third computes four engagement scores (login recency, usage consistency, support frustration, payment reliability) plus a weighted composite. Each validated customer profile is then passed, one item at a time, to an LangChain AI Agent node backed by the **OpenRouter Chat Model** (`stepfun/step-3.5-flash:free`), which returns a structured JSON churn assessment. A Code node parses and normalises that JSON; a further Code node assigns a risk tier (Low ≤ 40, Medium 41–70, High ≥ 71). An IF node gates subsequent actions: customers scoring Medium or High receive a retention email via Gmail and have an action row appended to the **Retention\_Actions** sheet, while all customers (regardless of tier) have a prediction row appended to the **Daily\_Predictions** sheet. Finally, a summary Code node aggregates run-level metrics (total analysed, counts by tier, duration).

---

## Workflow Diagram

_See the attached canvas screenshot (`AI_Churn_Prediction___Customer_Retention_Engine.webp`)._

The workflow has a two-branch fan-in for data collection (Customers + Support Tickets → Merge), a long linear processing spine (Merge Data → Validate → Score → AI → Parse → Classify), a single conditional gate (High-Risk Split IF node), and two terminal fan-out paths from that gate: the "true" branch drives email sending and action logging; the "false" branch (and also the "true" branch) both converge on prediction logging. The Execution Summary node sits at the far right as the final terminal.

```
Schedule Trigger
    ├─► Fetch Customers ──────────────────────────────┐
    └─► Fetch Support Tickets ──────────────────────┐  │
                                                    ↓  ↓
                                                  Merge
                                                    ↓
                                          Merge Data (Code Node)
                                                    ↓
                                          Validate Data (Code Node)
                                                    ↓
                                    Calculate Engagement Scores (Code Node)
                                                    ↓
                                          AI Churn Scoring (Agent)
                                                    ↓
                                        Parse AI Response (Code Node)
                                                    ↓
                                         Classify Risk (Code Node)
                                                    ↓
                                           High-Risk Split (IF)
                                          /              \
                               true (medium|high)      false (low)
                                    /                         \
                    Send Retention Email              Append Predictions
                           ↓                               ↓
                   Append Predictions ←──────────────────────
                           ↓
                   Append Actions (Google Sheets)
                           ↓
                   Execution Summary (Code Node)
```

> **Note:** Both the `true` and `false` outputs of High-Risk Split feed into **Append Predictions**. Only the `true` path additionally triggers the retention email and action log before prediction logging.

---

## Trigger Details

| Field | Value |
|---|---|
| **Node Name** | Schedule Trigger |
| **Node Type** | `n8n-nodes-base.scheduleTrigger` (v1.3) |
| **Schedule** | Every day at **09:00** (hour = 9, no minute override → defaults to :00) |
| **Timezone** | Not explicitly set in workflow settings — defaults to the n8n instance timezone. Recommend setting `settings.timezone` explicitly to avoid daylight-saving ambiguity. |
| **Initiator** | Time-based; no external event or webhook |

---

## Execution Flow

1. **Schedule Trigger** fires at 09:00 each day.
2. Two Google Sheets reads execute in parallel:
   - **Fetch Customers** reads every row from the `Customers` sheet.
   - **Fetch Support Tickets** reads every row from the `Support_Tickets` sheet.
3. The **Merge** node (append mode) combines the two arrays into a single item stream containing both customer rows and ticket rows.
4. **Merge Data (Code Node)** separates the combined stream back into customers and tickets, joins them on `customer_id`, and computes `days_since_login`, `days_since_payment`, `tenure_months`, `negative_tickets`, and `avg_resolution_time` per customer. Each customer is emitted as one item.
5. **Validate Data (Code Node)** runs quality checks on each customer item (required fields, date parsability, email format, plan/payment-status enumerations) and attaches a `validation` object with `status`, `errors`, and `warnings`.
6. **Calculate Engagement Scores (Code Node)** computes four sub-scores and a weighted composite for each customer. Items that failed validation receive zeroed scores and are flagged `analysis_skipped: true`.
7. **AI Churn Scoring** (LangChain Agent, one item at a time) sends a structured prompt with the customer profile and scores to OpenRouter. The model returns a JSON object containing `churn_risk_score`, `risk_level`, `churn_signals`, `root_causes`, `recommended_retention_action`, and `confidence_score`.
8. **Parse AI Response (Code Node)** extracts the JSON block from the model output, normalises numeric fields to valid ranges (score 0–100, confidence 0–1), ensures arrays are arrays, and attaches `parsing_status: "success"` or `"failed"`.
9. **Classify Risk (Code Node)** reads `churn_risk_score` and attaches a `risk_classification` object with `tier` (low / medium / high), `urgency` (monitor / engage / intervene), `action_required` (boolean), and `classified_at` timestamp.
10. **High-Risk Split (IF node)** evaluates `risk_classification.tier`. The **true** branch fires if tier equals `"medium"` OR `"high"` (case-sensitive, strict typing). The **false** branch fires for `"low"`.
    - **True path:** → Send Retention Email → Append Predictions → Append Actions → Execution Summary
    - **False path:** → Append Predictions → Execution Summary
11. **Send Retention Email (Email Node)** sends an HTML email via Gmail to the customer's email address for medium/high-risk customers.
12. **Append Predictions (Google Sheets)** appends one row per customer to the `Daily_Predictions` sheet, recording the timestamp, customer identifiers, plan type, AI scores, risk level, and top signals.
13. **Append Actions (Google Sheets)** (only reached via true path) appends one row per actioned customer to `Retention_Actions`, recording the email action with a follow-up date of +1 day and `outcome_status: "pending"`.
14. **Execution Summary (Code Node)** aggregates the run: total items, counts by risk tier, duration in seconds, and a `churn_risk_rate` percentage.

---

## Node List

| # | Node Name | Node Type | Role / Purpose | Disabled |
|---|---|---|---|---|
| 1 | Schedule Trigger | Schedule Trigger | Entry point — daily 09:00 trigger | No |
| 2 | Fetch Customers (Google Sheets) | Google Sheets (read) | Read all rows from the `Customers` sheet | No |
| 3 | Fetch Support Tickets (Google Sheets) | Google Sheets (read) | Read all rows from the `Support_Tickets` sheet | No |
| 4 | Merge | Merge (append) | Combine customer rows and ticket rows into one stream | No |
| 5 | Merge Data (Code Node) | Code (JS) | Separate, join, and enrich customer + ticket data | No |
| 6 | Validate Data (Code Node) | Code (JS) | Data quality checks; attach validation status | No |
| 7 | Calculate Engagement Scores (Code Node) | Code (JS) | Compute LRS, UCS, SFS, PRS, and composite score | No |
| 8 | AI Churn Scoring | LangChain Agent | LLM-powered churn risk assessment per customer | No |
| 9 | OpenRouter Chat Model | LangChain LM (OpenRouter) | AI language model sub-node powering node 8 | No |
| 10 | Parse AI Response (Code Node) | Code (JS) | Parse, normalise, and validate LLM JSON output | No |
| 11 | Classify Risk (Code Node) | Code (JS) | Assign tier (low/medium/high) and urgency label | No |
| 12 | High-Risk Split | IF | Gate: route medium/high-risk customers to email | No |
| 13 | Send Retention Email (Email Node) | Gmail | Send personalised HTML retention email | No |
| 14 | Append Predictions (Google Sheets) | Google Sheets (append) | Persist prediction per customer to `Daily_Predictions` | No |
| 15 | Append Actions (Google Sheets) | Google Sheets (append) | Log email action to `Retention_Actions` | No |
| 16 | Execution Summary (Code Node) | Code (JS) | Aggregate and output run-level metrics | No |

---

## Node-by-Node Description

### Schedule Trigger
Fires once per day at 09:00 using n8n's built-in scheduler. Emits a single item with execution metadata, which fans out to both Google Sheets read nodes simultaneously. No credentials required.

---

### Fetch Customers (Google Sheets)
Reads all rows from the `Customers` tab (gid=0) of the shared Google Spreadsheet (`1zgRuVmJnCtl1Kks-ym6PGurgcO5KCT3lb453fiaL3o8`) using OAuth2. Each row becomes one n8n item. The sheet is expected to contain columns: `customer_id`, `email`, `company_name`, `plan_type`, `signup_date`, `last_login_date`, `monthly_usage`, `support_tickets`, `last_payment_date`, `payment_status`.

---

### Fetch Support Tickets (Google Sheets)
Reads all rows from the `Support_Tickets` tab (gid=400627009) of the same spreadsheet. Each row is one ticket record. Expected columns: `customer_id`, `ticket_id`, `created_date`, `sentiment`, `resolution_time_hours`, `category`. Runs in parallel with **Fetch Customers**.

---

### Merge
An n8n Merge node in **append** mode. Accepts customers on input 1 and tickets on input 2, and emits all rows as a single combined item stream (customers first, then tickets) for the downstream Code node to separate. No parameters are configured on this node.

---

### Merge Data (Code Node)
The most complex transformation node in the workflow. It:
1. Splits the combined stream into customer rows (those with `customer_id` AND `email`) and ticket rows (those with `ticket_id`).
2. Builds a `Map<customer_id → ticket[]>` lookup.
3. For each customer, joins its ticket array; computes `days_since_login`, `days_since_payment`, `tenure_months` from `Date.now()`; counts `negative_tickets`; and averages `resolution_time_hours`.
4. Emits one enriched item per customer.
The node wraps everything in a `try/catch` and re-throws errors with the original message, so failures surface visibly in n8n's execution log.

**Output fields added:** `days_since_login`, `days_since_payment`, `tenure_months`, `total_tickets`, `negative_tickets`, `avg_resolution_time`, `recent_tickets` (last 5 tickets).

---

### Validate Data (Code Node)
Iterates every customer item and runs the following checks:

| Check | Type | Fields |
|---|---|---|
| Required fields | Error | `customer_id`, `email`, `plan_type`, `payment_status` |
| Date parseability | Error | `last_login_date`, `last_payment_date` |
| Email format | Error | `email` (regex `/^[^\s@]+@[^\s@]+\.[^\s@]+$/`) |
| Valid plan type | Warning | must be one of `Free`, `Pro`, `Enterprise` |
| Valid payment status | Warning | must be one of `active`, `failed`, `pending`, `expired` |
| Non-negative days | Warning | `days_since_login >= 0` |

Each item is augmented with a `validation` object: `{ status: "passed"|"failed", errors: [], warnings: [], error_count, warning_count }`. All items (including failed ones) continue downstream — failed items have their engagement scores zeroed in the next node. No items are dropped here.

---

### Calculate Engagement Scores (Code Node)
Computes four sub-scores and one weighted composite for each customer that passed validation. Items flagged `validation.status === "failed"` receive all-zero scores and `analysis_skipped: true`.

**Sub-score formulas:**

| Score | Abbrev. | Weight | Key Inputs | Formula Summary |
|---|---|---|---|---|
| Login Recency Score | LRS | 35 % | `days_since_login` | Piecewise linear decay: 100 at 0 days → 10 floor at 90+ days |
| Usage Consistency Score | UCS | 30 % | `monthly_usage`, `activity_score`, `feature_usage_count` | Weighted average of three normalised components (40/40/20 split); `monthly_usage` normalised to 5,000 baseline; features normalised to 15 |
| Support Frustration Score | SFS | 20 % | `negative_tickets`, `total_tickets`, `avg_resolution_time` | 75 if no tickets; else `(1 − negativity_ratio) × 100 − resolution_penalty` (penalty capped at 40); floor 10 |
| Payment Reliability Score | PRS | 15 % | `payment_status`, `days_since_payment`, `plan_type` | Base from status map (active=100, pending=60, failed=20, expired=10); deductions after 45 overdue days; +5 bonus for Enterprise; floor 10 |

**Composite Engagement Score (CES):**
```
CES = LRS × 0.35 + UCS × 0.30 + SFS × 0.20 + PRS × 0.15
```

**Additional boolean flags added per customer:**

| Flag | Condition |
|---|---|
| `inactive_days_threshold_exceeded` | `days_since_login > 30` |
| `low_usage_pattern` | `monthly_usage < 1000` |
| `high_support_issues` | `negative_tickets > 2` |
| `payment_problems` | `payment_status !== "active"` |

---

### AI Churn Scoring
A LangChain **Agent** node (typeVersion 3) with a defined prompt. It processes one customer item at a time and calls the **OpenRouter Chat Model** sub-node. The model is instructed to return only a valid JSON object — no markdown, no prose.

**System prompt summary:** Instructs the model to act as a SaaS churn-prediction analyst; defines a scoring rubric with base scores by activity level and named modifiers (payment issues +20-30, high support frustration +10-20, Enterprise plan −10, long tenure −5, high activity score −15); specifies the three risk level bands; mandates the exact JSON output schema.

**User prompt (dynamic per customer):** Sends customer ID, company, plan, tenure, composite engagement score, all four sub-scores, raw support metrics, payment status, days since payment, and a `{recent_tickets_summary}` placeholder (note: this placeholder is not currently interpolated — see Known Limitations).

**Expected JSON output:**
```json
{
  "churn_risk_score": 0–100,
  "risk_level": "low" | "medium" | "high",
  "churn_signals": ["...", "...", "..."],
  "root_causes": ["...", "..."],
  "recommended_retention_action": "...",
  "confidence_score": 0–1
}
```

The node has pinned test data (20 sample AI responses) embedded in the JSON export, which means in test/pinned mode the model is not actually called — the pinned outputs are replayed instead.

---

### OpenRouter Chat Model
A LangChain `lmChatOpenRouter` sub-node connected to **AI Churn Scoring** via the `ai_languageModel` port. Configured to use model `stepfun/step-3.5-flash:free` (a free-tier model on OpenRouter). No additional options are set (temperature, max tokens, etc. are all defaults). Credentials reference the `openRouterApi` credential named **"OpenRouter account for AI Journal"**.

---

### Parse AI Response (Code Node)
Iterates every item from **AI Churn Scoring** and:
1. Reads `item.json.output` (the raw string from the agent).
2. Extracts the first `{...}` block via regex to handle any extraneous text the model may produce.
3. `JSON.parse()`s the extracted block.
4. Clamps `churn_risk_score` to [0, 100] and `confidence_score` to [0, 1].
5. Lowercases `risk_level`; defaults to `"medium"` if invalid.
6. Ensures `churn_signals` and `root_causes` are arrays.
7. On any error (missing `output`, no JSON found, parse failure): returns `{ parsing_status: "failed", error: "...", raw_output: "..." }` — the item continues downstream with nulled score fields rather than blocking the run.

---

### Classify Risk (Code Node)
Reads `ai_scoring.churn_risk_score` (note: the field is read via optional chaining on `cust.ai_scoring?.churn_risk_score`, but the upstream Parse node spreads parsed fields to the top level — see Assumptions). Applies the threshold function:

| Score range | Tier | Urgency |
|---|---|---|
| ≤ 40 | `low` | `monitor` |
| 41–70 | `medium` | `engage` |
| ≥ 71 | `high` | `intervene` |
| null / undefined | `medium` | `engage` (fallback) |

Attaches `risk_classification: { tier, score, urgency, classified_at, action_required, batch_index }` to each item.

---

### High-Risk Split
An IF node (v2.2, strict type validation, case-sensitive). Evaluates `{{ $json.risk_classification.tier }}` against two OR-combined conditions:
- equals `"medium"` → true branch
- equals `"High"` → true branch _(note: the second condition compares against `"High"` with capital H, while Classify Risk emits lowercase `"high"` — see Known Limitations)_

The **true branch** routes to the email-sending path. The **false branch** routes directly to **Append Predictions**.

---

### Send Retention Email (Email Node)
Sends an HTML email via Gmail to the at-risk customer. Key configuration:

| Field | Value |
|---|---|
| **To** | `storysnippents@gamil.com` (hardcoded — see Known Limitations) |
| **Subject** | `"We Want to Help You Get More Value from Platform"` (static) |
| **From** | Gmail account credential |
| **Body** | HTML template with inline CSS; personalises `company_name` and `plan_type` from `Calculate Engagement Scores (Code Node)` via `$('Calculate Engagement Scores (Code Node)').item.json.*` |
| **Attribution footer** | Disabled (`appendAttribution: false`) |

The email body references `{{ $json.ai_scoring }}` to render the churn signals section. As `ai_scoring` is an object, this will render as `[object Object]` unless the LangChain agent stores the parsed result there (see Known Limitations).

---

### Append Predictions (Google Sheets)
Appends one row per customer to the `Daily_Predictions` tab (gid=476486533) of the main spreadsheet. Column mapping:

| Sheet Column | Expression / Value |
|---|---|
| `timestamp` | `={{ $now }}` |
| `customer_id` | `={{ $('Calculate Engagement Scores (Code Node)').item.json.customer_id }}` |
| `email` | `={{ $('Calculate Engagement Scores (Code Node)').item.json.email }}` |
| `company_name` | `={{ $('Calculate Engagement Scores (Code Node)').item.json.company_name }}` |
| `plan_type` | `={{ $('Calculate Engagement Scores (Code Node)').item.json.plan_type }}` |
| `churn_risk_score` | `={{ $json.churn_risk_score }}` |
| `risk_level` | `={{ $json.risk_level }}` |
| `recommended_action` | `={{ $json.risk_classification.action_required }}` _(boolean — see Known Limitations)_ |
| `confidence_score` | `={{ $json.confidence_score }}` |
| `retention_action_sent` | `={{ $json.recommended_retention_action }}` |
| `top_signal` | `{{ $json.churn_signals[0] }}\n{{ $json.churn_signals[1] }}\n...` (first 4 signals) |
| `csm_assigned` | `={{ $('Calculate Engagement Scores (Code Node)').item.json.customer_id }}` _(customer ID used as placeholder — see Known Limitations)_ |

---

### Append Actions (Google Sheets)
Appends one row per medium/high-risk customer to the `Retention_Actions` tab (gid=834404975). Column mapping:

| Sheet Column | Expression / Value |
|---|---|
| `timestamp` | `={{ $now.toISO() }}` |
| `customer_id` | `={{ $json.customer_id }}` |
| `action_type` | `email` (static string) |
| `action_detail` | `=Retention email sent to {{ $json.email }}` |
| `status` | `sent` (static) |
| `follow_up_date` | `={{ $now.plus(1, 'day').toISO() }}` |
| `outcome_status` | `pending` (static) |

---

### Execution Summary (Code Node)
Reads all items from the previous node, counts them by `risk_level`, and produces a single summary item:

```json
{
  "execution": {
    "id": "<n8n execution id>",
    "status": "success",
    "duration_seconds": <integer>,
    "started_at": "<ISO>",
    "completed_at": "<ISO>"
  },
  "summary": {
    "workflow_name": "AI Churn Prediction & Retention Engine",
    "total_customers_analyzed": <n>,
    "high_risk_customers": <n>,
    "medium_risk_customers": <n>,
    "low_risk_customers": <n>,
    "retention_actions_triggered": <same as high>,
    "churn_risk_rate": "<percent>",
    "success_rate": "100%"
  }
}
```

This node always emits `success_rate: "100%"` regardless of individual item parsing failures. It uses `$execution.startTime` for elapsed time calculation.

---

## Node Configuration

### AI Churn Scoring — System Prompt (condensed)

```
You are an expert SaaS customer success analyst specialising in churn prediction.
Output ONLY valid JSON. No markdown, no explanations.

Score 0-100. Base:
  - Inactive 90+ days → 60-80 base
  - Low usage → 40-60 base
  - Active → 0-25 base

Modifiers:
  - Payment issues: +20-30
  - High support frustration: +10-20
  - Enterprise plan: -10
  - Tenure 2+ years: -5
  - Activity score > 75: -15

Risk levels: 0-40 low | 41-70 medium | 71-100 high

Required JSON keys:
  churn_risk_score, risk_level, churn_signals[], root_causes[],
  recommended_retention_action, confidence_score
```

### High-Risk Split — IF Conditions

```
Condition 1: $json.risk_classification.tier == "medium"  (OR)
Condition 2: $json.risk_classification.tier == "High"
```

### Append Actions — Static Field Values

```
action_type:    "email"
status:         "sent"
outcome_status: "pending"
follow_up_date: now + 1 day (ISO)
```

---

## Branching Logic

The single conditional node in this workflow is **High-Risk Split**.

| Output | Condition | Destination |
|---|---|---|
| **True** | `tier == "medium"` OR `tier == "High"` | Send Retention Email → Append Predictions → Append Actions → Execution Summary |
| **False** | All other values (effectively `"low"`) | Append Predictions → Execution Summary |

Both branches converge at **Append Predictions**, meaning every customer — regardless of risk level — gets a prediction row. Only medium/high customers additionally receive an email and an action row.

---

## Variables & Expressions

Notable expressions used across the workflow:

| Node | Expression | Purpose |
|---|---|---|
| Merge Data | `(now - loginDate) / (1000*60*60*24)` | Compute `days_since_login` |
| Merge Data | `(now - signupDate) / (1000*60*60*24*30)` | Compute `tenure_months` |
| Calculate Scores | `Math.min(100, (monthlyUsage / 5000) * 100)` | Normalise usage to 100-pt scale |
| Calculate Scores | `lrs * 0.35 + ucs * 0.30 + sfs * 0.20 + prs * 0.15` | Weighted composite |
| AI Churn Scoring | `{{ $json.engagement_scores.composite_engagement_score }}` | Inject into LLM prompt |
| High-Risk Split | `{{ $json.risk_classification.tier }}` | IF condition value |
| Append Predictions | `$('Calculate Engagement Scores (Code Node)').item.json.*` | Back-reference earlier node data |
| Append Predictions | `{{ $json.churn_signals[0] }}\n...` | Multi-line top-signals cell |
| Append Actions | `{{ $now.plus(1, 'day').toISO() }}` | Follow-up date = tomorrow |

---

## Credentials Used

| Node | Credential Name | Credential Type | Service |
|---|---|---|---|
| Fetch Customers | Google Sheets account | `googleSheetsOAuth2Api` | Google Sheets |
| Fetch Support Tickets | Google Sheets account | `googleSheetsOAuth2Api` | Google Sheets |
| OpenRouter Chat Model | OpenRouter account for AI Journal | `openRouterApi` | OpenRouter AI |
| Send Retention Email | Gmail account | `gmailOAuth2` | Gmail (Google) |
| Append Predictions | Google Sheets account | `googleSheetsOAuth2Api` | Google Sheets |
| Append Actions | Google Sheets account | `googleSheetsOAuth2Api` | Google Sheets |

All three Google Sheets nodes share the same OAuth2 credential (`id: twmxeUbOfErGsbbM`). Actual secret values are not included in the export.

---

## Input Data

### From Google Sheets — `Customers` sheet

Expected columns (confirmed from node column-mapping expressions):

| Column | Type | Notes |
|---|---|---|
| `customer_id` | String | Required; used as join key |
| `email` | String | Required; validated by regex |
| `company_name` | String | Used in email template |
| `plan_type` | String | `Free` / `Pro` / `Enterprise` |
| `signup_date` | Date string | ISO 8601 expected |
| `last_login_date` | Date string | ISO 8601 expected |
| `monthly_usage` | Number | API calls or events per month |
| `support_tickets` | Number | Count (not directly used in scoring — ticket data comes from Support_Tickets sheet) |
| `last_payment_date` | Date string | ISO 8601 expected |
| `payment_status` | String | `active` / `failed` / `pending` / `expired` |

### From Google Sheets — `Support_Tickets` sheet

| Column | Type | Notes |
|---|---|---|
| `customer_id` | String | Join key to customer |
| `ticket_id` | String | Used to identify ticket rows |
| `created_date` | Date string | Not currently used in scoring |
| `sentiment` | String | `positive` / `neutral` / `negative` |
| `resolution_time_hours` | Number | Used in SFS computation |
| `category` | String | Stored in `recent_tickets`; not scored |

---

## Output Data

### `Daily_Predictions` sheet (all customers, every run)

| Column | Value |
|---|---|
| `timestamp` | Execution datetime |
| `customer_id` | From customer record |
| `email` | From customer record |
| `company_name` | From customer record |
| `plan_type` | From customer record |
| `churn_risk_score` | 0–100 integer from AI |
| `risk_level` | `low` / `medium` / `high` |
| `recommended_action` | Boolean (`action_required`) — _see Known Limitations_ |
| `confidence_score` | 0–1 float from AI |
| `retention_action_sent` | AI-recommended action string |
| `top_signal` | Up to 4 churn signals, newline-separated |
| `csm_assigned` | `customer_id` (placeholder) |

### `Retention_Actions` sheet (medium/high-risk customers only)

| Column | Value |
|---|---|
| `timestamp` | ISO datetime |
| `customer_id` | Customer ID |
| `action_type` | `"email"` (always) |
| `action_detail` | `"Retention email sent to <email>"` |
| `status` | `"sent"` (always) |
| `follow_up_date` | Tomorrow (ISO) |
| `outcome_status` | `"pending"` (always) |

### Execution Summary (in-memory, last node output)

An aggregated run-metrics item (total analyzed, tier counts, churn rate, duration). Not persisted to any external system in the current configuration.

---

## Data Transformation

The workflow performs meaningful data reshaping across three Code nodes:

1. **Merge Data:** Two flat sheets → one denormalised customer-centric record per customer. Adds calculated temporal fields and aggregated ticket metrics.
2. **Validate Data:** Adds a structured `validation` wrapper to every item; no rows are dropped.
3. **Calculate Engagement Scores:** Four scoring functions transform raw metrics into normalised 0–100 sub-scores and a weighted composite; adds four boolean risk-indicator flags.
4. **Parse AI Response:** Converts a raw LLM string into a typed, validated JSON structure with clamped numerics.
5. **Classify Risk:** Adds a tier/urgency classification object derived from a single numeric threshold function.

---

## AI / LLM Integration

| Field | Value |
|---|---|
| **Node** | AI Churn Scoring |
| **Node Type** | `@n8n/n8n-nodes-langchain.agent` (v3) |
| **Provider** | OpenRouter |
| **Model** | `stepfun/step-3.5-flash:free` |
| **Prompt Mode** | `define` (custom system + user prompt, not chat history) |
| **Temperature** | Not set (model default) |
| **Max Tokens** | Not set (model default) |
| **Tools / Memory** | None connected (Memory and Tool ports are empty despite being shown on the canvas) |
| **Output field** | `item.json.output` (the raw string the agent returns) |

The AI node runs once per customer item. With 20 customers and pinned data in the export, test runs replay the 20 pinned responses without hitting the API. In production (unpinned), each customer triggers one OpenRouter API call.

The system prompt enforces JSON-only output. The user prompt is pre-filled with engagement scores and customer metadata via n8n expressions. Two template placeholders remain unresolved: `{feature_usage_count}`, `{login_frequency}`, and `{recent_tickets_summary}` are literal strings (not n8n expressions) — the model receives them verbatim as placeholder text.

---

## API Integrations / External Services

| Service | Used By | Purpose | Auth Method |
|---|---|---|---|
| Google Sheets API (v4) | Fetch Customers, Fetch Support Tickets, Append Predictions, Append Actions | Read source data; write results | OAuth 2.0 |
| OpenRouter API | OpenRouter Chat Model | AI inference via `stepfun/step-3.5-flash:free` | API Key (`openRouterApi`) |
| Gmail API | Send Retention Email | Send HTML retention emails | OAuth 2.0 |

---

## Error Handling

| Node | Behaviour on Error |
|---|---|
| Merge Data (Code Node) | `try/catch` re-throws; execution halts this item path |
| Validate Data | No `try/catch`; validation errors are surfaced as item-level flags, not thrown exceptions — items continue |
| Calculate Engagement Scores | Validation-failed items receive zeroed scores; no exception raised |
| AI Churn Scoring | n8n agent default error behaviour (halts item if API call fails) |
| Parse AI Response (Code Node) | Per-item `try/catch`; parsing failures return `{ parsing_status: "failed" }` — items continue downstream with null risk fields |
| Classify Risk (Code Node) | Null/undefined score handled; defaults to `medium` tier |
| All other nodes | No explicit error handling configured |

**No `errorWorkflow` is configured** at the workflow settings level. There are no error-notification nodes (Slack, email to ops, etc.). Failed executions will appear in n8n's built-in execution log only.

**No retry logic** is configured on any node (no `retryOnFail`, `maxTries`, or `waitBetweenTries` settings). A transient OpenRouter or Gmail API failure will fail that item's execution without automatic retry.

---

## Execution Order

`settings.executionOrder` is set to **`"v1"`** (connection-based execution order). In v1 mode, n8n executes nodes in dependency order: a node runs as soon as all its incoming connections have data. This means **Fetch Customers** and **Fetch Support Tickets** run in parallel (both depend only on Schedule Trigger), and **Merge** waits for both to complete.

---

## Scheduling Details

| Field | Value |
|---|---|
| **Trigger type** | Schedule (cron-based) |
| **Frequency** | Every day |
| **Hour** | 9 (09:00) |
| **Minute** | 0 (default) |
| **Timezone** | Instance default — _not explicitly set in workflow settings. Recommend adding `"timezone": "UTC"` (or your business timezone) to `settings`_ |

---

## Security Considerations

- **No webhook** is exposed by this workflow — attack surface is limited to the scheduled execution environment.
- **OAuth2 credentials** are used for both Google Sheets and Gmail, which is more secure than API-key-based auth for Google services. Tokens are managed by n8n's credential store.
- **OpenRouter API key** is stored as an n8n credential (`openRouterApi`), not hardcoded.
- **The retention email `To` address is hardcoded** as `storysnippents@gamil.com` — this is almost certainly a development/test address and must be replaced before production use. In production the recipient should be `{{ $json.email }}` (the customer's actual email).
- Customer PII (email, company name) flows through the LLM prompt to OpenRouter's API. Ensure this aligns with your data-processing agreements and applicable privacy regulations (GDPR, CCPA, etc.).
- The Google Spreadsheet is shared — verify that sharing permissions are appropriately restricted to authorised service accounts only.
- _[Further security review to be provided by workflow owner / security team]_

---

## Performance Considerations

- **AI node is the bottleneck.** The LangChain Agent runs synchronously per item. With 20 customers and a hosted free-tier model, latency per call is likely 1–5 seconds, putting total AI scoring at 20–100 seconds. For large customer bases (1,000+), this will require batching, parallel sub-workflows, or upgrading to a faster/paid model.
- **Google Sheets API** has a rate limit of 300 read requests per minute per project. Reading two sheets per execution is well within limits for typical volumes; however, **append** operations for large customer lists may require batching.
- The **pinned data** on `AI Churn Scoring` bypasses live API calls during development — remember to un-pin before production activation.

---

## Execution Time

_[To be provided by workflow owner — requires live execution data. Estimated: 1–3 minutes for ≤ 20 customers with live OpenRouter calls; scales linearly with customer count due to sequential AI calls.]_

---

## Resource Usage

_[To be provided by workflow owner — requires execution data from n8n monitoring.]_

---

## Testing & Validation

_[To be provided by workflow owner.]_

The workflow export contains **pinned data on the `AI Churn Scoring` node** (20 sample AI responses covering low, medium, and high risk cases). This means the workflow can be tested end-to-end without live OpenRouter API calls by running with pinned data active.

Recommended test sequence:
1. Run with all 20 pinned AI responses to validate the full pipeline (merge → score → classify → split → email → sheets).
2. Un-pin the AI node and test with 1–3 real customers against the live OpenRouter API.
3. Validate that `Daily_Predictions` and `Retention_Actions` sheets receive correct rows.
4. Confirm the Gmail node sends emails only to medium/high-risk customers.
5. Activate the schedule and monitor the first 3 live runs.

---

## Troubleshooting

| Symptom | Likely Cause | Resolution |
|---|---|---|
| Merge step returns zero customers | `Customers` sheet is empty or column headers mismatch what the Merge Data Code expects | Verify sheet has data starting row 2 and that column names match exactly |
| Validate Data marks all items as failed | Date columns contain non-ISO formats (e.g. `03/04/2026` instead of `2026-03-04`) | Reformat date columns in Google Sheets to ISO 8601 |
| AI Churn Scoring node times out | OpenRouter API slow or rate-limited on free tier | Add delay between items; switch to a paid/faster model; check OpenRouter status |
| All customers routed to false branch (no emails sent) | `Classify Risk` emits `"high"` (lowercase) but High-Risk Split checks `"High"` (capital H) | Fix the IF condition to use lowercase `"high"` (see Known Limitations) |
| Retention email body shows `[object Object]` for churn signals | `{{ $json.ai_scoring }}` renders the raw object | Replace with `{{ $json.churn_signals.join(', ') }}` or iterate the array |
| `recommended_action` column in Daily_Predictions shows `true`/`false` | Expression maps `risk_classification.action_required` (boolean) | Change expression to `{{ $json.recommended_retention_action }}` |
| `csm_assigned` column shows customer ID, not a CSM name | Placeholder expression used | Replace with a lookup to a CSM-assignment table or static assignment |
| Gmail credential error | OAuth2 token expired | Re-authenticate the Gmail credential in n8n's credential manager |

---

## Known Limitations

1. **Hardcoded `To` email address.** The `Send Retention Email` node sends all retention emails to `storysnippents@gamil.com` (note: appears to be a typo of `@gmail.com`). In production the recipient must be the customer's own email address.
2. **High-Risk Split condition case mismatch.** One IF condition checks `tier == "High"` (capital H) while `Classify Risk` emits lowercase `"high"`. The `"High"` condition will never be true in production. Only `"medium"` customers will be routed to the email path. Fix: change `"High"` → `"high"` in the IF node.
3. **`recommended_action` column writes a boolean.** The `Append Predictions` column mapping uses `$json.risk_classification.action_required` (a boolean) for the `recommended_action` column. It should use `$json.recommended_retention_action` (the AI text).
4. **`csm_assigned` column uses `customer_id` as a placeholder.** No CSM assignment logic exists; the column will contain the customer ID, not an actual CSM name.
5. **Unresolved prompt placeholders.** The AI Churn Scoring user prompt contains three literal placeholder strings — `{feature_usage_count}`, `{login_frequency}`, and `{recent_tickets_summary}` — that are not n8n expressions. The model receives them as-is, which may reduce prediction accuracy for those signals.
6. **Validation failures are not quarantined.** Customers that fail the `Validate Data` checks still flow through to the AI node and prediction sheet (with zeroed scores). There is no separate error-output path or notification for data quality failures.
7. **No error workflow or failure notifications.** If the workflow fails mid-run (e.g. OpenRouter is down, Gmail quota is exceeded), there is no alert to operations. Failures are visible only in n8n's internal execution log.
8. **No retry logic on any node.** A single transient API failure will silently fail that customer's processing.
9. **Execution Summary counts `retention_actions_triggered` as equal to `high_risk_customers` count**, but the IF node actually routes both medium AND high-risk customers to the email path. The metric will undercount actual email sends.
10. **`recent_tickets_summary` in email body.** The email template body references `{{ $json.ai_scoring }}` (object) rather than a formatted signal list. This will render as `[object Object]` in the email.
11. **No pagination for Google Sheets reads.** If the `Customers` or `Support_Tickets` sheet exceeds ~10,000 rows (default Google Sheets API limit), the read operation may truncate results silently.

---

## Assumptions

- `Customers` and `Support_Tickets` sheets have headers in row 1 and data from row 2; the Google Sheets node reads them correctly via the default `headerRow: 1` setting.
- The `Merge` node's append mode is intentional — it combines both sheet's rows into one flat stream for the downstream Code node to split.
- `monthly_usage` in the Customers sheet represents API calls or product-usage events per month (used in the UCS normalisation against a 5,000-unit baseline).
- `activity_score` is not present in the Customers sheet schema but is referenced in the UCS formula — it defaults to 50 (neutral) if absent, which is the Code node's default parameter value.
- `feature_usage_count` is similarly not in the Customers sheet and defaults to 0 in the UCS formula.
- The workflow is intended to run in the n8n instance's local timezone; since no timezone is set in `settings`, that is whatever the instance default is.
- The 20 pinned AI responses in the export are representative of realistic outputs from the configured model and prompts.
- "Retention actions triggered" in the summary is intended to count medium + high customers, even though the code currently counts only high-risk items.

---

## Version History / Change Log

_[To be provided by workflow owner.]_

---

## References

| Resource | URL |
|---|---|
| Source Google Spreadsheet | `https://docs.google.com/spreadsheets/d/1zgRuVmJnCtl1Kks-ym6PGurgcO5KCT3lb453fiaL3o8/` |
| OpenRouter model page | `https://openrouter.ai/stepfun/step-3.5-flash` |
| n8n LangChain Agent docs | `https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent/` |
| n8n Schedule Trigger docs | `https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.scheduletrigger/` |

---

## Maintenance Notes

_[To be provided by workflow owner.]_

Suggested items to capture here:
- Who to contact when the OpenRouter API key needs rotation.
- Procedure for adding new customer signals to the engagement scoring node.
- Policy for how long `Daily_Predictions` rows are retained before archiving.
- Process for updating the LLM model (e.g. switching from `step-3.5-flash:free` to a paid model once volume grows).
- Steps to re-authenticate the Gmail and Google Sheets OAuth2 credentials when tokens expire.

---

_End of document. Sections requiring workflow owner input are marked_ `_[To be provided by workflow owner]_`.
