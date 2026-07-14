# AI Lead Qualification System — Workflow Documentation

**Workflow ID:** `cIpWnKWLzzGOQvn7`
**Status:** Inactive (`active: false`)
**Execution Order:** v1
**Total Nodes:** 9 (8 main flow + 1 AI sub-model)

---

## 1. Overview

This n8n workflow receives lead data via webhook, normalizes it, sends it to an AI Agent (powered by OpenRouter) for qualification, logs the result to Google Sheets, and sends a Telegram alert when a lead is qualified as **Hot**.

```
Webhook → Normalize Data → Prepare AI Input → AI Agent → Parse AI Response
        → Save Lead (Google Sheets) → If (Qualification = Hot) → Telegram Notification
```

The AI Agent node also connects to a separate **OpenRouter Chat Model** sub-node (via the `ai_languageModel` connection), which supplies the actual LLM.

---

## 2. Visual Flow (as built)

| Order | Node | Type |
|---|---|---|
| 1 | Webhook | `n8n-nodes-base.webhook` |
| 2 | Normalize Data | `n8n-nodes-base.set` |
| 3 | Prepare AI Input | `n8n-nodes-base.code` |
| 4 | AI Agent | `@n8n/n8n-nodes-langchain.agent` |
| 4a | OpenRouter Chat Model | `@n8n/n8n-nodes-langchain.lmChatOpenRouter` (sub-node feeding AI Agent) |
| 5 | Parse AI Response | `n8n-nodes-base.code` |
| 6 | Save Lead | `n8n-nodes-base.googleSheets` |
| 7 | If | `n8n-nodes-base.if` |
| 8 | Telegram Notification | `n8n-nodes-base.telegram` |

---

## 3. Node-by-Node Breakdown

### Node 1 — Webhook
- **Type:** `n8n-nodes-base.webhook` (v2.1)
- **HTTP Method:** `POST`
- **Path:** `/lead-capture`
- **Response Mode:** `lastNode` (returns whatever the last executed node outputs)
- **Webhook ID:** `dcd4bddd-3d23-4acf-adc5-232b4ec9072a`
- **Expected incoming payload (`body`):**
  ```json
  {
    "name": "string",
    "email": "string",
    "phone": "string",
    "company": "string",
    "message": "string",
    "source": "string"
  }
  ```
- **Connects to:** Normalize Data

---

### Node 2 — Normalize Data
- **Type:** `n8n-nodes-base.set` (v3.4)
- **Purpose:** Extracts and flattens the raw webhook body into clean top-level fields.
- **Field Assignments:**

  | Output Field | Expression |
  |---|---|
  | `lead_name` | `{{ $json.body.name }}` |
  | `lead_email` | `{{ $json.body.email }}` |
  | `lead_phone` | `{{ $json.body.phone }}` |
  | `lead_company` | `{{ $json.body.company }}` |
  | `lead_message` | `{{ $json.body.message }}` |
  | `lead_source` | `{{ $json.body.source }}` |
  | `timestamp` | `{{ $now.toISO() }}` |

- **⚠️ Note:** No `lead_id` field is generated here, even though the downstream **Prepare AI Input** node references `$input.item.json.lead_id`. This means `lead_id` will currently pass through as `undefined` unless added. See [Section 6 – Known Issues](#6-known-issues--recommended-fixes).
- **Connects to:** Prepare AI Input

---

### Node 3 — Prepare AI Input
- **Type:** `n8n-nodes-base.code` (v2, JavaScript)
- **Purpose:** Builds the natural-language prompt block that gets embedded into the AI Agent's user message, and repackages lead fields under clean keys (`name`, `email`, etc.).
- **Logic:**
  ```javascript
  const leadData = {
    name: $input.item.json.lead_name,
    email: $input.item.json.lead_email,
    phone: $input.item.json.lead_phone,
    company: $input.item.json.lead_company,
    message: $input.item.json.lead_message,
    source: $input.item.json.lead_source
  };

  const aiPrompt = `Analyze this lead and provide qualification:
  Lead Information:
  - Name: ${leadData.name}
  - Email: ${leadData.email}
  - Company: ${leadData.company}
  - Message: ${leadData.message}
  - Source: ${leadData.source}
  Evaluate and respond ONLY with valid JSON.`;

  return {
    json: {
      ...leadData,
      ai_prompt: aiPrompt,
      lead_id: $input.item.json.lead_id,
      timestamp: $input.item.json.timestamp
    }
  };
  ```
- **Output Fields:** `name`, `email`, `phone`, `company`, `message`, `source`, `ai_prompt`, `lead_id`, `timestamp`
- **Connects to:** AI Agent

---

### Node 4 — AI Agent
- **Type:** `@n8n/n8n-nodes-langchain.agent` (v3)
- **Prompt Type:** `define` (custom prompt text, not auto-chat)
- **User Prompt (`text`):**
  ```
  Analyze the following lead information and qualify the lead.

  Return ONLY a valid JSON object using this EXACT structure:

  {
    "qualification": "Hot" | "Warm" | "Cold",
    "intent_score": 1-10,
    "urgency_level": "High" | "Medium" | "Low",
    "budget_signals": "Strong" | "Moderate" | "Weak" | "None",
    "relevance_score": 1-10,
    "reasoning": "brief explanation",
    "recommended_action": "specific next step"
  }

  Qualification Rules:
  - Hot: Clear buying intent, high urgency, strong/moderate budget signals, high relevance, scores 8–10
  - Warm: Moderate intent, some urgency OR budget indication, medium relevance, scores 5–7
  - Cold: Low/unclear intent, no urgency, weak/no budget signals, low relevance, scores 1–4

  Lead Data:
  {{ $json.ai_prompt }}

  Important:
  - Do NOT explain your output
  - Do NOT include markdown or commentary
  - Output ONLY the JSON object
  ```
- **System Message (`options.systemMessage`):**
  ```
  You are a senior B2B lead qualification expert with experience in sales
  operations, CRM scoring, and inbound lead analysis.

  Your task is to evaluate leads objectively based on intent, urgency,
  budget signals, and relevance.

  You MUST:
  - Respond with ONLY valid JSON
  - Follow the exact JSON schema provided
  - Use concise, professional reasoning
  - Never add explanations, markdown, or extra text outside JSON
  - Be conservative and realistic in scoring (avoid over-scoring weak leads)

  You are optimized for accuracy, consistency, and automation workflows.
  ```
- **Model connection (`ai_languageModel`):** OpenRouter Chat Model (see Node 4a)
- **Output:** `output` field containing the raw LLM text response (expected to be a JSON string).
- **Connects to:** Parse AI Response

---

### Node 4a — OpenRouter Chat Model
- **Type:** `@n8n/n8n-nodes-langchain.lmChatOpenRouter` (v1)
- **Model:** `nvidia/nemotron-nano-12b-v2-vl:free`
- **Credential:** `OpenRouter account` (id: `IfKkKP8K0XM7arme`)
- **Role:** Supplies the language model backend to the AI Agent node via the `ai_languageModel` connection (not part of the main data flow — it's a sub-connection).

---

### Node 5 — Parse AI Response
- **Type:** `n8n-nodes-base.code` (v2, JavaScript)
- **Purpose:** Extracts the JSON object from the AI Agent's raw text output and merges it with the original lead data for storage.
- **Logic:**
  ```javascript
  const leadData = $input.first().json;
  const aiResponse = $input.first().json.output;

  let parsedAI;
  try {
    const jsonMatch = aiResponse.match(/\{[\s\S]*\}/);
    parsedAI = JSON.parse(jsonMatch[0]);
  } catch (error) {
    parsedAI = {
      qualification: "Cold",
      intent_score: 1,
      urgency_level: "Low",
      budget_signals: "None",
      relevance_score: 1,
      reasoning: "AI parsing failed",
      recommended_action: "Manual review required"
    };
  }

  return {
    json: {
      lead_id: leadData.lead_id,
      timestamp: leadData.timestamp,
      name: leadData.name,
      email: leadData.email,
      phone: leadData.phone,
      company: leadData.company,
      message: leadData.message,
      source: leadData.source,
      qualification: parsedAI.qualification,
      intent_score: parsedAI.intent_score,
      urgency_level: parsedAI.urgency_level,
      budget_signals: parsedAI.budget_signals,
      relevance_score: parsedAI.relevance_score,
      reasoning: parsedAI.reasoning,
      recommended_action: parsedAI.recommended_action
    }
  };
  ```
- **Fallback behavior:** If the AI output isn't valid JSON, the lead is automatically defaulted to **Cold** with a "Manual review required" note — this prevents workflow failure on malformed AI output.
- **Connects to:** Save Lead

---

### Node 6 — Save Lead (Google Sheets)
- **Type:** `n8n-nodes-base.googleSheets` (v4.7)
- **Operation:** `append`
- **Spreadsheet:** `qualified lead` (ID: `1fJhStZ-3FzfjugkMYM3CKnPSp_KbAEaWrCp2JNvYmP4`)
- **Sheet:** `Sheet1` (gid=0)
- **Credential:** `Google Sheets account` (id: `twmxeUbOfErGsbbM`)
- **Mapped Columns:**

  | Sheet Column | Source Expression |
  |---|---|
  | Qualification | `{{ $json.qualification }}` |
  | Intent Score | `{{ $json.intent_score }}` |
  | Urgency Level | `{{ $json.urgency_level }}` |
  | Budget Signals | `{{ $json.budget_signals }}` |
  | Relevance Score | `{{ $json.relevance_score }}` |
  | Reasoning | `{{ $json.reasoning }}` |
  | Recommended Action | `{{ $json.recommended_action }}` |
  | Timestamp | `{{ $('Normalize Data').item.json.timestamp }}` |
  | Name | `{{ $('Normalize Data').item.json.lead_name }}` |
  | Email | `{{ $('Normalize Data').item.json.lead_email }}` |
  | Phone | `{{ $('Normalize Data').item.json.lead_phone }}` |
  | Company | `{{ $('Normalize Data').item.json.lead_company }}` |
  | Message | `{{ $('Normalize Data').item.json.lead_message }}` |
  | Source | `{{ $('Normalize Data').item.json.lead_source }}` |

- **⚠️ Note:** The sheet schema includes a **"Lead ID"** column, but it is **not mapped** in the `value` block, so it will always be written blank. See [Known Issues](#6-known-issues--recommended-fixes).
- **Connects to:** If

---

### Node 7 — If
- **Type:** `n8n-nodes-base.if` (v2.2)
- **Condition:** `{{ $json.Qualification }}` **equals** `Hot`
  - This reads the `Qualification` field as returned by the Google Sheets node after the append operation (the appended row is echoed back with column-name keys).
- **True branch →** Telegram Notification
- **False branch →** *(no node connected — workflow silently ends for Warm/Cold leads)*
- **⚠️ Note:** The `rightValue` is stored as `"=Hot"` in the JSON. The leading `=` denotes an n8n expression prefix; verify in the UI that this renders as the plain string `Hot` and not an unresolved expression. See [Known Issues](#6-known-issues--recommended-fixes).

---

### Node 8 — Telegram Notification
- **Type:** `n8n-nodes-base.telegram` (v1.2)
- **Operation:** Send Message (default)
- **Chat ID:** `7350267365` *(hardcoded)*
- **Credential:** `Telegram account for AI Journal` (id: `WM4UJf38iPD545XB`)
- **Message Template:**
  ```
  🔥 HOT LEAD ALERT 🔥

  📋 Lead Details:
  - Name: {{ $json.name }}
  - Company: {{ $json.company }}
  - Email: {{ $json.email }}
  - Phone: {{ $json.phone }}

  📊 Qualification:
  - Status: {{ $json.qualification }}
  - Intent Score: {{ $json.intent_score }}/10
  - Urgency: {{ $json.urgency_level }}
  - Budget Signals: {{ $json.budget_signals }}

  💡 AI Reasoning:
  {{ $json.reasoning }}

  ✅ Recommended Action:
  {{ $json.recommended_action }}

  📝 Original Message:
  {{ $json.message }}

  🕐 Received: {{ $json.timestamp }}
  🆔 Lead ID: {{ $json.lead_id }}
  ```
- **⚠️ Note:** By this point in the flow, `$json` refers to the **Google Sheets node's output** (column-cased keys like `Qualification`, `Name`), not the lowercase `qualification`/`name` keys from Parse AI Response. Several expressions here (`$json.name`, `$json.qualification`, `$json.message`, `$json.lead_id`) may resolve to `undefined` depending on how the Google Sheets node echoes field casing. See [Known Issues](#6-known-issues--recommended-fixes).
- **Connects to:** *(end of workflow)*

---

## 4. Full Connection Map

```
Webhook
  └─▶ Normalize Data
        └─▶ Prepare AI Input
              └─▶ AI Agent  ◀── (ai_languageModel) ── OpenRouter Chat Model
                    └─▶ Parse AI Response
                          └─▶ Save Lead (Google Sheets)
                                └─▶ If
                                      ├─ true ─▶ Telegram Notification
                                      └─ false ─▶ (dead end)
```

---

## 5. Credentials Required

| Credential | Node(s) | Type |
|---|---|---|
| OpenRouter account | OpenRouter Chat Model | `openRouterApi` |
| Google Sheets account | Save Lead | `googleSheetsOAuth2Api` |
| Telegram account for AI Journal | Telegram Notification | `telegramApi` |

---

## 6. Known Issues & Recommended Fixes

| # | Issue | Impact | Fix |
|---|---|---|---|
| 1 | `Normalize Data` never sets `lead_id` | `lead_id` is `undefined` throughout the rest of the workflow, including the Sheets row and Telegram message | Add a `lead_id` field in Normalize Data, e.g. `{{ $json.body.email }}_{{ $now.toMillis() }}` |
| 2 | "Lead ID" Google Sheets column is defined in schema but not mapped in `value` | Column always blank | Add `"Lead ID": "={{ $json.lead_id }}"` to the Save Lead column mapping |
| 3 | `If` node checks `$json.Qualification` (capitalized) — depends on exact key casing returned by Google Sheets append | Risk of condition never matching if casing differs, meaning **no Hot leads ever get notified** | Test after a real run; if it fails, either reference `$('Parse AI Response').item.json.qualification` directly, or move the `If` node **before** Save Lead so it reads Parse AI Response output directly |
| 4 | `If` node's `rightValue` is `"=Hot"` | The `=` prefix may be interpreted as an expression rather than a literal string, potentially causing a JS reference error (`Hot is not defined`) | Change to plain `Hot` (no leading `=`) in the UI condition value |
| 5 | Telegram node references `$json.name`, `$json.qualification`, `$json.message`, `$json.lead_id` (lowercase) after the Google Sheets node, which likely returns capitalized keys (`Name`, `Qualification`, `Message`) | Message fields may render blank/undefined in the alert | Reference `$('Parse AI Response').item.json.<field>` explicitly instead of relying on `$json` from Save Lead |
| 6 | Telegram `chatId` is hardcoded to a single user | Not scalable to a team; can't route by lead type or user | Move to an n8n environment variable/credential, or use a Telegram group chat ID |
| 7 | `If` false branch has no connected node | Warm/Cold leads are saved but nothing else happens (silent, expected — but worth confirming it's intentional) | Optionally add a lightweight branch (e.g., tagging or logging) for Warm leads |

**Recommended immediate fix priority:** #3 and #5 first (they directly affect whether Hot-lead alerts actually fire correctly), then #1/#2, then #4, #6, #7.

---

## 7. Testing the Workflow

```bash
curl -X POST https://<your-n8n-instance>/webhook/lead-capture \
  -H "Content-Type: application/json" \
  -d '{
    "name": "John Smith",
    "email": "john@example.com",
    "phone": "+1234567890",
    "company": "Acme Corp",
    "message": "Interested in enterprise pricing. Need solution ASAP for 500 users. Budget approved.",
    "source": "Website Form"
  }'
```

**Verify after running:**
1. New row appended in Google Sheet "qualified lead" → Sheet1
2. Check whether `Qualification` = `Hot` for this sample (strong intent/urgency/budget signals)
3. Confirm Telegram message was received in chat `7350267365`
4. Inspect the message content for any blank/`undefined` fields (see Issue #5 above)

---

## 8. Summary

The workflow is functionally complete for the happy path (lead in → AI qualification → Sheet log), but has a handful of **field-referencing inconsistencies** between the lowercase keys produced by the Code nodes and the capitalized column names used in Google Sheets/Telegram. These should be resolved before production use to guarantee Hot-lead alerts reliably fire with fully populated data.
