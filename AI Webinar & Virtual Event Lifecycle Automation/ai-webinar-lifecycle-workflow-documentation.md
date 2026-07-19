# AI Webinar & Virtual Event Lifecycle Automation — Workflow Documentation

| Field | Value |
|---|---|
| **Workflow Name** | AI Webinar & Virtual Event Lifecycle Automation |
| **Workflow ID** | `0ALdOUbkuIAHy0aS` |
| **Internal Version ID** | `15f6324f-4ef2-48b2-83bc-c57ecfd97412` _(n8n internal hash, not a semantic version)_ |
| **Human Version** | v1.0 (initial documentation) |
| **Owner** | _[To be provided by workflow owner]_ |
| **Workflow Last Saved** | _[Not present in export metadata]_ |
| **Document Generated** | 2026-07-17 |
| **Active** | ❌ No — must be manually activated before production use |
| **Execution Order** | v1 (connection-based) |
| **Node Count** | 31 nodes |

---

## Purpose

This workflow automates the complete lifecycle of a virtual webinar or online event — from initial registration through post-event follow-up — using only free tools. It handles timed email reminders, ingests attendance data after the event ends, classifies each attendee into a lead segment using an AI agent, dispatches personalised follow-up emails, alerts the sales team about hot leads via Slack, distributes an AI-generated replay email to all registrants, and writes the final lead state back to a Google Sheets CRM.

---

## Business Use Case

Replaces a manual post-webinar workflow (or a paid stack of ActiveCampaign + HubSpot + Zoom Webinars Pro) with a zero-cost automated pipeline. The intended use case is B2B SaaS or consulting teams that run regular educational webinars and want to convert attendees into sales pipeline without manual lead scoring or email authoring.

> ⚠️ _Inferred from workflow structure — please confirm or replace with the actual business context._

---

## Workflow Summary

When someone registers for a webinar, a POST request hits the first webhook and the registrant's data is normalised, written to a Google Sheets CRM, and acknowledged with a confirmation email. The workflow then suspends itself twice using `Wait` nodes, sending a 24-hour reminder and a 1-hour reminder at precisely calculated times relative to the event start. 

After the event ends, an operator POSTs an attendance export to the second webhook. Each attendee record is split into its own execution item, enriched with a computed `watch_percentage` and `engagement_score`, merged with the original registration record from Sheets, and validated. A LangChain AI Agent backed by an OpenRouter model then scores each person as HOT, WARM, or COLD and drafts a personalised email body. A Switch node fans out to three Gmail nodes that send segment-appropriate emails. HOT leads above a 0.7 confidence threshold additionally trigger a Slack alert to the sales team.

Two hours later a second AI Agent generates a webinar summary package, which is assembled into a replay email and dispatched to every registrant read from the Sheets tab. Finally, each row in the CRM is updated with segment, scores, and sent-flags, and a digest message is posted to Slack.

---

## Workflow Diagram

See the attached canvas screenshot (`AI_Webinar___Virtual_Event_Lifecycle_Automation.png`). The workflow is a **two-entry-point, fan-out-then-converge** shape:

- **Top row** (left → right): Registration path — Webhook → Set → Sheets Append → Gmail Confirm → Code → Wait (24 h) → Gmail → Wait (1 h) → Gmail
- **Middle row**: Post-event attendance path — Webhook → Code → Sheets Lookup / Merge → If → AI Agent → Code → Switch → three Gmail branches → If1 → Slack
- **Bottom row**: Replay & CRM path — Wait1 → AI Agent → Code → Sheets Read → Gmail Replay → Sheets Update → Slack Digest

The two entry paths are **independent executions** that share no runtime state; they communicate only through the Google Sheets CRM rows.

```
[Webhook]──►[Set Registration Fields]──►[Sheets: Append]──►[Gmail: Confirm]
                                                                    │
                                               [Code: Calc Datetime]◄┘
                                                        │
                                              [Wait: 24 h before]
                                                        │
                                            [Gmail: 24-Hour Reminder]
                                                        │
                                              [Wait: 1 h before]
                                                        │
                                            [Gmail: 1-Hour Reminder]  ← END path 1


[Webhook: Attendance]──►[Code: Split]──►[Sheets: Lookup]──►[Merge]──►[If (validate)]
                                   └──────────────────────────►[Merge]
                                                                  │ true
                                                           [AI Agent]──◄──[OpenRouter Chat Model]
                                                                  │
                                                     [Code in JavaScript]
                                                                  │
                                                             [Switch]
                                                   ┌───────────┬┴──────────┐
                                             [Gmail:HOT]  [Gmail:WARM] [Gmail:COLD]
                                                   │
                                               [If1 (confidence>0.7)]
                                                   │ true
                                          [Slack: Hot Lead Alert]
                                                   │
                                               [Wait1 (2 h)]
                                                   │
                             [Generate AI Webinar Summary]──◄──[OpenRouter Chat Model1]
                                                   │
                                    [Code: Parse Summary Response]
                                                   │
                                  [Sheets: Read All Registrants]
                                                   │
                                      [Gmail: Send Replay Email]
                                                   │
                                  [Sheets: Update CRM Row]
                                                   │
                                  [Slack: Daily Digest Summary]  ← END path 2
```

---

## Trigger Details

This workflow has **two independent webhook triggers**.

### Trigger 1 — Webhook (Registration)

| Parameter | Value |
|---|---|
| Node name | `Webhook` |
| HTTP method | `POST` |
| Path | `webinar-register` |
| Full URL (production) | `https://<n8n-host>/webhook/webinar-register` |
| Full URL (test) | `https://<n8n-host>/webhook-test/webinar-register` |
| Response mode | `lastNode` — responds after the full chain executes |
| Authentication | None configured |

**Expected payload fields:**

| Field | Required | Description |
|---|---|---|
| `name` | Yes | Registrant full name |
| `email` | Yes | Email address |
| `company` | No | Company name (defaults to `Unknown`) |
| `phone` | No | Phone number |
| `event` | Yes | Event name string |
| `event_date` | Yes | Date in `YYYY-MM-DD` format |
| `event_time` | Yes | Time in `HH:MM` (24-hour) |
| `timezone` | No | IANA timezone string (defaults to `UTC`) |
| `source` | No | Lead source tag (defaults to `direct`) |

### Trigger 2 — Webhook: Attendance Data Intake

| Parameter | Value |
|---|---|
| Node name | `Webhook: Attendance Data Intake` |
| HTTP method | `POST` |
| Path | `webinar-attendance` |
| Full URL (production) | `https://<n8n-host>/webhook/webinar-attendance` |
| Response mode | `lastNode` |
| Authentication | None configured |

**Expected payload structure:**

```json
{
  "event_name": "AI Automation Webinar",
  "event_date": "2025-02-15",
  "event_duration_minutes": 60,
  "attendees": [
    {
      "email": "jane@company.com",
      "status": "attended",
      "duration_minutes": 57,
      "questions_asked": 2,
      "poll_responses": 3,
      "chat_messages": 4
    }
  ]
}
```

> ⚠️ **Neither webhook has authentication configured.** Any party with the URL can trigger these endpoints. See [Security Considerations](#security-considerations).

---

## Execution Flow

### Path 1 — Registration (triggered by Webhook)

1. **Webhook** receives a POST to `/webinar-register` and passes `$json.body` downstream.
2. **Set Registration Fields** normalises all incoming fields: trims whitespace, lowercases the email, sets defaults (`Unknown`, `UTC`, `direct`), generates a `registrant_id` in the format `REG-YYYYMMDD-XXXXXX`, and stamps static default values for `registration_status = registered`, `attendance_status = pending`, `segment = unprocessed`.
3. **Google Sheets: Append Registrant** appends a new row to the `Registrants` sheet in the configured spreadsheet, writing 14 mapped columns.
4. **Gmail: Confirmation Email** sends an immediate plain-text confirmation to the hardcoded test address _(see Known Limitations — this must be updated to `={{ $json.email }}`)_.
5. **Code: Calculate Event Datetime** runs JavaScript that converts `event_date` + `event_time` into Unix milliseconds and computes two ISO timestamps: `reminder_24h_at` (24 h before the event) and `reminder_1h_at` (1 h before). Both use `Math.max(0, ...)` so they never go negative if the event is already past.
6. **Wait** suspends until `reminder_24h_at` using `resume: specificTime`.
7. **Gmail: 24-Hour Reminder** sends a reminder email referencing data from `$('Google Sheets: Append Registrant').item.json`.
8. **Wait: 1h Reminder Delay** suspends until `reminder_1h_at` (from `Code: Calculate Event Datetime`).
9. **Gmail: 1-Hour Reminder** sends a final urgent reminder with a static join link. ← **Terminal node for Path 1.**

### Path 2 — Post-Event Processing (triggered by Webhook: Attendance Data Intake)

10. **Webhook: Attendance Data Intake** receives a POST to `/webinar-attendance` containing the full attendees array.
11. **Code: Split Attendees Array** explodes the `body.attendees` array into individual n8n items; for each it computes:
    - `watch_percentage = round((duration_minutes / event_duration_minutes) × 100)`
    - `engagement_score = (questions_asked × 10) + (poll_responses × 5) + (chat_messages × 2) + watch_percentage`
12. Each item is routed to **two nodes simultaneously**:
    - **Google Sheets: Lookup Registrant** → Merge input 0
    - **Merge** → input 1 (attendance data directly)
13. **Merge** combines the registration record (from Sheets) with the attendance item, joined on the `email` field using `combine` + `advanced` + `mergeByFields` mode.
14. **If** validates the merged record: `email` must be non-empty AND `attendance_status` must match the regex `^(attended|no-show|partial)$`. Records failing this go to the `false` (output 1) branch, which has **no connected node** — they are silently dropped (see Known Limitations).
15. **AI Agent** (backed by **OpenRouter Chat Model**) receives a structured prompt containing all attendee metrics and is asked to return a strict JSON object classifying the lead as `HOT`, `WARM`, or `COLD` with a reason, confidence score, email tone, key signal, and a draft email body.
16. **Code in JavaScript** parses the AI Agent's text output (`item.json.output`), strips any markdown fences, and JSON-parses the result. On parse failure it falls back to `segment: 'WARM'`. It merges the extracted fields (`segment`, `segment_reason`, `confidence`, `email_tone`, `key_signal`, `ai_email_body`) onto the existing item.
17. **Switch** routes on `$json.segment`:
    - Output 0 (`segment === "HOT"`) → **Gmail: HOT Lead Follow-up**
    - Output 1 (`segment === "WARM"`) → **Gmail: WARM Lead Nurture**
    - Output 2 (`segment === "COLD"`) → **Gmail: COLD Lead Re-engagement**
18. **Gmail: HOT Lead Follow-up** sends a sales-pitch HTML email referencing company, watch percentage, and the AI-drafted body. It then passes to **If1**.
19. **If1** checks `$('Code in JavaScript').item.json.confidence > 0.7`. If true:
    - **Slack: Hot Lead Alert** posts a formatted message to the `slackbot` DM _(intended to be a channel — see Known Limitations)_.
    - Execution then continues to **Wait1**.
    - If false: execution **ends** (no continuation node on the false branch — replay distribution is only triggered for HOT leads above the confidence threshold).
20. **Wait1** pauses for 2 hours (`amount: 2`, default unit = hours).
21. **Generate AI Webinar Summary** (backed by **OpenRouter Chat Model1**) is prompted to produce a replay email package (subject, preview text, headline, summary paragraph, three takeaways, CTA text, P.S. line) in JSON format, with the system role as a B2B SaaS content writer.
22. **Code: Parse Summary Response** attempts to parse the AI output from `item.json.choices[0].message.content` _(see Known Limitations — this path is incorrect for n8n AI Agent output; the correct key is `item.json.output`)_. Falls back to static copy if parsing fails.
23. **Google Sheets: Read All Registrants** reads rows from the `Registrants` sheet filtered by `event_name` and `replay_sent`.
24. **Gmail: Send Replay Email** sends the AI-generated replay email to each registrant row, using the summary fields via `$('Code: Parse Summary Response').item.json.*` cross-node references.
25. **Google Sheets: Update CRM Row** updates each registrant row with post-event data (attendance status, segment, scores, sent flags) matched on `email`.
26. **Slack: Daily Digest Summary** posts a completion message to Slackbot. ← **Terminal node for Path 2.**

---

## Node List

| # | Node Name | Type | Version | Purpose | Disabled |
|---|---|---|---|---|---|
| 1 | Webhook | Webhook | 2.1 | Registration entry point — POST `/webinar-register` | No |
| 2 | Set Registration Fields | Set | 3.4 | Normalise fields, generate ID, set defaults | No |
| 3 | Google Sheets: Append Registrant | Google Sheets | 4.7 | Write new registrant row to CRM | No |
| 4 | Gmail: Confirmation Email | Gmail | 2.1 | Send immediate registration confirmation | No |
| 5 | Code: Calculate Event Datetime | Code | 2 | Compute 24 h and 1 h reminder ISO timestamps | No |
| 6 | Wait | Wait | 1.1 | Suspend until 24 h before event | No |
| 7 | Gmail: 24-Hour Reminder | Gmail | 2.1 | Send 24 h reminder email | No |
| 8 | Wait: 1h Reminder Delay | Wait | 1.1 | Suspend until 1 h before event | No |
| 9 | Gmail: 1-Hour Reminder | Gmail | 2.1 | Send 1 h urgent reminder with join link | No |
| 10 | Webhook: Attendance Data Intake | Webhook | 2.1 | Attendance ingestion entry point — POST `/webinar-attendance` | No |
| 11 | Code: Split Attendees Array | Code | 2 | Explode attendees array; compute watch % and engagement score | No |
| 12 | Google Sheets: Lookup Registrant | Google Sheets | 4.7 | Fetch original registration row by email | No |
| 13 | Merge | Merge | 3.2 | Combine registration + attendance data on `email` | No |
| 14 | If | If | 2.2 | Validate email non-empty + status regex | No |
| 15 | AI Agent | LangChain Agent | 3 | Score attendee HOT/WARM/COLD; draft personalised email | No |
| 16 | OpenRouter Chat Model | LangChain OpenRouter | 1 | LLM provider for AI Agent — `nvidia/nemotron-nano-9b-v2:free` | No |
| 17 | Code in JavaScript | Code | 2 | Parse AI Agent JSON output; apply fallback | No |
| 18 | Switch | Switch | 3.3 | Route by `segment` to HOT / WARM / COLD email branch | No |
| 19 | Gmail: HOT Lead Follow-up | Gmail | 2.1 | Sales-pitch email for HOT leads | No |
| 20 | Gmail: WARM Lead Nurture | Gmail | 2.1 | Value-first nurture email for WARM leads | No |
| 21 | Gmail: COLD Lead Re-engagement | Gmail | 2.1 | Replay link re-engagement email for COLD leads | No |
| 22 | If1 | If | 2.2 | Gate Slack alert on `confidence > 0.7` | No |
| 23 | Slack: Hot Lead Alert | Slack | 2.3 | Post HOT lead card to Slack | No |
| 24 | Wait1 | Wait | 1.1 | Pause 2 hours before replay distribution | No |
| 25 | Generate AI Webinar Summary | LangChain Agent | 3 | Generate replay email copy package | No |
| 26 | OpenRouter Chat Model1 | LangChain OpenRouter | 1 | LLM provider for summary agent — `openrouter/free` | No |
| 27 | Code: Parse Summary Response | Code | 2 | Parse summary AI JSON; apply fallback | No |
| 28 | Google Sheets: Read All Registrants | Google Sheets | 4.7 | Fetch all registrant emails for replay blast | No |
| 29 | Gmail: Send Replay Email | Gmail | 2.1 | Send AI-generated replay email to all registrants | No |
| 30 | Google Sheets: Update CRM Row | Google Sheets | 4.7 | Write final segment, scores, and flags back to CRM | No |
| 31 | Slack: Daily Digest Summary | Slack | 2.3 | Post post-event completion digest | No |

---

## Node-by-Node Description

### Webhook
Entry point for the registration path. Listens for `POST /webinar-register`. `responseMode: lastNode` means the HTTP response is held open until the terminal node of the chain completes, then the final output is returned as the response body. No authentication is configured.

### Set Registration Fields
Runs 16 field assignments. Normalises `email` to lowercase/trimmed, `name` to trimmed, applies `|| 'Unknown'` / `|| ''` defaults for optional fields, and generates a unique `registrant_id` using a combination of the current date (via `$now.toFormat('yyyyMMdd')`) and a 6-character random alphanumeric suffix. Sets static defaults: `registration_status = registered`, `attendance_status = pending`, `segment = unprocessed`, `follow_up_sent = false`, `replay_sent = false`.

**Key configuration — all 16 field assignments:**

| Field | Expression / Value |
|---|---|
| `registrant_id` | `'REG-' + $now.toFormat('yyyyMMdd') + '-' + Math.random().toString(36).substring(2,8).toUpperCase()` |
| `name` | `$json.body.name.trim()` |
| `email` | `$json.body.email.toLowerCase().trim()` |
| `company` | `$json.body.company \|\| 'Unknown'` |
| `phone` | `$json.body.phone \|\| ''` |
| `event_name` | `$json.body.event` |
| `event_date` | `$json.body.event_date` |
| `event_time` | `$json.body.event_time` |
| `timezone` | `$json.body.timezone \|\| 'UTC'` |
| `source` | `$json.body.source \|\| 'direct'` |
| `registration_status` | `registered` (static) |
| `attendance_status` | `pending` (static) |
| `segment` | `unprocessed` (static) |
| `registered_at` | `$now.toISO()` |
| `follow_up_sent` | `false` (static) |
| `replay_sent` | `false` (static) |

### Google Sheets: Append Registrant
Appends one row per registration to the `Registrants` sheet (sheet tab ID `gid=0`) of spreadsheet `1avIj_F3leYjeplBNhDJuNV4j9KevaEtrgbYtJrPnamQ`. Mapped columns: `registrant_id`, `name`, `phone`, `event_date`, `event_time`, `timezone`, `source`, `registration_status`, `attendance_status`, `registered_at`, `email`, `company`, `event_name`, `segment`. Uses `googleSheetsOAuth2Api` credential.

### Gmail: Confirmation Email
Sends a plain-text confirmation immediately after the Sheets append. Subject: `You're registered! {{ $json.event_name }} – {{ $json.event_date }}`. Body includes registrant name, event details, their `registrant_id`, and a note about upcoming reminders. Uses `gmailOAuth2` credential.
> ⚠️ **`sendTo` is hardcoded to `storysnippents@gmail.com`** — this must be changed to `={{ $json.email }}` before production.

### Code: Calculate Event Datetime
JavaScript code node (22 lines). Reads `event_date`, `event_time`, and `timezone` from `items[0].json`, constructs an ISO datetime string, computes Unix ms for the event, then calculates:
- `reminder_24h_at = new Date(eventMs - 86_400_000).toISOString()`
- `reminder_1h_at  = new Date(eventMs - 3_600_000).toISOString()`

Both deltas are floored at 0 via `Math.max(0, ...)`. Returns all original fields plus the two new timestamp fields.

### Wait
Suspends the execution until the ISO datetime stored in `$json.reminder_24h_at`. Uses `resume: specificTime`. n8n persists the paused execution state to disk; no CPU or memory is consumed during the wait.

### Gmail: 24-Hour Reminder
Sends a reminder email 24 h before the event. References event details via the cross-node expression `$('Google Sheets: Append Registrant').item.json.*` since those fields are not available on the current item at this point in the chain. Plain-text body.

### Wait: 1h Reminder Delay
Suspends until `$('Code: Calculate Event Datetime').item.json.reminder_1h_at`.

### Gmail: 1-Hour Reminder
Final urgent reminder including a static join link (`https://your-webinar-link.com/join`) that **must be updated** before production. Also mentions the replay availability.

### Webhook: Attendance Data Intake
Second independent entry point. Listens for `POST /webinar-attendance`. Expects a JSON body with a top-level `attendees` array. No authentication configured.

### Code: Split Attendees Array
Reads `items[0].json.body.attendees` (14 lines of code). For each element computes `watch_percentage` and `engagement_score` and returns one n8n item per attendee. Default values of `0` are applied for all numeric fields when absent. Emits: `email`, `attendance_status`, `join_time`, `leave_time`, `duration_minutes`, `questions_asked`, `poll_responses`, `chat_messages`, `event_name`, `event_date`, `event_duration_minutes`, `watch_percentage`, `engagement_score`.

**Engagement score formula:**
```
engagement_score = (questions_asked × 10) + (poll_responses × 5) + (chat_messages × 2) + watch_percentage
```

### Google Sheets: Lookup Registrant
For each attendee item, reads the `Registrants` sheet and looks up the row whose `email` matches the attendee's email. This provides the original registration fields (`name`, `company`, `source`, `registrant_id`, etc.) for the merge. Uses `googleSheetsOAuth2Api`.

### Merge
Combines data from two inputs using `combine` + `advanced` mode, joining on `email` field from both sides. The registration data comes in on **input 0** (from Sheets Lookup) and the computed attendance data on **input 1** (direct from Code: Split Attendees Array).

> **Note:** The `advanced` join mode with `mergeByFields` retains only matched rows by default. Attendees whose email is not found in the Registrants sheet will be **dropped silently**.

### If
Validates the merged record before allowing it into the AI pipeline. Condition uses `AND` combinator:
1. `$json.email` is not empty (string, `notEmpty` operator)
2. `$json.attendance_status` matches regex `^(attended|no-show|partial)$`

**True branch (output 0):** valid records → AI Agent  
**False branch (output 1):** invalid records → **not connected** (silent drop — see Known Limitations)

### AI Agent
LangChain Agent node configured with `promptType: define`. The prompt is fully built from n8n expressions injecting attendee data. It instructs the model to return a strict JSON object only. Connected sub-nodes:
- **OpenRouter Chat Model** supplies the language model on the `ai_languageModel` channel.

**Prompt summary:** Provides all attendee metrics (email, name, company, status, watch %, duration, questions, polls, chat, engagement score, source) and three explicit segmentation rules:
- **HOT:** attended AND (watch_percentage ≥ 80 OR engagement_score ≥ 90)
- **WARM:** attended AND watch_percentage 30–79, or attended with low engagement
- **COLD:** no-show OR watch_percentage < 30

**Expected JSON output schema:**
```json
{
  "segment": "HOT|WARM|COLD",
  "reason": "string",
  "confidence": 0.0,
  "email_tone": "urgent|nurture|re-engage",
  "key_signal": "string",
  "email_body": "3-sentence personalised email body"
}
```

### OpenRouter Chat Model
Provides the LLM for the segmentation **AI Agent**. Model: `nvidia/nemotron-nano-9b-v2:free`. Temperature: `0.3`. Max tokens: `500`. Credential: `openRouterApi`.

### Code in JavaScript
Parses the AI Agent's output from `item.json.output` (the correct n8n AI Agent output key). Strips markdown code fences (`\`\`\`json`) using regex before `JSON.parse`. On exception, falls back to `segment: 'WARM'`, `confidence: 0.0`, and a generic email body. Merges parsed fields onto the existing item using spread (`...item.json`). Output fields added: `segment`, `segment_reason`, `confidence`, `email_tone`, `key_signal`, `ai_email_body`.

### Switch
Routes items to three output branches based on `$json.segment`:
- **Output 0** — `segment === "HOT"` → Gmail: HOT Lead Follow-up
- **Output 1** — `segment === "WARM"` → Gmail: WARM Lead Nurture
- **Output 2** — `segment === "COLD"` → Gmail: COLD Lead Re-engagement

Mode: `rules`. Fallback output: not configured (unmatched items are dropped).

### Gmail: HOT Lead Follow-up
Sends a high-urgency sales pitch email. References `$('If').item.json.*` for the lead's name, company, and watch percentage. Uses `$json.ai_email_body` for the AI-generated paragraph. Includes a static Calendly booking link (`https://calendly.com/your-team/demo`) that **must be updated**. Subject personalises to first name and company if available.

### Gmail: WARM Lead Nurture
Sends an HTML nurture email with a fixed list of three implementation tips and a resource hub link (`https://your-resource-hub.com` — **must be updated**). Uses `$json.ai_email_body` for the personalised paragraph. References lead name via `$('If').item.json.name.split(" ")[0]`.

### Gmail: COLD Lead Re-engagement
Sends an HTML email with a static replay link (`https://your-replay-link.com/webinar` — **must be updated**), a 7-day urgency framing, and a bulleted list of what the webinar covered. Low-pressure tone.

### If1
Secondary gate after the HOT email is sent. Checks `$('Code in JavaScript').item.json.confidence > 0.7` (number, `gt` operator). Only high-confidence HOT classifications proceed to Slack. The **false branch is not connected** — low-confidence HOT leads receive the email but no Slack alert, and replay distribution is also skipped for them (see Known Limitations).

### Slack: Hot Lead Alert
Posts a multi-line formatted text message. Currently configured to send to `USLACKBOT` (Slackbot DM) rather than a real channel — **must be reconfigured** to point to `#hot-leads`. Message includes: lead name, email, company, watch %, engagement score, AI key signal, and AI reason. Uses `slackApi` credential.

### Wait1
Pauses for `amount: 2` (no explicit `unit` field in parameters — n8n defaults this to hours). Effectively a 2-hour delay for replay recording to become available.

### Generate AI Webinar Summary
Second LangChain Agent node. System prompt: _"You are a B2B SaaS content writer… Return ONLY valid JSON. No markdown code fences."_ User prompt reads event name and date from `$('If').item.json.*` cross-node reference. Asks the model to return a JSON object with `email_subject`, `preview_text`, `headline`, `summary_paragraph`, `top_3_takeaways`, `cta_text`, `ps_line`. Connected sub-node: **OpenRouter Chat Model1**.

### OpenRouter Chat Model1
Provides the LLM for **Generate AI Webinar Summary**. Model: `openrouter/free` (generic free tier alias — resolves to whichever model OpenRouter currently maps to this alias, which may change). No temperature or max-token options set — uses OpenRouter defaults.

> ⚠️ Using `openrouter/free` is non-deterministic about which model is used. Consider pinning to a specific free model (e.g. `mistralai/mistral-7b-instruct:free`) for consistency.

### Code: Parse Summary Response
Attempts to read `item.json.choices[0].message.content` and JSON-parse it.
> ⚠️ **Bug:** The n8n AI Agent node returns its output as `item.json.output` (a plain string), not as a `choices` array. This node will always throw and fall back to the static copy unless corrected. See [Known Limitations](#known-limitations).

On fallback, static defaults are applied: subject `Watch the replay: <event_name>`, generic headline, three hardcoded takeaways, CTA `Watch Now`. Output fields added: `summary_email_subject`, `summary_preview`, `summary_headline`, `summary_paragraph`, `summary_takeaways` (pipe-delimited string), `summary_cta`, `summary_ps`.

### Google Sheets: Read All Registrants
Reads all rows from the `Registrants` sheet (same spreadsheet ID) filtered by:
1. `event_name` equals `$('If').item.json.event_name`
2. `replay_sent` — _(filter value is blank in the JSON, meaning this filter may not apply correctly — see Known Limitations)_

Returns all matching rows as individual items for the replay email loop.

### Gmail: Send Replay Email
Sends the AI-generated replay email to each registrant row's email. Subject and body are composed from `$('Code: Parse Summary Response').item.json.*` cross-node references. Replay link falls back to `https://your-replay-link.com/webinar` if `$json.replay_link` is not set on the row.

### Google Sheets: Update CRM Row
Updates existing rows in the `Registrants` sheet matched on `email`. Mapped write-back fields include: `attendance_status`, `duration_minutes`, `watch_percentage`, `engagement_score`, `questions_asked`, `poll_responses`, `segment`, `segment_reason`, `confidence`, `attended_at`, `follow_up_sent`, `replay_sent`. Uses `googleSheetsOAuth2Api`.

### Slack: Daily Digest Summary
Posts a completion message to Slackbot (same `USLACKBOT` target — **must be reconfigured** to `#events-ops`). References `$('Google Sheets: Read All Registrants').item.json.event_name`. Includes a hardcoded Google Sheets URL in the message body that must be updated.

---

## Branching Logic

### If — Attendance Record Validation
| Branch | Condition | Destination |
|---|---|---|
| True (output 0) | `email` not empty **AND** `attendance_status` matches `^(attended\|no-show\|partial)$` | AI Agent |
| False (output 1) | Either condition fails | _Not connected — records are silently dropped_ |

### Switch — Segment Routing
| Output | Condition | Destination |
|---|---|---|
| 0 | `segment === "HOT"` | Gmail: HOT Lead Follow-up |
| 1 | `segment === "WARM"` | Gmail: WARM Lead Nurture |
| 2 | `segment === "COLD"` | Gmail: COLD Lead Re-engagement |
| Fallback | No match | _Not configured — item dropped_ |

### If1 — Slack Alert Gate (HOT Confidence Threshold)
| Branch | Condition | Destination |
|---|---|---|
| True (output 0) | `confidence > 0.7` | Slack: Hot Lead Alert → Wait1 → replay chain |
| False (output 1) | `confidence ≤ 0.7` | _Not connected — no replay or digest for this lead_ |

> ⚠️ **Critical gap:** WARM and COLD leads, and low-confidence HOT leads, never reach `Wait1`. This means the replay email and CRM update are **only triggered when a HOT lead with confidence > 0.7 exists**. If an event has no HOT leads, or all HOT leads are low-confidence, the replay email and CRM update will never run.

---

## Variables & Expressions

Notable expressions used across the workflow:

| Expression | Node | What it computes |
|---|---|---|
| `'REG-' + $now.toFormat('yyyyMMdd') + '-' + Math.random().toString(36).substring(2,8).toUpperCase()` | Set Registration Fields | Unique registrant ID |
| `$json.body.email.toLowerCase().trim()` | Set Registration Fields | Normalised email |
| `new Date(\`${eventDate}T${eventTime}:00\`).getTime()` | Code: Calculate Event Datetime | Event Unix timestamp |
| `$('Google Sheets: Append Registrant').item.json.event_name` | Gmail: 24-Hour Reminder | Cross-node field reference |
| `$('Code: Calculate Event Datetime').item.json.reminder_1h_at` | Wait: 1h Reminder Delay | Resume timestamp |
| `Math.round(((attendee.duration_minutes\|\|0) / eventDuration) * 100)` | Code: Split Attendees Array | Watch percentage |
| `$('If').item.json.name.split(' ')[0]` | Gmail: HOT/WARM (subject/body) | First name extraction |
| `$('Code in JavaScript').item.json.confidence` | If1 | Confidence gate value |
| `$json.replay_link \|\| 'https://your-replay-link.com/webinar'` | Gmail: Send Replay Email | Replay URL with fallback |
| `$('Code: Parse Summary Response').item.json.summary_takeaways.split("\|\|\|")` | Gmail: Send Replay Email | Split pipe-delimited takeaways |

---

## Credentials Used

| Node(s) | Credential Name | Credential Type | Service |
|---|---|---|---|
| Google Sheets: Append Registrant, Lookup Registrant, Read All Registrants, Update CRM Row | `googleSheetsOAuth2Api` | Google Sheets OAuth2 | Google Sheets |
| Gmail: Confirmation Email, 24-Hour Reminder, 1-Hour Reminder, HOT Follow-up, WARM Nurture, COLD Re-engagement, Send Replay Email | `gmailOAuth2` | Gmail OAuth2 | Gmail |
| OpenRouter Chat Model | `openRouterApi` | OpenRouter API | OpenRouter (Nemotron Nano 9B free) |
| OpenRouter Chat Model1 | `openRouterApi` | OpenRouter API | OpenRouter (free alias) |
| Slack: Hot Lead Alert, Daily Digest Summary | `slackApi` | Slack API | Slack |

> Credential names above are references only. Actual secret values are never included in n8n JSON exports.

---

## Input Data

**Path 1 — Registration webhook:**
```json
{
  "name": "Jane Smith",
  "email": "jane@company.com",
  "company": "Acme Corp",
  "phone": "+1-555-0192",
  "event": "AI Automation Webinar",
  "event_date": "2025-02-15",
  "event_time": "14:00",
  "timezone": "America/New_York",
  "source": "linkedin_ad"
}
```

**Path 2 — Attendance webhook:**
```json
{
  "event_name": "AI Automation Webinar",
  "event_date": "2025-02-15",
  "event_duration_minutes": 60,
  "attendees": [
    { "email": "jane@company.com", "status": "attended", "duration_minutes": 57,
      "questions_asked": 2, "poll_responses": 3, "chat_messages": 4 }
  ]
}
```

---

## Output Data

| Terminal Node | Output / Side-effect |
|---|---|
| Gmail: 1-Hour Reminder | Email sent to registrant (currently test address) |
| Gmail: HOT Lead Follow-up | Sales-pitch HTML email sent |
| Gmail: WARM Lead Nurture | Nurture HTML email sent |
| Gmail: COLD Lead Re-engagement | Replay-link HTML email sent |
| Slack: Hot Lead Alert | Lead card message sent to Slackbot DM (needs reconfiguration) |
| Gmail: Send Replay Email | Replay email sent to all `event_name`-matching registrants |
| Google Sheets: Update CRM Row | Registrant row updated with segment, scores, and flags |
| Slack: Daily Digest Summary | Completion message sent to Slackbot DM (needs reconfiguration) |

---

## Data Transformation

1. **Set Registration Fields** — flattens the nested `body.*` webhook payload into flat fields; normalises casing and trims whitespace; synthesises `registrant_id`.
2. **Code: Split Attendees Array** — transforms a single array-containing item into N individual items; derives `watch_percentage` and `engagement_score` as computed fields.
3. **Merge** — enriches attendance items with registration metadata (name, company, source, registrant_id) joined on `email`.
4. **Code in JavaScript** — transforms the AI Agent's raw text output into structured fields; sanitises markdown formatting; applies a typed fallback.
5. **Code: Parse Summary Response** — transforms the summary agent's text into seven named fields; converts `top_3_takeaways` array to a `|||`-delimited string for use in Gmail expressions.

---

## API Integrations / External Services

| Service | Node(s) | Endpoint / Method | Auth | Purpose |
|---|---|---|---|---|
| Google Sheets API | Append, Lookup, Read All, Update CRM | Sheets API v4 via OAuth2 | OAuth2 (`googleSheetsOAuth2Api`) | CRM read/write |
| Gmail API | All 7 Gmail nodes | Gmail API v1 `messages.send` | OAuth2 (`gmailOAuth2`) | All outbound emails |
| OpenRouter API | OpenRouter Chat Model, OpenRouter Chat Model1 | `https://openrouter.ai/api/v1/chat/completions` | API key (`openRouterApi`) | LLM inference |
| Slack API | Hot Lead Alert, Daily Digest Summary | Slack Web API `chat.postMessage` | Bot token (`slackApi`) | Notifications |

---

## AI / LLM Integration

### AI Agent — Lead Segmentation

| Parameter | Value |
|---|---|
| Node | AI Agent |
| Model node | OpenRouter Chat Model |
| Model | `nvidia/nemotron-nano-9b-v2:free` |
| Temperature | 0.3 |
| Max tokens | 500 |
| Prompt type | `define` (fully constructed in the node) |
| System role | _(not set — relies on the user-role prompt alone)_ |
| Output consumed by | Code in JavaScript (reads `item.json.output`) |

**Segmentation rules embedded in prompt:**
- HOT: attended AND (watch_percentage ≥ 80 OR engagement_score ≥ 90)
- WARM: attended AND watch_percentage 30–79, or low engagement
- COLD: no-show OR watch_percentage < 30

**Output schema:**
```json
{
  "segment": "HOT|WARM|COLD",
  "reason": "one sentence",
  "confidence": 0.95,
  "email_tone": "urgent|nurture|re-engage",
  "key_signal": "string",
  "email_body": "3-sentence personalised email body"
}
```

### Generate AI Webinar Summary — Replay Email Copy

| Parameter | Value |
|---|---|
| Node | Generate AI Webinar Summary |
| Model node | OpenRouter Chat Model1 |
| Model | `openrouter/free` (non-deterministic alias) |
| Temperature | Not set (OpenRouter default) |
| Max tokens | Not set (OpenRouter default) |
| System prompt | "You are a B2B SaaS content writer… Return ONLY valid JSON. No markdown code fences." |
| Output consumed by | Code: Parse Summary Response |

> ⚠️ Due to the `choices[0].message.content` bug in Code: Parse Summary Response, the AI-generated summary is currently never used; static fallback copy always applies.

---

## Error Handling

| Node | continueOnFail | onError | Notes |
|---|---|---|---|
| All nodes | Not configured (default `false`) | Default (stop execution) | No explicit error handling on any node |

- **No error workflow** is configured in `settings`.
- **If (validate)** provides the only explicit bad-data guard — invalid records are silently dropped (no logging).
- **Code in JavaScript** and **Code: Parse Summary Response** both have `try/catch` blocks with fallback values — this is the only resilience in the workflow.
- **Wait1** false branch from If1 means the entire replay + CRM update chain silently does not run for non-HOT or low-confidence leads.

---

## Retry Strategy

No retry logic is configured on any node. No `retryOnFail`, `maxTries`, or `waitBetweenTries` settings are present. If an OpenRouter API call, Gmail send, or Sheets write fails, the execution stops with an error and does not retry.

---

## Execution Order

`settings.executionOrder: "v1"` — connection-based execution order. Branches from Split/fan-out nodes execute sequentially in the order they appear in the connections map, not in parallel.

---

## Security Considerations

| Item | Current State | Recommendation |
|---|---|---|
| Webhook authentication (both triggers) | None — any request to the URL can trigger the workflow | Add webhook header auth or basic auth in the Webhook node `Authentication` setting |
| Gmail `sendTo` field | Hardcoded to `storysnippents@gmail.com` | Must use `={{ $json.email }}` to send to actual registrant |
| Replay and resource hub links | Static placeholder URLs in email bodies | Update before production; consider storing in a Sheets config row rather than hardcoding |
| Google Spreadsheet ID | Hardcoded in all Sheets nodes (`1avIj_F3leYjeplBNhDJuNV4j9KevaEtrgbYtJrPnamQ`) | This ties the workflow to one specific spreadsheet — document this as a deploy-time dependency |
| Slack target | Both Slack nodes send to `USLACKBOT` (Slackbot DM) | Reconfigure to real channels (`#hot-leads`, `#events-ops`) |
| OpenRouter API key | Stored as n8n credential `openRouterApi` | Rotate periodically; ensure n8n instance credentials store is encrypted |
| Credential sign-off | _[To be reviewed and signed off by workflow owner / security reviewer]_ | — |

---

## Performance Considerations

- **Wait nodes** with `specificTime` resume: executions are persisted to disk and consume no resources during the wait. Thousands of concurrent registrant executions are feasible assuming sufficient disk.
- **Loops:** There is no explicit loop node. `Google Sheets: Read All Registrants` returns multiple rows and `Gmail: Send Replay Email` processes each as a separate item — this is n8n's built-in item-level iteration. For very large registrant lists (1000+), this may take several minutes and is subject to Gmail API rate limits.
- **OpenRouter rate limits:** The free tier typically allows ~200 requests/day per model. For events with more than ~200 attendees, the AI segmentation calls may be throttled. Add a `Split In Batches` node before the AI Agent with batch size 1 and a small wait to manage rate limits.
- **Execution time:** _[To be provided by workflow owner — requires production execution data]_
- **Resource usage:** _[To be provided by workflow owner]_

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---|---|---|
| Confirmation email arrives at `storysnippents@gmail.com` instead of registrant | `sendTo` is hardcoded | Change all Gmail node `sendTo` fields to `={{ $json.email }}` |
| Reminders fire immediately at registration time | `event_date` + `event_time` is in the past | Use a future event date; the `Math.max(0,...)` guard sets wait to 0 ms for past events |
| Attendance records are dropped silently after Merge | Attendee email not found in Registrants sheet | Check that the email in the attendance payload exactly matches (same case, no spaces) the email stored during registration |
| AI Agent returns text but segment is always `WARM` | AI output is not valid JSON | Check the AI Agent output in n8n execution viewer; the model may be prefacing its response with prose. Strengthen the prompt with "Return ONLY raw JSON, starting with `{`" |
| No Slack message appears | Slack node targets `USLACKBOT`, not a real channel | Reconfigure the Slack nodes' `Select` field to a channel |
| Replay email contains static fallback copy | `Code: Parse Summary Response` reads `choices[0].message.content` but AI Agent outputs `item.json.output` | Change `item.json.choices[0].message.content` to `item.json.output` in that Code node |
| Replay email never sends | No HOT lead with `confidence > 0.7` in the batch | The `Wait1` → replay chain is only reachable via `If1`'s true branch; wire WARM/COLD paths into `Wait1` too |
| Google Sheets update writes to wrong row | `email` match is case-sensitive in Sheets `update` | Ensure the Set Registration Fields node's email normalisation (`.toLowerCase()`) is applied consistently to incoming attendance data too |
| Workflow activates but webhooks return 404 | Workflow is in test mode | Toggle workflow to **Active**; production webhook URL replaces `/webhook-test/` with `/webhook/` |

---

## Known Limitations

1. **All Gmail `sendTo` fields are hardcoded** to `storysnippents@gmail.com`. All 7 Gmail nodes must have this corrected to `={{ $json.email }}` before production.
2. **Both Slack nodes target `USLACKBOT`** (Slackbot DM) instead of real channels. Must be reconfigured to `#hot-leads` and `#events-ops`.
3. **Replay distribution is only triggered for HOT leads with confidence > 0.7.** WARM and COLD leads never reach `Wait1`. If an event has no qualifying HOT leads, replay emails are never sent and CRM rows are never updated.
4. **`Code: Parse Summary Response` reads the wrong output key.** `item.json.choices[0].message.content` does not exist on AI Agent output; the correct key is `item.json.output`. The AI-generated summary is therefore never applied — the static fallback is always used.
5. **`openrouter/free` model alias** in `OpenRouter Chat Model1` is non-deterministic and may change without notice. Should be pinned to a specific free model.
6. **Invalid attendance records (If false branch) are silently dropped** with no logging or alerting.
7. **`Google Sheets: Read All Registrants` filter for `replay_sent`** has a blank `lookupValue` in the JSON, which may not correctly filter to only un-replayed registrants.
8. **No retry logic on any node.** A transient Gmail, Sheets, or OpenRouter failure will abort the execution.
9. **No webhook authentication on either trigger.** Unauthenticated POSTs from any source will execute the workflow.
10. **Static placeholder URLs** in email bodies (`your-webinar-link.com/join`, `calendly.com/your-team/demo`, `your-resource-hub.com`, `your-replay-link.com/webinar`) must all be updated.
11. **`Slack: Daily Digest Summary` hardcodes the Google Spreadsheet URL** in the message text rather than deriving it from the node's configured spreadsheet ID.
12. **`Generate AI Webinar Summary` references `$('If').item.json.name`** for the webinar name in the prompt — this will use the attendee's personal `name` field instead of `event_name` unless corrected.

---

## Assumptions

- The `attendance_status` values sent in the attendance webhook match exactly one of `attended`, `no-show`, or `partial`. Any other value (e.g. `no_show` with underscore) will fail the `If` validation and be dropped.
- The Google Sheets `Registrants` tab has column headers in Row 1 matching the field names used in all four Sheets nodes.
- The Google account connected via `gmailOAuth2` has permission to send on behalf of the desired sender address.
- `Wait1`'s `amount: 2` without an explicit `unit` field resolves to 2 hours in n8n v1.30+ (confirmed default behaviour).
- The workflow is intended to be triggered manually/via external POST for both entry points — there is no internal scheduling.
- The engagement score formula weights are business decisions (questions × 10, polls × 5, chat × 2) and can be adjusted in `Code: Split Attendees Array`.

---

## Testing & Validation

_[To be provided by workflow owner — include test execution IDs, sample input/output pairs, and confirmation that emails were received at correct addresses.]_

---

## Version History / Change Log

| Version | Date | Author | Changes |
|---|---|---|---|
| v1.0 | _[TBD]_ | _[TBD]_ | Initial workflow creation |

_[To be maintained by workflow owner going forward.]_

---

## References

- n8n LangChain Agent docs: https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent/
- OpenRouter free models: https://openrouter.ai/models?q=free
- Gmail API quota: https://developers.google.com/gmail/api/reference/quota
- Google Sheets API: https://developers.google.com/sheets/api
- n8n Wait node: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.wait/
- n8n Webhook node: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.webhook/

_[Additional project-specific references to be added by workflow owner.]_

---

## Maintenance Notes

- **Before each webinar:** Verify the event date/time passed to the registration webhook is correct and in the future.
- **After each webinar:** POST the attendance export to `/webinar-attendance` within a reasonable window (the 2-hour replay wait begins from the moment the HOT lead's Slack alert fires).
- **Quarterly:** Rotate the OpenRouter API key; re-authenticate Gmail OAuth2 and Google Sheets OAuth2 credentials before expiry.
- **Model updates:** `nvidia/nemotron-nano-9b-v2:free` may be deprecated by OpenRouter. Monitor the free model catalogue and update the credential/model name accordingly.
- **Spreadsheet maintenance:** Archive or clear old event rows in the `Registrants` sheet periodically to prevent the `Read All Registrants` node from returning stale data across events.

_[Additional operational notes to be added by workflow owner.]_
