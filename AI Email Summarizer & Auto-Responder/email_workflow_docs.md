# AI Email Summarizer & Auto-Responder
## Official Workflow Documentation (Based on Deployed n8n JSON)

> This documentation reflects the **exact, deployed workflow** (`UMkIBdlcx98nkpe9`) exported from n8n — every node, parameter, code block, and connection below matches the actual JSON and workflow screenshot. It supersedes earlier speculative drafts.

---

## Table of Contents
1. [Overview](#overview)
2. [Workflow Diagram](#workflow-diagram)
3. [Node Inventory](#node-inventory)
4. [Detailed Node-by-Node Reference](#detailed-node-by-node-reference)
5. [Connections Map](#connections-map)
6. [Google Sheet Schema](#google-sheet-schema)
7. [End-to-End Data Flow Example](#end-to-end-data-flow-example)
8. [AI Prompt Logic](#ai-prompt-logic)
9. [Known Limitations & Bugs](#known-limitations--bugs)
10. [Credentials Required](#credentials-required)
11. [Testing Guide](#testing-guide)
12. [Troubleshooting](#troubleshooting)
13. [Improvement Recommendations](#improvement-recommendations)

---

## Overview

| Property | Value |
|---|---|
| **Workflow Name** | AI Email Summarizer & Auto-Responder |
| **Workflow ID** | `UMkIBdlcx98nkpe9` |
| **Status** | Inactive (`active: false` in export) |
| **Execution Order** | v1 |
| **Total Nodes** | 14 |
| **AI Provider** | OpenRouter (free tier) |
| **AI Model** | `meta-llama/llama-3.1-405b-instruct:free` |
| **Storage** | Google Sheets ("AI Email Tracker") |
| **Notification Channel** | None (Telegram not used — offline n8n limitation) |
| **Trigger Type** | Gmail Trigger, polling every minute, unread mail only |

### What It Does
1. Polls Gmail every minute for **unread** emails.
2. Parses the raw Gmail payload into clean fields.
3. Sends the email to an AI Agent that **summarizes it AND decides if auto-reply is safe** (`YES`/`NO`) — in a single LLM call with a strict output format.
4. Parses that structured AI output with regex.
5. Passes the parsed data to a **second AI Agent** that drafts the actual reply text.
6. Parses the second AI output with regex.
7. Logs everything to Google Sheets with `Status: Pending Review`.
8. An **IF node** checks the `AutoReply` column value.
   - **TRUE (`YES`)** → re-fetches the row from the sheet → sends the Gmail reply → updates the sheet row to `auto reply sent`.
   - **FALSE (`NO`)** → updates the sheet row to `to be revied manually` (sic).

---

## Workflow Diagram

```
┌────────────┐   ┌──────────────┐   ┌──────────────┐   ┌───────────────┐
│ Gmail      │──▶│ Parse Gmail  │──▶│ Summay of    │──▶│ Parsed        │
│ Trigger    │   │ content      │   │ Mail (Agent) │   │ Summary       │
└────────────┘   └──────────────┘   └──────┬───────┘   └───────┬───────┘
                                      ▲ ai_languageModel         │
                                 ┌────┴─────┐                    ▼
                                 │OpenRouter │            ┌──────────────┐
                                 │Chat Model │            │ Draft reply  │
                                 └──────────┘             │ to be sent   │◀── ai_languageModel ──┐
                                                           │ (Agent)      │                        │
                                                           └──────┬───────┘                 ┌──────┴──────┐
                                                                  ▼                          │OpenRouter    │
                                                           ┌──────────────┐                  │Chat Model1   │
                                                           │ Parsed reply │                  └─────────────┘
                                                           └──────┬───────┘
                                                                  ▼
                                                           ┌──────────────┐
                                                           │ Store data   │
                                                           │ in sheet     │  (append, Status="Pending Review")
                                                           └──────┬───────┘
                                                                  ▼
                                                           ┌──────────────┐
                                                           │ Decide to    │
                                                           │ sent auto-   │
                                                           │ reply or not │ (IF: AutoReply == "YES")
                                                           └──┬────────┬──┘
                                                     TRUE ▼          ▼ FALSE
                                          ┌────────────────────┐   ┌──────────────┐
                                          │ get data from sheet│   │ Update status│
                                          │ to send auto reply │   │ (manual      │
                                          │ (lookup by From/    │   │ review)      │
                                          │  Status/Timestamp) │   └──────────────┘
                                          └─────────┬───────────┘
                                          ┌──────────┴──────────┐
                                          ▼                     ▼
                                   ┌────────────┐       ┌───────────────┐
                                   │ Send Reply │       │ Update status1│
                                   │ (Gmail)    │       │ (auto reply   │
                                   └────────────┘       │  sent)        │
                                                         └───────────────┘
```

---

## Node Inventory

| # | Node Name | Type | Purpose |
|---|---|---|---|
| 1 | **Gmail Trigger** | `n8n-nodes-base.gmailTrigger` | Polls inbox every minute for unread mail |
| 2 | **Parse Gmail content** | `n8n-nodes-base.code` | Extracts structured fields from raw Gmail payload |
| 3 | **Summay of Mail** | `@n8n/n8n-nodes-langchain.agent` | AI Agent — summarizes + decides auto-reply eligibility |
| 4 | **OpenRouter Chat Model** | `@n8n/n8n-nodes-langchain.lmChatOpenRouter` | LLM backing node 3 (maxTokens 300, temp 0.3) |
| 5 | **Parsed Summary** | `n8n-nodes-base.code` | Regex-parses Agent #1's structured text output |
| 6 | **Draft reply to be sent** | `@n8n/n8n-nodes-langchain.agent` | AI Agent — drafts the actual reply text |
| 7 | **OpenRouter Chat Model1** | `@n8n/n8n-nodes-langchain.lmChatOpenRouter` | LLM backing node 6 (default options) |
| 8 | **Parsed reply** | `n8n-nodes-base.code` | Regex-parses Agent #2's structured text output |
| 9 | **Store data in sheet** | `n8n-nodes-base.googleSheets` | Appends new row to Google Sheet |
| 10 | **Decide to sent autoreply or not** | `n8n-nodes-base.if` | Branches on `AutoReply == "YES"` |
| 11 | **get data from sheet to send auto reply** | `n8n-nodes-base.googleSheets` | Re-reads the row (lookup) before sending |
| 12 | **Update status** | `n8n-nodes-base.googleSheets` | Sets Status to manual-review text (FALSE branch) |
| 13 | **Send Reply** | `n8n-nodes-base.gmail` | Sends the actual email reply |
| 14 | **Update status1** | `n8n-nodes-base.googleSheets` | Sets Status to "auto reply sent" (TRUE branch, post-send) |

---

## Detailed Node-by-Node Reference

### 1. Gmail Trigger
**Type:** `n8n-nodes-base.gmailTrigger` (v1.3)

```json
{
  "pollTimes": { "item": [{ "mode": "everyMinute" }] },
  "filters": {
    "includeSpamTrash": false,
    "readStatus": "unread"
  }
}
```

- **Polls every 1 minute** — this is aggressive; consider 5 min in production to conserve API quota.
- Only picks up **unread** mail, spam/trash excluded.
- **Credential:** `Gmail account` (OAuth2, id `7XNYGaBFEcVkLMju`).
- ⚠️ Does **not** mark messages as read afterward — if nothing downstream changes the read status, the same email could be re-picked up on the next poll unless Gmail's own trigger de-duplication (via internal message ID tracking) prevents it. n8n's Gmail Trigger tracks last-seen message internally, so normally this is safe, but worth confirming in testing.

---

### 2. Parse Gmail content
**Type:** `n8n-nodes-base.code` (JavaScript, v2)

```javascript
const items = $input.all();

return items.map(item => {
  const email = item.json;

  return {
    json: {
      emailId: email.id,
      threadId: email.threadId || '',
      
      from: email.From || '',
      fromEmail: email.From?.match(/[\w.-]+@[\w.-]+/)?.[0] || '',
      to: email.To || '',

      subject: email.Subject || 'No Subject',

      // Gmail snippet acts as preview/body
      body: email.snippet || '',

      // Convert Gmail internalDate (ms) → ISO string
      receivedDate: email.internalDate
        ? new Date(Number(email.internalDate)).toISOString()
        : new Date().toISOString(),

      labels: email.labels?.map(l => l.name) || [],

      originalData: email
    }
  };
});
```

**Key points:**
- `body` is set to Gmail's **`snippet`** field — this is a **short preview (~100–200 characters)**, NOT the full email body. This is a significant limitation (see [Known Limitations](#known-limitations--bugs)).
- `from` reads the raw `email.From` header string (e.g. `"John Doe <john@example.com>"`).
- `fromEmail` extracts just the address via regex.
- `receivedDate` converts Gmail's `internalDate` (epoch milliseconds, as string) into ISO 8601.
- `originalData` preserves the full raw trigger payload for potential downstream debugging (not currently used later in the workflow).

---

### 3. Summay of Mail *(AI Agent — Summarizer + Decision Maker)*
**Type:** `@n8n/n8n-nodes-langchain.agent` (v3)  
*(Node name contains a typo — "Summay" instead of "Summary" — preserved as-is from the live workflow)*

**Prompt (`text`, sent as the human/user message):**
```
Email From: {{$json.from}}
Subject: {{$json.subject}}
Body: {{$json.body}}
Timestamp: {{ $json.receivedDate }}
```

**System Message:**
```
You are an expert email analyst.

Your tasks:
1. Summarize the email in 3–5 concise bullet points covering:
   - Main purpose
   - Key requests or questions
   - Urgency level
   - Required action

2. Decide whether this email is SAFE to auto-reply with a neutral acknowledgement.
   - Reply "YES" if the email only requires acknowledgment (e.g., internship application, contact inquiry, information request).
   - Reply "NO" if the email involves decisions, hiring outcomes, complaints, finance, legal matters, or emotional/personal issues.

CRITICAL OUTPUT RULE:
Return the output in the EXACT format below.
Do NOT add extra text, explanations, or markdown outside this structure.

FORMAT:

Email From: <same as input>
Subject: <same as input>
Body: <same as input>
Timestamp: <same as input>

Email Summary:
- <bullet point 1>
- <bullet point 2>
- <bullet point 3>
- <bullet point 4>

Auto Reply Allowed: YES or NO
```

**Connected LLM:** `OpenRouter Chat Model` (via `ai_languageModel` connection, not `main`)

**Design notes:**
- This single AI call does **double duty** — summarization AND the auto-reply eligibility decision — exactly matching the earlier requirement ("AI decides based on how we prompted it, during summarization").
- The decision criteria are intentionally **conservative**: only pure acknowledgment-type emails (applications, contact inquiries, info requests) get `YES`. Anything involving decisions, money, legal, hiring outcomes, or emotional content is `NO`.
- The strict "echo back the input fields" format is a deliberate technique to make downstream regex parsing reliable, since the model re-states `From`, `Subject`, `Body`, `Timestamp` verbatim before adding its analysis.

---

### 4. OpenRouter Chat Model
**Type:** `@n8n/n8n-nodes-langchain.lmChatOpenRouter` (v1)

```json
{
  "model": "meta-llama/llama-3.1-405b-instruct:free",
  "options": {
    "maxTokens": 300,
    "temperature": 0.3
  }
}
```

- **Free model:** Llama 3.1 405B Instruct via OpenRouter's free tier.
- **Low temperature (0.3)** favors consistent, deterministic-style output — appropriate for a structured summarization task.
- **300 max tokens** — enough for a 4-bullet summary plus the echoed input fields, but could truncate on unusually long emails.
- **Credential:** `OpenRouter account for AI Journal` (id `YCawzcwbZd7VllSz`) — note this credential is named for a different project but reused here.
- Connects to node 3 via the `ai_languageModel` connection type (LangChain sub-node pattern), **not** the `main` data pipeline.

---

### 5. Parsed Summary
**Type:** `n8n-nodes-base.code` (JavaScript, v2)

```javascript
const items = $input.all();

return items.map(item => {
  const text = item.json.output || '';

  // ---------- FROM ----------
  const fromMatch = text.match(/Email From:\s*(.*)/i);
  const from = fromMatch ? fromMatch[1].trim() : '';

  // ---------- SUBJECT ----------
  const subjectMatch = text.match(/Subject:\s*(.*)/i);
  const subject = subjectMatch ? subjectMatch[1].trim() : '';

  // ---------- BODY ----------
  const bodyMatch = text.match(
    /Body:\s*([\s\S]*?)(?:\nTimestamp:)/i
  );
  const body = bodyMatch ? bodyMatch[1].trim() : '';

  // ---------- TIMESTAMP ----------
  const timestampMatch = text.match(/Timestamp:\s*(.*)/i);
  const timestamp = timestampMatch ? timestampMatch[1].trim() : '';

  // ---------- SUMMARY ----------
  const summaryMatch = text.match(
    /Email Summary:\s*([\s\S]*?)(?:\nAuto Reply Allowed:|$)/i
  );
  const summary = summaryMatch ? summaryMatch[1].trim() : '';

  // ---------- AUTO REPLY DECISION ----------
  const autoReplyMatch = text.match(/Auto Reply Allowed:\s*(YES|NO)/i);
  const autoReplyAllowed = autoReplyMatch
    ? autoReplyMatch[1].toUpperCase()
    : 'NO';

  return {
    json: {
      from,
      subject,
      body,
      timestamp,
      summary,
      autoReplyAllowed
    }
  };
});
```

**Key points:**
- Reads from `item.json.output` — this is the LangChain Agent node's standard output field.
- Uses regex with lookahead-style boundaries (`(?:\nTimestamp:)`, `(?:\nAuto Reply Allowed:|$)`) to isolate multi-line blocks (`Body`, `Email Summary`).
- **Fail-safe default:** if `Auto Reply Allowed` can't be parsed, it defaults to `'NO'` — a safe, conservative fallback that prevents accidental auto-sends on malformed AI output.

---

### 6. Draft reply to be sent *(AI Agent — Reply Drafter)*
**Type:** `@n8n/n8n-nodes-langchain.agent` (v3)

**Prompt (`text`):**
```
From: {{ $json.from }}
Subject: {{ $json.subject }}
Body: {{ $json.body }}
Summary: {{ $json.summary }}
Timestamp: {{ $json.timestamp }}
AutoReplyAllowed: {{ $json.autoReplyAllowed }}

Draft a professional reply following the rules.
```

**System Message:**
```
You are a professional email assistant.

Your task:
- Draft a polite, concise, and context-aware professional email reply.
- Match the tone of the original email.
- Keep the reply under 150 words unless the topic is complex.


CRITICAL OUTPUT RULE:
Return the output in the EXACT structured format below.
Do NOT add explanations, markdown, headings, or extra text.

FORMAT:
From: <same as input> and the original mail also
Subject: <same as input>
Body: <same as input>
Summary: <same as input>
Timestamp:  <same as input>
AutoReplyAllowed: <same as input>
Draft Reply: <your drafted reply only>
```

**Connected LLM:** `OpenRouter Chat Model1` (via `ai_languageModel`)

**Design notes:**
- Runs **regardless of the `autoReplyAllowed` value** — there is **no IF/filter gate before this node**. A draft is generated for every single email, even ones marked `NO`. The `AutoReplyAllowed` flag is only acted upon much later, at the "Decide to sent autoreply or not" IF node (node 10).
- This means the workflow **always spends AI tokens drafting a reply**, whether or not it will actually be sent — useful because a human reviewer can use the draft as a starting point for manually-handled emails too.

---

### 7. OpenRouter Chat Model1
**Type:** `@n8n/n8n-nodes-langchain.lmChatOpenRouter` (v1)

```json
{
  "model": "meta-llama/llama-3.1-405b-instruct:free",
  "options": {}
}
```

- Same free model as node 4, but **no temperature/token overrides** — uses OpenRouter/LangChain defaults.
- **Credential:** Same `OpenRouter account for AI Journal`.

---

### 8. Parsed reply
**Type:** `n8n-nodes-base.code` (JavaScript, v2)

```javascript
const items = $input.all();

return items.map(item => {
  const text = item.json.output || '';

  // ---------- FROM ----------
  const fromMatch = text.match(/From:\s*(.*)/i);
  const from = fromMatch ? fromMatch[1].trim() : '';

  // ---------- SUBJECT ----------
  const subjectMatch = text.match(/Subject:\s*(.*)/i);
  const subject = subjectMatch ? subjectMatch[1].trim() : '';

  // ---------- BODY ----------
  const bodyMatch = text.match(
    /Body:\s*([\s\S]*?)(?:\nSummary:|\nTimestamp:|\nAutoReplyAllowed:|$)/i
  );
  const body = bodyMatch ? bodyMatch[1].trim() : '';

  // ---------- SUMMARY ----------
  const summaryMatch = text.match(
    /Summary:\s*([\s\S]*?)(?:\nTimestamp:|\nAutoReplyAllowed:|\nDraft Reply:|$)/i
  );
  const summary = summaryMatch ? summaryMatch[1].trim() : '';

  // ---------- TIMESTAMP ----------
  const timestampMatch = text.match(/Timestamp:\s*(.*)/i);
  const timestamp = timestampMatch ? timestampMatch[1].trim() : '';

  // ---------- AUTO REPLY ALLOWED ----------
  const autoReplyMatch = text.match(/AutoReplyAllowed:\s*(YES|NO)/i);
  const autoReplyAllowed = autoReplyMatch
    ? autoReplyMatch[1].toUpperCase()
    : 'NO';

  // ---------- DRAFT REPLY ----------
  const replyMatch = text.match(/Draft Reply:\s*([\s\S]*)$/i);
  const reply = replyMatch ? replyMatch[1].trim() : '';

  return {
    json: {
      from,
      subject,
      body,
      summary,
      timestamp,
      autoReplyAllowed,
      reply
    }
  };
});
```

- Mirrors the parsing pattern of node 5, but extended to also capture the final **`Draft Reply`** block (everything after `Draft Reply:` to end of string).
- Same conservative fallback: unparseable `AutoReplyAllowed` → defaults to `'NO'`.
- Output of this node (`from`, `subject`, `body`, `summary`, `timestamp`, `autoReplyAllowed`, `reply`) is what gets written to Google Sheets next.

---

### 9. Store data in sheet
**Type:** `n8n-nodes-base.googleSheets` (v4.7)

```yaml
Operation: Append
Document: AI Email Tracker
  (ID: 17fTawTaC4SGPCrVqtwYsUpRX4czX1uvFANc52LDDBOo)
Sheet: Sheet1 (gid=0)
```

**Column Mapping:**
| Sheet Column | Value Expression |
|---|---|
| From | `={{ $json.from }}` |
| Subject | `={{ $json.subject }}` |
| Summary | `={{ $json.summary }}` |
| Draft Reply | `={{ $json.reply }}` |
| Timestamp | `={{ $json.timestamp }}` |
| Status | `Pending Review` (static text) |
| AutoReply | `={{ $json.autoReplyAllowed }}` |

- Every processed email — auto-reply eligible or not — is logged with an initial status of **"Pending Review"**, which is later overwritten by node 12 or node 14 depending on the branch taken.
- **Credential:** `Google Sheets account` (id `twmxeUbOfErGsbbM`).

---

### 10. Decide to sent autoreply or not
**Type:** `n8n-nodes-base.if` (v2.2)

```json
{
  "conditions": [
    {
      "leftValue": "={{ $json.AutoReply }}",
      "rightValue": "YES",
      "operator": { "type": "string", "operation": "equals" }
    }
  ],
  "combinator": "and",
  "options": { "caseSensitive": true, "typeValidation": "strict" }
}
```

- Single condition: **`AutoReply` field must exactly equal the string `"YES"`** (case-sensitive, strict type validation).
- Note the field being checked is `$json.AutoReply` (capital-case, matching the **Google Sheets append node's output column name**), not the earlier `autoReplyAllowed` (camelCase) used inside the Code nodes. This is because the input to this IF node is the **output of the "Store data in sheet" node**, which returns Google Sheets row data using the sheet's column headers.
- **TRUE branch** → node 11 (`get data from sheet to send auto reply`)
- **FALSE branch** → node 12 (`Update status`)

---

### 11. get data from sheet to send auto reply *(TRUE branch)*
**Type:** `n8n-nodes-base.googleSheets` (v4.7, Operation: default/read with filters)

**Lookup Filters (all must match):**
| Column | Value |
|---|---|
| From | `={{ $json.From }}` |
| Status | `={{ $json.Status }}` |
| Timestamp | `={{ $json.Timestamp }}` |

- Re-fetches the just-written row from the sheet using From + Status + Timestamp as a composite lookup key, rather than passing data directly from the previous node.
- This re-read guarantees the "Send Reply" node works off the **persisted, sheet-confirmed** version of the data.
- ⚠️ Because `Status` is included as a match filter and Status was just written as `"Pending Review"`, this lookup will only succeed if no other process has changed the Status in between — see [Known Limitations](#known-limitations--bugs).

---

### 12. Update status *(FALSE branch — manual review)*
**Type:** `n8n-nodes-base.googleSheets` (v4.7, Operation: Update)

```yaml
Matching Column: Timestamp
Update Values:
  Status: "to be revied manually"   # [sic] — typo preserved from source
  Timestamp: ={{ $json.Timestamp }}
```

- Updates the just-created row (matched by `Timestamp`) to flag it for human review.
- Note the literal typo `"to be revied manually"` in the deployed workflow (should read "reviewed").

---

### 13. Send Reply
**Type:** `n8n-nodes-base.gmail` (v2.1)

```yaml
Send To: ={{ $json.From }}
Subject: ={{ $json.Subject }}
Message: ={{ $json['Draft Reply'] }}
Options:
  appendAttribution: false
```

- Sends via the same `Gmail account` OAuth2 credential used by the trigger.
- **No explicit threading parameters** (`inReplyTo`, `threadId`, `references` are not set) — meaning replies will likely be sent as **new emails**, not threaded replies. This is a known gap (see limitations).
- `appendAttribution: false` disables n8n's default "Sent via n8n" footer.
- This node has an empty `main` output connection in the JSON (`"Send Reply": { "main": [[]] }`) — it is a terminal node in this branch.

---

### 14. Update status1 *(TRUE branch — after send)*
**Type:** `n8n-nodes-base.googleSheets` (v4.7, Operation: `appendOrUpdate`)

```yaml
Matching Column: Timestamp
Update Values:
  Status: "auto reply sent"
  Timestamp: ={{ $json.Timestamp }}
```

- Runs **in parallel with `Send Reply`** (both are fed directly from node 11's output — see connections below), not strictly after confirmation the email was sent.
- Uses `appendOrUpdate`, so if the `Timestamp` match fails to find an existing row, it will insert a new one instead of erroring.

---

## Connections Map

Exactly as defined in the workflow JSON:

```
Gmail Trigger → Parse Gmail content
Parse Gmail content → Summay of Mail
OpenRouter Chat Model → (ai_languageModel) → Summay of Mail
Summay of Mail → Parsed Summary
Parsed Summary → Draft reply to be sent
OpenRouter Chat Model1 → (ai_languageModel) → Draft reply to be sent
Draft reply to be sent → Parsed reply
Parsed reply → Store data in sheet
Store data in sheet → Decide to sent autoreply or not
Decide to sent autoreply or not → [TRUE]  → get data from sheet to send auto reply
Decide to sent autoreply or not → [FALSE] → Update status
get data from sheet to send auto reply → Send Reply           (parallel branch 1)
get data from sheet to send auto reply → Update status1       (parallel branch 2)
Send Reply → (nothing — terminal)
```

**Important:** `Send Reply` and `Update status1` both fire **simultaneously** off of node 11's output — they are not sequential. This means the sheet status could theoretically update to "auto reply sent" even in the rare case that `Send Reply` fails, since n8n doesn't gate one on the other's success by default.

---

## Google Sheet Schema

**Spreadsheet:** `AI Email Tracker`  
**Sheet:** `Sheet1` (gid=0)  
**URL:** `https://docs.google.com/spreadsheets/d/17fTawTaC4SGPCrVqtwYsUpRX4czX1uvFANc52LDDBOo`

| Column | Type | Populated By | Possible Values |
|---|---|---|---|
| Timestamp | String | Store data in sheet | ISO date string (from AI-echoed timestamp) |
| From | String | Store data in sheet | Raw `From` header text |
| Subject | String | Store data in sheet | Email subject |
| Summary | String | Store data in sheet | AI-generated bullet summary |
| Draft Reply | String | Store data in sheet | AI-drafted reply text |
| Status | String | Store data in sheet → Update status / Update status1 | `Pending Review` → `to be revied manually` **or** `auto reply sent` |
| AutoReply | String | Store data in sheet | `YES` / `NO` |

**Status Lifecycle:**
```
Pending Review ──┬─(AutoReply=YES)──▶ auto reply sent
                 └─(AutoReply=NO)───▶ to be revied manually
```

---

## End-to-End Data Flow Example

**1. Incoming Gmail Trigger payload (simplified):**
```json
{
  "id": "18d1234567890",
  "threadId": "18d1234567890",
  "From": "Priya Sharma <priya@example.com>",
  "To": "me@myemail.com",
  "Subject": "Internship Application - Marketing",
  "snippet": "Hi, I'm writing to apply for the marketing internship...",
  "internalDate": "1709374215000"
}
```

**2. After "Parse Gmail content":**
```json
{
  "emailId": "18d1234567890",
  "from": "Priya Sharma <priya@example.com>",
  "fromEmail": "priya@example.com",
  "subject": "Internship Application - Marketing",
  "body": "Hi, I'm writing to apply for the marketing internship...",
  "receivedDate": "2026-03-02T10:30:15.000Z"
}
```

**3. Raw AI output from "Summay of Mail":**
```
Email From: Priya Sharma <priya@example.com>
Subject: Internship Application - Marketing
Body: Hi, I'm writing to apply for the marketing internship...
Timestamp: 2026-03-02T10:30:15.000Z

Email Summary:
- Candidate applying for marketing internship
- Requests confirmation of application receipt
- Urgency: Low
- Required action: Acknowledge receipt

Auto Reply Allowed: YES
```

**4. After "Parsed Summary":**
```json
{
  "from": "Priya Sharma <priya@example.com>",
  "subject": "Internship Application - Marketing",
  "body": "Hi, I'm writing to apply...",
  "timestamp": "2026-03-02T10:30:15.000Z",
  "summary": "- Candidate applying...\n- Requests confirmation...",
  "autoReplyAllowed": "YES"
}
```

**5. Raw AI output from "Draft reply to be sent":**
```
From: Priya Sharma <priya@example.com>
Subject: Internship Application - Marketing
Body: Hi, I'm writing to apply...
Summary: - Candidate applying...
Timestamp: 2026-03-02T10:30:15.000Z
AutoReplyAllowed: YES
Draft Reply: Hi Priya, Thank you for applying to our marketing internship program. We've received your application and our team will review it shortly. We appreciate your interest and will be in touch regarding next steps. Best regards, The Hiring Team
```

**6. After "Parsed reply" → row written to sheet:**
| Timestamp | From | Subject | Summary | Draft Reply | Status | AutoReply |
|---|---|---|---|---|---|---|
| 2026-03-02T10:30:15.000Z | Priya Sharma \<priya@example.com\> | Internship Application - Marketing | - Candidate applying... | Hi Priya, Thank you... | Pending Review | YES |

**7. IF node evaluates `AutoReply == "YES"` → TRUE → email sent → Status updated to `auto reply sent`.**

---

## AI Prompt Logic

### Auto-Reply Eligibility Criteria (as coded into the system prompt)

| Decision | Trigger Conditions |
|---|---|
| **YES** (auto-reply safe) | Email only needs acknowledgment — e.g., internship applications, general contact inquiries, information requests |
| **NO** (manual review needed) | Email involves decisions, hiring outcomes, complaints, finance, legal matters, or emotional/personal issues |

This is a **binary, conservative filter** — by design, it defaults toward caution (`NO`) both in the prompt instructions and in the regex fallback (`autoReplyAllowed` defaults to `'NO'` if parsing fails).

### Reply Drafting Constraints (as coded into the system prompt)
- Polite, concise, context-aware, professional tone
- Match the tone of the original email
- Under 150 words unless the topic is complex
- Strict output format, no markdown, no explanations

---

## Known Limitations & Bugs

| # | Issue | Impact | Suggested Fix |
|---|---|---|---|
| 1 | **`body` uses Gmail `snippet`, not full message text** | AI only sees a short preview (~100-200 chars) of the email, not the full content — summaries and drafts may miss context from longer emails | Use the Gmail node's "Get" operation or enable full body retrieval in the trigger, extracting `textPlain`/`payload.body` instead of `snippet` |
| 2 | **No email threading on send** | `Send Reply` doesn't set `threadId`/`inReplyTo`, so replies likely appear as new emails rather than threaded replies | Pass `threadId` from Parse Gmail content through to Send Reply and set it in the Gmail node's thread options |
| 3 | **Draft is always generated, even for `NO` emails** | Wastes an LLM call on every manual-review email (acceptable trade-off since it gives reviewers a starting draft, but worth knowing) | Optional: add an IF gate before node 6 if minimizing AI calls is a priority |
| 4 | **`Send Reply` and `Update status1` run in parallel, not sequentially** | Sheet could show "auto reply sent" even if the email actually failed to send | Chain `Update status1` after `Send Reply`'s output instead of branching both off node 11 |
| 5 | **Lookup in node 11 filters on `Status = "Pending Review"`** | If two emails are processed close together or the status changes between write and read, the lookup could return 0 or multiple rows | Use a unique ID (e.g., `emailId`) as the sole lookup key instead of a compound From/Status/Timestamp filter |
| 6 | **1-minute polling interval** | More aggressive than necessary; increases Gmail/OpenRouter API call volume | Increase to 5 minutes unless near-real-time response is required |
| 7 | **Typos in live workflow** | `"Summay of Mail"`, `"to be revied manually"` | Cosmetic only — rename nodes/text for clarity if desired |
| 8 | **No error handling / retry logic on any node** | A single API failure (Gmail, OpenRouter, or Sheets) will halt that execution with no fallback | Add `Continue On Fail` + error branch, or n8n Error Trigger workflow |
| 9 | **No Telegram or other manual-review notification** | Emails flagged "to be revied manually" require the user to manually check the sheet | Confirmed as an accepted limitation (offline n8n could not integrate Telegram per user) |

---

## Credentials Required

| Credential | Type | Used By Nodes |
|---|---|---|
| **Gmail account** | Gmail OAuth2 | Gmail Trigger, Send Reply |
| **OpenRouter account for AI Journal** | OpenRouter API | OpenRouter Chat Model, OpenRouter Chat Model1 |
| **Google Sheets account** | Google Sheets OAuth2 | Store data in sheet, get data from sheet to send auto reply, Update status, Update status1 |

---

## Testing Guide

### Test Case 1 — Should Auto-Reply
```
Subject: Quick question about your services
Body: Hi, could you send me more info about your pricing plans?
Expected: AutoReply = YES → Status ends as "auto reply sent"
```

### Test Case 2 — Should Require Manual Review
```
Subject: Formal Complaint Regarding Delayed Order
Body: I am very disappointed with the delay and would like a refund.
Expected: AutoReply = NO → Status ends as "to be revied manually"
```

### Test Case 3 — Hiring Decision (should be NO)
```
Subject: Interview Outcome
Body: We'd like to discuss the offer terms for the Marketing Manager role.
Expected: AutoReply = NO (involves decisions/hiring outcome)
```

### Verification Checklist
```
□ New row appears in "AI Email Tracker" sheet for every test email
□ Summary column contains 3–5 bullet points
□ Draft Reply column contains a complete, on-topic draft
□ AutoReply column correctly reflects YES/NO per the criteria
□ Status transitions correctly based on AutoReply value
□ For YES cases: actual email is received in test inbox
□ For NO cases: no email is sent, only sheet status updates
```

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---|---|---|
| Workflow doesn't trigger | Gmail OAuth expired, or workflow is inactive | Check `active` flag; re-authenticate Gmail credential |
| `Parsed Summary` / `Parsed reply` fields are empty | AI didn't follow the strict output format (model drift, or hit `maxTokens` cap before finishing) | Increase `maxTokens`, review raw `output` field in node's execution data, tighten system prompt |
| `AutoReply` always `NO` even for simple emails | Regex parsing failure defaulting to `'NO'`, or AI misjudging criteria | Check raw Agent output for exact `Auto Reply Allowed:` line formatting |
| Node 11 (`get data from sheet...`) returns no rows | Status was already changed, or Timestamp format mismatch between write and lookup | Switch lookup key to a stable unique ID (e.g. `emailId`) instead of compound match |
| Reply sent as new email, not in thread | No `threadId`/`inReplyTo` set on Send Reply node | Pass `threadId` through the pipeline and configure it in the Gmail Send node |
| Duplicate rows in sheet for same email | Gmail Trigger re-polling same message | Verify trigger de-dupe behavior; consider marking messages read after processing |

---

## Improvement Recommendations

**Priority 1 (Data Quality):**
1. Replace `snippet`-based body extraction with full email body text.
2. Use `emailId` as the canonical lookup/match key across all Google Sheets nodes instead of Timestamp/From/Status combinations.

**Priority 2 (Reliability):**
3. Chain `Update status1` strictly after `Send Reply` succeeds (sequential, not parallel).
4. Add error handling (`Continue On Fail` + logging) on all API-calling nodes (Gmail, OpenRouter, Sheets).

**Priority 3 (UX/Correctness):**
5. Set `threadId`/`inReplyTo` on `Send Reply` so replies thread properly in Gmail.
6. Fix the `"to be revied manually"` typo.
7. Consider reducing polling frequency from every 1 minute to every 5 minutes.

**Priority 4 (Future Enhancements):**
8. Add a notification channel for "Pending Review" items once Telegram (or email digest) becomes available in the hosting environment.
9. Add sender-domain-based rules (trusted vs. unknown) to sharpen the auto-reply decision beyond content-only analysis.

---

*Documentation generated from workflow export `versionId: de11a941-6793-453f-930b-a95bc7a0cbe2`, `instanceId: bc74f12774e3871a4e36f0d67c6934d039e29c5abcb25a6144d9e7b521426c15`.*
