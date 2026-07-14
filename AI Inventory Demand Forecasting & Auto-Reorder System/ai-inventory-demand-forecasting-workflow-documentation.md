# AI Inventory Demand Forecasting & Auto-Reorder System — Workflow Documentation

| Field | Value |
|---|---|
| **Workflow Name** | AI Inventory Demand Forecasting & Auto-Reorder System |
| **Workflow ID** | `ny8kAQAcOz4IM7wz` |
| **Version ID** | `a4379518-c59c-46e0-8b7f-6927aec5ba60` (internal n8n revision hash — not a semantic version) |
| **Semantic Version** | _[To be provided by workflow owner — e.g. v1.0]_ |
| **Owner** | _[To be provided by workflow owner]_ |
| **Active** | ❌ No — workflow is currently **inactive** |
| **Execution Order** | v1 (connection-based) |
| **Last Saved** | _[Not present in export]_ |
| **Document Generated** | 2026-07-13 |

---

## Purpose

This workflow automates nightly inventory demand forecasting and purchase order generation for an e-commerce or retail operation. It pulls inventory, sales, and supplier data from Google Sheets, enriches each SKU with computed sales velocity and seasonality metrics, sends the enriched data to an AI language model (via OpenRouter) to generate 14-day and 30-day demand forecasts, calculates optimal reorder quantities, and — for SKUs that are critical or below their reorder point — generates purchase orders and emails a purchasing manager for approval.

---

## Business Use Case

_(Inferred from workflow structure — confirm or replace with official description.)_

Retail and e-commerce teams face stockouts and overstock because reorder decisions rely on manual analysis or stale reports. This workflow removes that manual step: it runs automatically every morning at 2 AM, identifies which SKUs need replenishment, and places a purchase order request in front of the right person with a single approve/reject decision link — before the business day starts.

---

## Workflow Summary

The Schedule Trigger fires at 2:00 AM daily and fans out to three parallel Google Sheets reads: one for current inventory levels, one for sales history, and one for supplier information. Inventory data is joined with supplier data in the Merge Supplier Data node (joined on `Supplier_ID`). The resulting records pass through the Feature Engineering code node, which computes `sales_velocity_7_day`, `sales_velocity_30_day`, `seasonality_index`, and `days_of_inventory_remaining` per SKU.

The enriched SKU list enters a batch loop (5 SKUs per iteration), where each batch is sent to the LangChain AI Agent node. The agent uses an OpenRouter-hosted free LLM (`arcee-ai/trinity-large-preview:free`) to produce a structured JSON forecast containing 14/30-day demand estimates, a confidence score, trend direction, and reorder urgency. The AI Response Parser code node extracts and sanitises this JSON, applying velocity-based fallback defaults when the model output is malformed.

The Reorder Quantity Calculator then computes `optimal_reorder_qty` using a safety-stock formula, and an IF node gates the downstream path: only SKUs flagged as critical or below their reorder point continue. A second IF node separates critical-stock SKUs from lower-urgency ones. Critical SKUs proceed to Purchase Order Generator, which builds a full PO object with a unique `PO_ID` and an `Approval_Token`. The Send Approval Notification node emails the manager, and the PO and forecast results are appended to dedicated Google Sheets tabs.

> **Note:** As currently wired, the Fetch Sales History output is not connected to any downstream node, meaning the Feature Engineering node receives no sales data and all velocity/seasonality values compute to zero. See [Known Limitations](#known-limitations).

---

## Workflow Diagram

See attached canvas screenshot. The workflow is shaped as follows:

```
Schedule Trigger
    ├── Fetch Inventory Data ──┐
    ├── Fetch Sales History    │  (DISCONNECTED — see Known Limitations)
    └── Fetch Supplier Data ───┤
                               ▼
                     Merge Supplier Data
                               │
                     Feature Engineering
                               │
                       Loop Over Items ◄─────────────────────────┐
                         (batch=5)                                │
                               │ (loop output)                    │
                     AI Demand Forecast ──────────────────────────┘
                        ├── (feed back to loop)
                        └── AI Response Parser
                               │
                   Reorder Quantity Calculator
                               │
                  IF: Needs Reorder Check (is_critical OR needs_reorder)
                       │ true               │ false
                       ▼                   (terminal — no action)
             IF: Critical Stock Alert
                  │ true      │ false
                  ▼         (terminal — no action)
         Purchase Order Generator
                  │
         Send Approval Notification (Gmail)
                  │
          Store Purchase Order (Sheets)
                  │
         Store Forecast Results (Sheets)
```

---

## Trigger Details

| Field | Value |
|---|---|
| **Node** | Schedule Trigger |
| **Type** | `n8n-nodes-base.scheduleTrigger` v1.3 |
| **Schedule** | Daily at **2:00 AM** |
| **Interval Type** | Hour-of-day (triggerAtHour: 2) |
| **Timezone** | Not explicitly set in workflow settings — defaults to n8n instance timezone |
| **Manual Execution** | Supported — use the "Execute workflow" button for testing |

The trigger fans out immediately to three parallel data-fetch nodes. All three Google Sheets reads begin simultaneously before the flow reconverges at Merge Supplier Data.

---

## Execution Flow

1. **Schedule Trigger** fires at 2 AM and activates three parallel paths simultaneously.

2. **Fetch Inventory Data**, **Fetch Sales History**, and **Fetch Supplier Data** each read their respective sheets from the same Google Spreadsheet (`1qCO9yv-4CgrjSmw1t-Oh64remBo15tLv0OwHFlRqSoc`).
   - ⚠️ **Fetch Sales History** output is not connected — its data does not reach Feature Engineering. Sales items will be empty.

3. **Merge Supplier Data** performs an inner join combining Fetch Inventory Data (input 0) with Fetch Supplier Data (input 1), matching on the `Supplier_ID` field. This enriches each inventory row with supplier name, email, and minimum order quantity.

4. **Feature Engineering** (Code Node) processes the merged records and computes per-SKU derived metrics: 7-day and 30-day sales velocity, seasonality index (last 30 days vs. prior 30 days), days of inventory remaining, and a 90-day sales history array. A boolean `is_critical` flag is set if stock is zero or days remaining is less than lead time.

5. **Loop Over Items** (SplitInBatches, batch size 5) feeds enriched SKU objects in groups of five to the AI step.

6. **OpenRouter Chat Model** (sub-node) supplies the LLM model `arcee-ai/trinity-large-preview:free` to the agent as the `ai_languageModel` input.

7. **AI Demand Forecast** (LangChain Agent) receives each batch item and calls the LLM with a structured system prompt and a per-SKU user prompt. The agent returns a JSON object containing `forecast_14_days`, `forecast_30_days`, `confidence_score`, `seasonality_flag`, `trend_direction`, `reorder_urgency`, and `reasoning`.
   - The agent output feeds back into **Loop Over Items** (to continue the batch loop) and simultaneously into **AI Response Parser**.

8. **AI Response Parser** (Code Node) extracts the AI's JSON from the raw response, stripping markdown fences and handling malformed output. If parsing fails, it falls back to velocity-based estimates with `parse_failed: true` and `confidence_score: 0.5`.

9. **Reorder Quantity Calculator** (Code Node) computes `optimal_reorder_qty` using:
   ```
   dailyDemand = forecast_30_days / 30
   leadTimeDemand = dailyDemand × Lead_Time_Days
   safetyStock = max(leadTimeDemand × 0.25, velocity_7_day × 5)
   rawQty = leadTimeDemand + safetyStock − Current_Stock
   reorder_qty = ceil(rawQty / Minimum_Order_Qty) × Minimum_Order_Qty
   ```
   It also sets `needs_reorder` (true if stock ≤ reorder point or is_critical) and re-evaluates `is_critical` (true if stock = 0 or reorder_urgency = "critical").

10. **IF: Needs Reorder Check** routes items where `is_critical = true` OR `needs_reorder = true` to the **true** branch. Items requiring no action go to the false branch, which has no downstream connection (silent terminal).

11. **IF: Critical Stock Alert** further filters the reorder-needed items: only those with `is_critical = true` continue to **true** branch → **Purchase Order Generator**. Items where `needs_reorder = true` but `is_critical = false` go to the false branch, which is also unconnected.
    - ⚠️ This means non-critical reorder items currently do not get POs generated. See [Known Limitations](#known-limitations).

12. **Purchase Order Generator** (Code Node) creates a full PO object for each critical SKU: unique `PO_ID` (date + random alphanumeric suffix), `Approval_Token` (random string), and `Expected_Delivery_Date` (today + lead time + 1 day processing).

13. **Send Approval Notification** emails the PO details to `storysnippents@gmail.com` (hardcoded) via Gmail OAuth2. The email body includes SKU status, AI forecast, PO details, and approve/reject URLs.

14. **Store Purchase Order** appends the PO record to the **Purchase Orders Table** sheet with columns: `PO_ID`, `SKU`, `Quantity`, `Supplier_ID`, `Expected_Delivery`, `Status`.

15. **Store Forecast Results** appends forecast data to the **Forecast Results Table** sheet with columns: `SKU`, `forecast_14_days`, `forecast_30_days`, `confidence_score`, `generated_at`.
    - ⚠️ This node pulls data from `$('Purchase Order Generator').item.json`, so forecast records are only written for items that reached the PO generation step (i.e., critical-only items).

---

## Node List

| # | Node Name | Type | Role/Purpose | Disabled |
|---|---|---|---|---|
| 1 | Schedule Trigger | Schedule Trigger | Fires daily at 2 AM to start the workflow | No |
| 2 | Fetch Inventory Data | Google Sheets (read) | Reads all rows from the Inventory Table sheet | No |
| 3 | Fetch Sales History | Google Sheets (read) | Reads all rows from the Sales History Table sheet | No |
| 4 | Fetch Supplier Data | Google Sheets (read) | Reads all rows from the Suppliers Table sheet | No |
| 5 | Merge Supplier Data | Merge (combine) | Inner-joins inventory with supplier data on Supplier_ID | No |
| 6 | Feature Engineering | Code (JavaScript) | Computes velocity, seasonality, days-remaining per SKU | No |
| 7 | Loop Over Items | Split In Batches | Batches enriched SKU items (5 per batch) for AI processing | No |
| 8 | OpenRouter Chat Model | OpenRouter LLM (sub-node) | Provides `arcee-ai/trinity-large-preview:free` as AI model | No |
| 9 | AI Demand Forecast | LangChain Agent | Calls LLM with per-SKU data to generate demand forecast JSON | No |
| 10 | AI Response Parser | Code (JavaScript) | Parses/sanitises LLM JSON output; applies fallback defaults | No |
| 11 | Reorder Quantity Calculator | Code (JavaScript) | Calculates optimal reorder qty using safety-stock formula | No |
| 12 | IF: Needs Reorder Check | IF | Routes items where is_critical OR needs_reorder is true | No |
| 13 | IF: Critical Stock Alert | IF | Routes only is_critical=true items to PO generation | No |
| 14 | Purchase Order Generator | Code (JavaScript) | Generates PO object with ID, token, delivery date | No |
| 15 | Send Approval Notification | Gmail (send message) | Emails the PO approval request to the purchasing manager | No |
| 16 | Store Purchase Order | Google Sheets (append) | Appends PO row to Purchase Orders Table | No |
| 17 | Store Forecast Results | Google Sheets (append) | Appends forecast row to Forecast Results Table | No |

---

## Node-by-Node Description

### Schedule Trigger
Activates the workflow once per day at 2:00 AM using n8n's built-in schedule trigger. The trigger outputs a single item and fans out to three Google Sheets nodes in parallel. No timezone is explicitly set in the workflow — it inherits the n8n instance default.

### Fetch Inventory Data
Reads all rows from the **Inventory Table** tab (gid=0) of the shared Google Spreadsheet. The sheet is expected to contain columns including `SKU`, `Current_Stock`, `Reorder_Point`, `Supplier_ID`, `Lead_Time_Days`, and `Unit_Cost`. All rows are returned without filtering.

### Fetch Sales History
Reads all rows from the **Sales History Table** tab (gid=1368983359). Expected columns: `SKU`, `Date`, `Units_Sold`. **This node's output is not connected to any downstream node**, so its data never enters the pipeline. See [Known Limitations](#known-limitations).

### Fetch Supplier Data
Reads all rows from the **Suppliers Table** tab (gid=1584781752). Expected columns: `Supplier_ID`, `Supplier_Name`, `Supplier_Email`, `Minimum_Order_Qty`. These are joined with inventory records downstream.

### Merge Supplier Data
Combines the inventory dataset (input 0, from Fetch Inventory Data) with the supplier dataset (input 1, from Fetch Supplier Data) using n8n's **Combine** merge mode, matching on the `Supplier_ID` field. This appends `Supplier_Name`, `Supplier_Email`, and `Minimum_Order_Qty` to each inventory row. Only rows with a matching `Supplier_ID` in both inputs are retained (inner join behaviour).

### Feature Engineering
A JavaScript Code node (Run Once For All Items) that accepts merged inventory+supplier rows and computes the following derived fields per SKU:

| Output Field | Description |
|---|---|
| `sales_velocity_7_day` | Average daily units sold over the last 7 days |
| `sales_velocity_30_day` | Average daily units sold over the last 30 days |
| `seasonality_index` | Ratio of last-30-day avg to prior-30-day avg (1.0 = no seasonality) |
| `days_of_inventory_remaining` | Current_Stock ÷ velocity_7_day (9999 if velocity is zero) |
| `sales_history_90d` | Array of {date, units} entries from the last 90 days, sorted chronologically |
| `is_critical` | `true` if Current_Stock = 0 OR days_remaining < Lead_Time_Days |

The node reads `$input.all(0)` for inventory items and `$input.all(1)` for sales items. Since Fetch Sales History is not connected, `$input.all(1)` always returns `[]`, so all velocity and seasonality values are 0 and 1.0 respectively at runtime.

### Loop Over Items (SplitInBatches)
Iterates over the enriched SKU list in batches of 5. The `reset: false` setting preserves batch state across executions. The **loop** output (index 1) sends each batch to AI Demand Forecast. The **done** output (index 0) is unconnected — when all batches are exhausted, the workflow terminates without an explicit summary step.

### OpenRouter Chat Model
A LangChain sub-node that configures the LLM to use: `arcee-ai/trinity-large-preview:free` (a free model on OpenRouter). It connects to AI Demand Forecast as the `ai_languageModel` input. Model temperature and other inference parameters are left at defaults — no `temperature` or `max_tokens` overrides are set in this node.

### AI Demand Forecast
A LangChain Agent (v3) with a custom system prompt and user prompt:

**System Prompt** instructs the model to act as an inventory forecasting AI, respond only in strict JSON (no markdown), and output an object with fields: `SKU`, `forecast_14_days`, `forecast_30_days`, `confidence_score`, `seasonality_flag`, `trend_direction`, `reorder_urgency`, and `reasoning`. Forecasting rules specify a weighted velocity (70% 7-day, 30% 30-day) with a seasonality multiplier.

**User Prompt** injects per-item values via n8n expressions: `$json.SKU`, `$json.Current_Stock`, `$json.Lead_Time_Days`, `$json.Reorder_Point`, `$json.sales_velocity_7_day`, `$json.sales_velocity_30_day`, `$json.seasonality_index`, `$json.days_of_inventory_remaining`, and `JSON.stringify($json.sales_history_90d)`.

The agent's output connects both back to Loop Over Items (to advance the batch counter) and to AI Response Parser.

### AI Response Parser
A JavaScript Code node that processes all items output by AI Demand Forecast. For each item it:
1. Extracts the raw text response from `choices[0].message.content`, `content[0].text`, or `.output`.
2. Strips markdown code fences (`` ```json `` / ` ``` `).
3. Extracts the first `{...}` JSON object using a regex match.
4. Attempts `JSON.parse()`; on failure, strips trailing commas and retries.
5. If all parsing fails, applies fallback defaults: velocity-based forecast estimates, `confidence_score: 0.5`, `parse_failed: true`.

The node merges the parsed forecast with the original enriched SKU data from `$items("Merge Supplier Data")` and appends `forecast_generated_at` (current ISO timestamp).

> **Note:** The parser references `$items("Merge Supplier Data")` by index to reattach inventory context. This assumes item order is preserved between Merge Supplier Data and AI Demand Forecast — this assumption holds as long as the loop processes items sequentially, but could break under parallel execution.

### Reorder Quantity Calculator
A JavaScript Code node (Run Once For All Items) that applies the reorder formula to each item:

```javascript
dailyDemand       = forecast_30_days / 30
leadTimeDemand    = dailyDemand × Lead_Time_Days
safetyStock       = max(leadTimeDemand × 0.25, velocity_7_day × 5)
rawQty            = leadTimeDemand + safetyStock − Current_Stock
reorder_qty       = ceil(rawQty / Minimum_Order_Qty) × Minimum_Order_Qty

// Fallback if forecast is zero but stock is at/below reorder point:
if (forecast_30 === 0 && currentStock <= reorderPoint):
    rawQty = reorderPoint − currentStock

// Ensure minimum = supplier MOQ; set to 0 if rawQty ≤ 0
```

Re-evaluates `is_critical` (stock = 0 OR reorder_urgency = "critical") and sets `needs_reorder` (stock ≤ reorder_point OR is_critical). Note: the original Feature Engineering `is_critical` flag (days_remaining < lead_time) is **overwritten** here with a different definition.

### IF: Needs Reorder Check
Evaluates two boolean conditions combined with **OR**:
- `$json.is_critical` equals `true`
- `$json.needs_reorder` equals `true`

**True branch** → IF: Critical Stock Alert  
**False branch** → unconnected (no further action for adequately-stocked SKUs)

### IF: Critical Stock Alert
Evaluates a single condition:
- `$json.is_critical` equals `true`

**True branch** → Purchase Order Generator  
**False branch** → unconnected

This means the workflow currently only generates purchase orders for **critical** SKUs, not for SKUs that need reordering but aren't critical. See [Known Limitations](#known-limitations).

### Purchase Order Generator
A JavaScript Code node that:
1. Filters to items where `needs_reorder = true` (redundant safety check at this point in the flow).
2. Generates a `PO_ID` as `PO-{YYYYMMDD}-{6-char random alphanumeric}`.
3. Generates an `Approval_Token` as two concatenated `Math.random().toString(36)` fragments (note: not cryptographically secure).
4. Calculates `Expected_Delivery_Date` as today + `Lead_Time_Days` + 1 (processing day).
5. Sets `Status: "PENDING_APPROVAL"`.
6. Computes `Estimated_Total_Cost` as `Unit_Cost × Quantity` if `Unit_Cost` is non-zero.

### Send Approval Notification
Sends a Gmail message using OAuth2 to the hardcoded recipient `storysnippents@gmail.com`. The subject line is dynamically composed: `[Inventory Alert] {REORDER_URGENCY} — SKU {SKU} Needs Reorder`. The body includes a full PO summary with approve/reject URLs using the `Approval_Token`. The approval endpoint base URL (`https://your-domain.com/approve`) is a placeholder and has not been implemented.

### Store Purchase Order
Appends one row per PO to the **Purchase Orders Table** sheet (gid=1248489354). Columns written: `PO_ID`, `SKU`, `Quantity`, `Supplier_ID`, `Expected_Delivery`, `Status`. Data is pulled from `$('Purchase Order Generator').item.json`. The node uses explicit field mapping (defineBelow mode) with `convertFieldsToString: false`.

### Store Forecast Results
Appends one row per processed SKU to the **Forecast Results Table** sheet (gid=1283099030). Columns written: `SKU`, `forecast_14_days`, `forecast_30_days`, `confidence_score`, `generated_at`. Two schema fields (`seasonality_flag`, `trend_direction`) are marked `removed: true` — they appear in the sheet schema but are not written. Data is also pulled from `$('Purchase Order Generator').item.json`, limiting records to critical/PO-generating items only.

---

## Node Configuration

### Google Spreadsheet (shared across all Sheets nodes)
| Field | Value |
|---|---|
| Spreadsheet ID | `1qCO9yv-4CgrjSmw1t-Oh64remBo15tLv0OwHFlRqSoc` |
| Spreadsheet Name | AI Inventory Demand Forecasting & Auto-Reorder System |
| URL | `https://docs.google.com/spreadsheets/d/1qCO9yv-4CgrjSmw1t-Oh64remBo15tLv0OwHFlRqSoc/edit` |

### Sheet Tab Mapping
| Node | Sheet Name | Sheet GID |
|---|---|---|
| Fetch Inventory Data | Inventory Table | 0 |
| Fetch Sales History | Sales History Table | 1368983359 |
| Fetch Supplier Data | Suppliers Table | 1584781752 |
| Store Purchase Order | Purchase Orders Table | 1248489354 |
| Store Forecast Results | Forecast Results Table | 1283099030 |

### Merge Supplier Data — Key Parameters
```
mode: combine
fieldsToMatchString: Supplier_ID
```

### Loop Over Items — Key Parameters
```
batchSize: 5
options.reset: false
```

### AI Demand Forecast — System Prompt (full)
```
You are an expert inventory demand forecasting AI for an e-commerce business.
You analyze sales velocity, seasonality trends, and historical patterns to forecast
future demand and flag reorder urgency.

RULES:
- Respond ONLY with valid JSON. No markdown, no explanation, no preamble.
- All numeric fields must be numbers (not strings).
- confidence_score must be between 0.0 and 1.0.
- seasonality_flag: true if seasonality_index deviates >20% from 1.0.
- forecast values must be integers (whole units).
- base your forecast on weighted velocity: 70% 7-day velocity, 30% 30-day velocity.
- apply seasonality_index multiplier to forecast.

OUTPUT SCHEMA (strict):
{
  "SKU": "string",
  "forecast_14_days": integer,
  "forecast_30_days": integer,
  "confidence_score": float,
  "seasonality_flag": boolean,
  "trend_direction": "increasing|stable|decreasing",
  "reorder_urgency": "critical|high|medium|low",
  "reasoning": "one sentence max"
}
```

### Send Approval Notification — Key Parameters
```
sendTo:  storysnippents@gmail.com  (hardcoded)
subject: [Inventory Alert] {{$json.Reorder_Urgency.toUpperCase()}} — SKU {{$json.SKU}} Needs Reorder
```

---

## Branching Logic

### IF: Needs Reorder Check
| Condition | Combinator | Result |
|---|---|---|
| `$json.is_critical === true` | OR | **True** → proceed to IF: Critical Stock Alert |
| `$json.needs_reorder === true` | OR | **True** → proceed to IF: Critical Stock Alert |
| Neither condition true | — | **False** → terminal (no further processing) |

### IF: Critical Stock Alert
| Condition | Combinator | Result |
|---|---|---|
| `$json.is_critical === true` | AND (single condition) | **True** → Purchase Order Generator |
| `$json.is_critical === false` | — | **False** → terminal (no PO generated) |

**Combined effect:** Only SKUs where `is_critical = true` receive a purchase order. SKUs where `needs_reorder = true` but `is_critical = false` are silently dropped after the first IF node.

---

## Loop Logic

**Loop Over Items** (SplitInBatches v3) implements the batch-processing loop:

| Setting | Value |
|---|---|
| Batch Size | 5 SKUs per iteration |
| Reset | false (state persists between executions — only resets when workflow runs fresh) |
| Done output (index 0) | Unconnected — loop exit is silent |
| Loop output (index 1) | → AI Demand Forecast |

**Loop mechanism:** AI Demand Forecast feeds its output back into Loop Over Items (main input 0) in addition to sending it to AI Response Parser. This is how n8n's SplitInBatches pattern works: each processed batch returns to the loop node, which then feeds the next batch until exhausted. The "done" signal fires when no more batches remain but — since it is unconnected — no downstream summarisation or cleanup occurs.

---

## Variables & Expressions

| Expression | Node | Purpose |
|---|---|---|
| `={{ $json.SKU }}` | AI Demand Forecast (user prompt) | Injects the SKU identifier into the AI prompt |
| `={{ $json.Current_Stock }}` | AI Demand Forecast | Current inventory level for AI context |
| `={{ $json.sales_velocity_7_day }}` | AI Demand Forecast | 7-day velocity metric |
| `={{ $json.sales_velocity_30_day }}` | AI Demand Forecast | 30-day velocity metric |
| `={{ $json.seasonality_index }}` | AI Demand Forecast | Seasonality ratio |
| `={{ JSON.stringify($json.sales_history_90d) }}` | AI Demand Forecast | Serialises the 90-day sales array for the prompt |
| `={{ $json.Reorder_Urgency.toUpperCase() }}` | Send Approval Notification (subject) | Capitalises urgency level in email subject |
| `={{ $json.Is_Critical ? "🔴 CRITICAL" : "🟡 LOW STOCK" }}` | Send Approval Notification (body) | Ternary for status icon in email body |
| `={{ Math.round($json.Confidence_Score * 100) }}%` | Send Approval Notification (body) | Converts confidence float to percentage |
| `={{ $('Purchase Order Generator').item.json.PO_ID }}` | Store Purchase Order | Pulls PO_ID from the PO Generator node by name |
| `=$input.all(0)` / `=$input.all(1)` | Feature Engineering (code) | Reads inventory (input 0) and sales (input 1) — input 1 is unconnected |
| `=$items("Merge Supplier Data")[index]` | AI Response Parser (code) | Reattaches inventory context to parsed forecast by index |

---

## Credentials Used

| Node | Credential Name | Type | Service |
|---|---|---|---|
| Fetch Inventory Data | Google Sheets account | `googleSheetsOAuth2Api` | Google Sheets |
| Fetch Sales History | Google Sheets account | `googleSheetsOAuth2Api` | Google Sheets |
| Fetch Supplier Data | Google Sheets account | `googleSheetsOAuth2Api` | Google Sheets |
| Store Purchase Order | Google Sheets account | `googleSheetsOAuth2Api` | Google Sheets |
| Store Forecast Results | Google Sheets account | `googleSheetsOAuth2Api` | Google Sheets |
| OpenRouter Chat Model | OpenRouter account for AI Journal | `openRouterApi` | OpenRouter (LLM API) |
| Send Approval Notification | Gmail account | `gmailOAuth2` | Gmail |

All credentials are referenced by ID/name only — no secret values are stored in the export.

---

## Input Data

**Trigger:** The workflow has no external input payload — it is triggered on schedule with no parameters.

**Inventory Table** (expected columns):
| Column | Type | Description |
|---|---|---|
| SKU | String | Unique product identifier |
| Current_Stock | Number | Units currently on hand |
| Reorder_Point | Number | Stock level that triggers reorder |
| Supplier_ID | String | Foreign key to Suppliers Table |
| Lead_Time_Days | Number | Days from PO to delivery |
| Unit_Cost | Number | Per-unit cost (optional) |

**Sales History Table** (expected columns):
| Column | Type | Description |
|---|---|---|
| SKU | String | Product identifier |
| Date | Date string | Sale date (parseable by `new Date()`) |
| Units_Sold | Number | Units sold on that date |

**Suppliers Table** (expected columns):
| Column | Type | Description |
|---|---|---|
| Supplier_ID | String | Primary key |
| Supplier_Name | String | Human-readable name |
| Supplier_Email | String | Contact email for PO routing |
| Minimum_Order_Qty | Number | MOQ constraint |

---

## Output Data

| Terminal Node | Output | Destination |
|---|---|---|
| Store Purchase Order | Appended row per critical PO | Purchase Orders Table (gid=1248489354) |
| Store Forecast Results | Appended row per critical SKU forecast | Forecast Results Table (gid=1283099030) |
| Send Approval Notification | Email per PO | `storysnippents@gmail.com` (hardcoded) |
| IF: Needs Reorder Check (false) | No action | Silent terminal |
| IF: Critical Stock Alert (false) | No action | Silent terminal |
| Loop Over Items (done) | No action | Silent terminal |

---

## Data Transformation

1. **Merge Supplier Data** joins two flat row sets (inventory + supplier) into enriched records using `Supplier_ID` as the key. Rows in Inventory without a matching supplier are dropped.

2. **Feature Engineering** transforms raw inventory rows into forecast-ready objects by computing velocity (rolling averages over 7 and 30 days), seasonality index (period ratio), days-remaining (stock/velocity), and a `sales_history_90d` array for AI context.

3. **AI Response Parser** normalises the unstructured LLM text output into a typed JavaScript object, coercing `forecast_14_days`/`forecast_30_days` to integers and `confidence_score` to a float.

4. **Reorder Quantity Calculator** converts the forecast into actionable quantities, applying the safety-stock formula and rounding up to supplier MOQ.

5. **Purchase Order Generator** reframes per-SKU reorder data as a formal PO document, adding generated identifiers (`PO_ID`, `Approval_Token`) and derived fields (`Expected_Delivery_Date`, `Estimated_Total_Cost`).

6. **Store Forecast Results** reads data back from `$('Purchase Order Generator').item.json` rather than from its immediate input — this means it writes the PO-augmented object, not the raw forecast object.

---

## API Integrations / External Services

| Service | Node | Endpoint / Method | Auth Type | Purpose |
|---|---|---|---|---|
| Google Sheets API v4 | Fetch Inventory Data, Fetch Sales History, Fetch Supplier Data | `GET spreadsheets/{id}/values/{range}` | OAuth2 | Read source data |
| Google Sheets API v4 | Store Purchase Order, Store Forecast Results | `POST spreadsheets/{id}/values/{range}:append` | OAuth2 | Write results |
| OpenRouter API | OpenRouter Chat Model | `POST https://openrouter.ai/api/v1/chat/completions` | API Key | LLM inference |
| Gmail API | Send Approval Notification | `POST /gmail/v1/users/me/messages/send` | OAuth2 | Manager approval email |

**OpenRouter Model:** `arcee-ai/trinity-large-preview:free` — this is a free-tier model. Free models on OpenRouter may have rate limits, context length restrictions, or reduced reliability compared to paid models.

---

## Error Handling

**No error handling is configured on any node.** Specifically:

- No node has `continueOnFail` or `onError` set.
- No `errorWorkflow` is defined in the workflow settings.
- No retry configuration exists on any node (no `retryOnFail`, `maxTries`, or `waitBetweenTries`).
- No Error Trigger node is present.
- There is no Slack alert, email, or logging node on a failure path.

If any node fails (API timeout, malformed data, rate limit, etc.), the entire workflow execution stops silently. The AI Response Parser provides **soft error handling** for AI output — if the model returns malformed JSON, it falls back to velocity-based defaults — but this is the only in-flow resilience.

---

## Retry Strategy

No retry logic is configured for any node. For a production deployment, consider adding retry settings on at minimum:

- **Fetch Inventory / Sales / Supplier Data** — Google Sheets API can return transient 5xx errors.
- **AI Demand Forecast (via Loop)** — OpenRouter free-tier models may be rate-limited.
- **Send Approval Notification** — Gmail API is generally reliable but could fail on transient network issues.

---

## Scheduling Details

| Field | Value |
|---|---|
| Trigger Type | Schedule (n8n-nodes-base.scheduleTrigger) |
| Frequency | Daily |
| Trigger Hour | 2 (2:00 AM) |
| Trigger Minute | Not configured — defaults to :00 |
| CRON equivalent | `0 2 * * *` |
| Timezone | Inherits n8n instance default (not set in workflow) |

---

## Execution Order

The workflow uses `executionOrder: v1` (connection-based execution order). In this mode, n8n executes nodes in the order they receive completed data from upstream, respecting connection topology. The three initial Google Sheets reads triggered by Schedule Trigger run in parallel under v1. All other paths are sequential.

---

## Security Considerations

- **Gmail recipient is hardcoded** to `storysnippents@gmail.com`. If this is a personal/test account in a production deployment, all PO notifications would go to the wrong destination.
- **Approval URLs are placeholder** (`https://your-domain.com/approve?token=...`). The approve/reject endpoint does not exist and must be built before the workflow is production-ready.
- **Approval Token is not cryptographically secure.** It uses `Math.random().toString(36)`, which is not suitable for security-sensitive token generation. Use `crypto.randomBytes()` for production approval tokens.
- **Google Spreadsheet ID is hardcoded.** Changing the data source requires editing the workflow JSON, not a configuration file. Consider using a Set node or n8n environment variables to externalise this.
- **No webhook authentication** — not applicable (no webhook trigger used).
- **OpenRouter API key** is stored as a named n8n credential (`OpenRouter account for AI Journal`) — this is appropriate and follows n8n best practice.

---

## Performance Considerations

- **Batch Size = 5:** At 5 SKUs per AI call, a catalog of 1,000 SKUs requires 200 LLM API calls. At 500ms–2s per call (free tier), this is approximately 2–7 minutes of AI processing time.
- **Free model latency:** `arcee-ai/trinity-large-preview:free` on OpenRouter's free tier may have higher latency and lower priority than paid models. During peak hours, this could extend significantly.
- **No rate-limit protection:** There is no Wait node or backoff between loop iterations. At scale, this risks hitting OpenRouter's rate limits on the free tier.
- **Google Sheets API quota:** The default quota is 100 read requests per 100 seconds per user. With 3 parallel reads at startup and 2 append operations per critical SKU, this is unlikely to be an issue for catalogs under ~500 SKUs.
- **Sales history scope:** The Feature Engineering node fetches all rows from Sales History (no date filter at the Sheets level), so large history tables would increase memory usage.

---

## Execution Time

_[To be provided by workflow owner — requires actual execution data from n8n's execution log]_

Estimated range based on configuration:
- Small catalog (< 100 SKUs): ~2–4 minutes
- Medium catalog (100–500 SKUs): ~5–20 minutes
- Large catalog (1,000+ SKUs): ~30–60+ minutes (may require optimisation)

---

## Resource Usage

_[To be provided by workflow owner — requires monitoring data]_

---

## Troubleshooting

**No forecast data in the Forecast Results Table:**
- Check that the workflow reached the Purchase Order Generator node. Because Store Forecast Results pulls data from `$('Purchase Order Generator').item.json`, forecasts are only written for critical-stock items.
- Verify at least one SKU has `is_critical = true` in the Reorder Quantity Calculator output.

**All velocity values are 0 / AI forecasts are based on zero demand:**
- This is the expected behaviour until Fetch Sales History is wired to Feature Engineering. Connect it as a second input to the Feature Engineering node (input index 1).

**AI Demand Forecast returns no output / empty responses:**
- Check the OpenRouter API key credential (`OpenRouter account for AI Journal`) is valid and the key has not expired or been rate-limited.
- The `arcee-ai/trinity-large-preview:free` model may be temporarily unavailable. Try substituting `mistralai/mistral-7b-instruct:free` or `nvidia/llama-3.1-nemotron-70b-instruct:free` in the OpenRouter Chat Model node.

**Approval email not received:**
- Check that the Gmail credential is authorised with the correct Google account.
- Confirm the recipient `storysnippents@gmail.com` — it is hardcoded and may need to be updated to the real purchasing manager's address.

**IF: Critical Stock Alert is routing some reorder items to the false (unconnected) branch:**
- This is expected current behaviour. SKUs where `needs_reorder = true` but `is_critical = false` are dropped here. Connect the false branch to the Purchase Order Generator if you want all reorder items to generate POs, not just critical ones.

**Loop completes but no summary / final count is available:**
- The Loop Over Items "done" output is unconnected. Add a terminal Set or Notification node on the done output to log or report completion if needed.

---

## Known Limitations

1. **Fetch Sales History is disconnected.** The Sales History node's output is not connected to Feature Engineering. As a result, `sales_velocity_7_day`, `sales_velocity_30_day`, `seasonality_index`, and `days_of_inventory_remaining` always compute to `0`, `0`, `1.0`, and `9999` respectively. The AI agent still runs, but it receives empty `sales_history_90d` arrays and zero velocities, severely limiting forecast accuracy.

2. **Non-critical reorder items are silently dropped.** The IF: Critical Stock Alert false branch is unconnected. SKUs where `needs_reorder = true` but `is_critical = false` receive no purchase order, no notification, and no stored record. Only critical-stock SKUs complete the full pipeline.

3. **Store Forecast Results only stores critical-item forecasts.** Because this node references `$('Purchase Order Generator').item.json`, it only fires for items that reached the PO generator (i.e., critical items). Forecast data for adequately-stocked or non-critical-reorder SKUs is not persisted.

4. **Approval tokens are not cryptographically secure.** `Math.random()` is not suitable for security-sensitive approval tokens in a production environment.

5. **Approval endpoint is a placeholder.** The `https://your-domain.com/approve` and `https://your-domain.com/reject` URLs in the email body do not point to a real service. These must be implemented before the workflow is production-ready.

6. **Gmail recipient is hardcoded.** All approval notifications go to a single hardcoded address. There is no routing logic based on supplier, SKU category, or team.

7. **No error handling or retry logic.** Any node failure silently terminates the workflow with no alert, no fallback path, and no partial-run recovery.

8. **Workflow is inactive.** The workflow was exported in an inactive state (`"active": false`). It must be activated in the n8n UI before nightly scheduling takes effect.

9. **Seasonality_flag and trend_direction not stored.** Both fields are marked `removed: true` in the Store Forecast Results schema, so they are not written to the sheet even though the AI generates them.

---

## Testing & Validation

_[To be provided by workflow owner — test cases and expected outputs not available from the export]_

Recommended pre-activation tests:
- **Unit test Feature Engineering:** Manually execute with a small set of known SKUs and verify computed velocity/seasonality values against hand-calculated expectations.
- **AI output test:** Run a single-SKU test with `Loop Over Items` batch size = 1 and inspect the AI Demand Forecast raw output before the parser.
- **Approval email test:** Trigger manually with one critical SKU, confirm the email arrives at the intended address with correct content.
- **Sheets write test:** Verify column mapping aligns with actual sheet headers in Purchase Orders Table and Forecast Results Table.

---

## Assumptions

1. The Sales History Table date values are parseable by JavaScript's `new Date()` constructor (e.g. `YYYY-MM-DD` or ISO 8601 format). If dates are in a regional locale format (e.g. `DD/MM/YYYY`), velocity calculations will fail silently.
2. The `Supplier_ID` field is populated consistently across both the Inventory Table and Suppliers Table. Inventory rows with no matching supplier are silently dropped by the inner join.
3. The n8n instance timezone is set to the intended local business timezone. The 2 AM trigger fires in the instance's local time, not necessarily UTC.
4. The OpenRouter `arcee-ai/trinity-large-preview:free` model remains available on the free tier. Free-tier models on OpenRouter can be deprecated or rate-limited without notice.
5. The `$items("Merge Supplier Data")` reference in AI Response Parser preserves item ordering from Feature Engineering through the loop. This holds under sequential batch execution but has not been verified under parallel conditions.
6. `Unit_Cost` in the Inventory Table is denominated in the same currency as `Estimated_Total_Cost` in the PO email. If the sheet lacks a `Unit_Cost` column, this field will be null and cost estimates will not appear in the email.

---

## Version History / Change Log

_[To be provided by workflow owner]_

---

## References

- n8n SplitInBatches documentation: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.splitinbatches/
- n8n LangChain Agent node: https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent/
- OpenRouter free models: https://openrouter.ai/models?q=free
- Google Sheets API quotas: https://developers.google.com/sheets/api/limits

---

## Maintenance Notes

_[To be provided by workflow owner]_

Key areas requiring attention before production deployment:
- Wire Fetch Sales History → Feature Engineering (second input)
- Connect IF: Critical Stock Alert false branch → Purchase Order Generator (or a separate non-critical reorder path)
- Implement the `/approve` and `/reject` webhook endpoints and update the URLs in Send Approval Notification
- Replace `Math.random()` token generation in Purchase Order Generator with `crypto.randomBytes(16).toString('hex')`
- Add retry logic (`retryOnFail: true`, `maxTries: 3`) to the three Fetch nodes and the AI loop
- Add an error workflow in Settings → Error Workflow for failure alerting
- Replace hardcoded Gmail recipient with a configurable parameter or environment variable
- Activate the workflow after completing the above
