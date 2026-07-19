# AI Press Release & Journalist Outreach Automation — Workflow Documentation

| Field | Value |
|---|---|
| **Workflow Name** | AI press release and journalist outreach automation |
| **Workflow ID** | `xZ4CcXpJqKESR3Gr` |
| **Version ID (internal)** | `5f3ed0bc-f37b-4f1f-a762-f133f37bb34a` *(n8n internal revision hash — not a semantic version)* |
| **Semantic Version** | v1.0 (initial documentation) |
| **Owner** | *[To be provided by workflow owner]* |
| **Workflow Last Saved** | Not available in export (`updatedAt` absent) |
| **Document Generated** | 2026-07-16 |
| **Workflow Active** | No *(not activated at time of export)* |
| **Execution Order** | v1 |

---

## Purpose

This workflow automates the full lifecycle of a press release campaign triggered by a new company announcement. When a row is added to a designated Google Sheet, the system generates a professional press release via an LLM, discovers journalists from both a live Google News RSS feed and a seed database, scores each journalist for relevance using a second LLM, generates a personalised pitch email per qualified journalist using a third LLM, delivers those emails via Gmail, and records every outreach attempt — sent or skipped — to a tracking sheet.

---

## Business Use Case

*(Inferred — please confirm or replace.)* Eliminates the manual effort of writing press releases and drafting individual journalist pitches after each company announcement. Reduces time-to-outreach from hours to minutes and standardises the quality of PR communications without requiring a dedicated PR team.

---

## Workflow Summary

A new row in the `News_Trigger` Google Sheet tab fires the workflow. Announcement fields are sanitised and enriched with a unique campaign ID. An AI agent (Nemotron-Super-120B via OpenRouter) writes an AP-Style press release and returns it as structured JSON. In parallel, the workflow fetches recent journalist articles from Google News RSS and reads all rows from a seed journalist database. Both lists are merged and fed into a batch loop (5 journalists per iteration), where a lighter AI model (Nemotron-Nano-12B) scores each journalist's relevance to the announcement from 0 to 100. Journalists scoring below 60 are immediately logged to the tracking sheet as skipped. Those scoring 60 or above proceed to a third AI model (Trinity-Large-Preview) that writes a personalised pitch email. A final email-validity gate rejects entries without a well-formed email address. Valid emails are sent via Gmail, with Reply-To set to the company's press contact. Every journalist record — sent or skipped — is appended to the `Outreach_Tracker` sheet. When all batches are exhausted, an Execution Summary node computes totals and writes a campaign-level result.

---

## Workflow Diagram

See the attached canvas screenshot (`AI_press_release_and_journalist_outreach_automation.webp`).

**Shape:** The flow is linear from trigger through press-release generation and journalist discovery, fans-in at the Merge node, then enters a loop. Inside the loop, two serial IF-gate branches create four distinct paths: (a) score < 60 → log skipped; (b) score ≥ 60, invalid email → log skipped; (c) score ≥ 60, valid email → send email and loop back. The loop's "done" output bypasses the inner logic and feeds directly to the Execution Summary node.

---

## Trigger Details

| Field | Value |
|---|---|
| **Trigger Node** | `Google Sheets Trigger` |
| **Trigger Type** | `n8n-nodes-base.googleSheetsTrigger` v1 |
| **Event** | `rowAdded` — fires each time a new row is added to the monitored sheet |
| **Spreadsheet** | AI press release and journalist outreach automation *(ID: `1pTOQnzKsUaADiSifm5r4Pc64A0K5Ewn80Ek-wSLz5I8`)* |
| **Sheet Tab** | `News_Trigger` (gid=0) |
| **Poll Interval** | Every minute *(default polling; not webhook-mode)* |
| **Credential** | Google Sheets Trigger account (`googleSheetsTriggerOAuth2Api`) |

The trigger is poll-based, not push-based, so there is up to a one-minute delay between a row being added and execution beginning. The trigger passes all columns from the new row as key-value pairs in `$json`.

**Expected input columns in `News_Trigger`:**

| Column | Type | Notes |
|---|---|---|
| `company_name` | String | Required |
| `announcement_type` | String | e.g. `Product Launch`, `Funding Round` |
| `details` | String | Full announcement text (100–300 words ideal) |
| `target_industry` | String | Comma-separated, e.g. `HR Tech, SaaS` |
| `tone` | String | Optional; defaults to `professional` |
| `spokesperson` | String | Optional |
| `website` | String | Optional |
| `contact_email` | String | Used as Reply-To on outgoing emails |
| `embargo_date` | String/Date | Optional; blank = immediate release |

---

## Execution Flow

1. **Google Sheets Trigger** detects a new row in `News_Trigger` and passes all column values downstream.

2. **Normalize Input** sanitises every field (`.trim()`, lowercases `tone`), applies safe defaults for optional fields, and generates a `campaign_id` in the format `PR_<timestamp>_<6-char-random>`. Adds `created_at` ISO timestamp. Output is a cleaned JSON object.

3. **Press Release Generator (AI Agent)** — backed by `OpenRouter Chat Model` (Nemotron-Super-120B) — receives the normalised fields and produces a structured press release JSON with fields: `headline`, `subheadline`, `dateline`, `lead_paragraph`, `body_paragraph_1`, `body_paragraph_2`, `executive_quote`, `boilerplate`, `media_contact_block`, `full_press_release`.

4. **Parse Press Release (Code)** strips any accidental markdown fences from the AI's output, then `JSON.parse`s it. On parse failure, returns `{ error: "Invalid JSON from AI", raw: <raw string> }` and the workflow continues (no crash).

5. **Google News RSS Fetch (HTTP Request)** makes a GET request to Google News RSS using the `target_industry` field from step 2 as the search keyword. Returns raw XML. No API key required.

6. Two branches run in parallel from step 5:
   - **Parse RSS to Journalist Leads (Code)** — regex-parses the XML, extracts up to 20 `<item>` blocks, and returns an array of lead objects containing `article_title`, `publication`, `article_url`, `source_type: 'google_news_rss'`, and empty `name`/`email`/`beat` fields.
   - **Fetch Journalist Seed DB** — reads all rows from the `Journalist_DB` Google Sheet tab, returning known journalists with full contact details.

7. **Merge** combines both streams using `combineAll` (Cartesian cross-join), producing one unified list of all journalist candidates.

8. **Loop Over Items** (`splitInBatches`, batch size = 5, reset = false) begins the batch loop:
   - **Output 0 ("done")** — fires to **Execution Summary** once all items are exhausted.
   - **Output 1 ("loop")** — fires the current batch of up to 5 journalists into the inner pipeline.

9. **Journalist Relevance Scorer (AI Agent)** — backed by `OpenRouter Chat Model1` (Nemotron-Nano-12B) — scores each journalist 0–100 for relevance to the announcement, and returns `{ relevance_score, match_reason, suggested_angle }`.

10. **Parse Relevance Score (Code)** extracts `relevance_score`, `match_reason`, and `suggested_angle` from the AI output, merges them onto the journalist's existing data object, and defaults score to 0 on parse failure.

11. **Filter Score >= 60 (IF)** evaluates `$json.relevance_score >= 60`:
    - **True (score ≥ 60)** → proceeds to **Email Personalization AI**
    - **False (score < 60)** → goes to **Log to Outreach Tracker** (logged as `skipped`, reason `low_score`)

12. **Email Personalization AI (AI Agent)** — backed by `OpenRouter Chat Model2` (Trinity-Large-Preview) — writes a personalised pitch email for the journalist, returning `{ subject_line, email_body, personalization_note }`.

13. **Parse Pitch Email (Code)** cleans markdown fences, JSON-parses the AI output, validates the recipient email address with a regex, resolves name fallback to `'there'` if missing, and sets `email_valid: true/false`. On AI parse failure, substitutes a safe fallback subject and body.

14. **Check Email Valid (IF)** evaluates `$json.email_valid === true`:
    - **True** → proceeds to **Send Pitch Email**
    - **False** → goes to **Log to Outreach Tracker2** (logged as `skipped`, reason `missing_email` or `invalid_email`)

15. **Send Pitch Email (Gmail)** sends the email to `storysnippents@gmail.com` *(see Known Limitations — this is hardcoded and not the dynamic recipient)*. Subject is `$json.subject_line`, body is `$json.email_body` with `\n` converted to `<br>`. Reply-To is set to `$('Normalize Input').item.json.contact_email`.

16. After sending, **Loop Over Items** receives the "done" signal from `Send Pitch Email` and loops back to fetch the next batch (step 8). When all batches are processed, step 8's Output 0 fires.

17. **Execution Summary (Code)** aggregates across all items: counts sent vs skipped, computes average relevance score, and returns a campaign-level result object with `status: 'success'` or `'no_emails_sent'`.

---

## Node List

| # | Node Name | Type | Role | Disabled |
|---|---|---|---|---|
| 1 | Google Sheets Trigger | Google Sheets Trigger | Entry point — watches `News_Trigger` for new rows | No |
| 2 | Normalize Input | Code (JavaScript) | Sanitises fields, adds `campaign_id` and `created_at` | No |
| 3 | OpenRouter Chat Model | LM Chat OpenRouter | LLM sub-node for Press Release Generator — Nemotron-Super-120B | No |
| 4 | Press Release Generator (AI Agent) | AI Agent | Generates AP-Style press release JSON | No |
| 5 | Parse Press Release (Code) | Code (JavaScript) | Strips markdown fences, JSON.parses AI output | No |
| 6 | Google News RSS Fetch (HTTP Request) | HTTP Request | Fetches journalist leads from Google News RSS (free, no key) | No |
| 7 | Parse RSS to Journalist Leads (Code) | Code (JavaScript) | Regex-parses RSS XML into journalist lead objects | No |
| 8 | Fetch Journalist Seed DB | Google Sheets | Reads all rows from `Journalist_DB` sheet | No |
| 9 | Merge | Merge | Combines RSS leads + DB journalists via `combineAll` | No |
| 10 | Loop Over Items | Split In Batches | Iterates all journalists in batches of 5 | No |
| 11 | OpenRouter Chat Model1 | LM Chat OpenRouter | LLM sub-node for Journalist Relevance Scorer — Nemotron-Nano-12B | No |
| 12 | Journalist Relevance Scorer | AI Agent | Scores each journalist 0–100 for relevance | No |
| 13 | Parse Relevance Score | Code (JavaScript) | Extracts score data from AI output, merges onto journalist object | No |
| 14 | Email Personalization AI | AI Agent | Generates personalised pitch email per qualified journalist | No |
| 15 | OpenRouter Chat Model2 | LM Chat OpenRouter | LLM sub-node for Email Personalization AI — Trinity-Large-Preview | No |
| 16 | Parse Pitch Email | Code (JavaScript) | Validates email, extracts subject/body, sets `email_valid` flag | No |
| 17 | Execution Summary | Code (JavaScript) | Aggregates campaign totals after loop completes | No |
| 18 | Log to Outreach Tracker | Google Sheets | Appends low-score skipped records to `Outreach_Tracker` | No |
| 19 | Send Pitch Email | Gmail | Sends personalised pitch email via Gmail OAuth2 | No |
| 20 | Check Email Valid | IF | Gates on `email_valid` boolean | No |
| 21 | Filter Score >= 60 | IF | Gates on `relevance_score >= 60` | No |
| 22 | Log to Outreach Tracker2 | Google Sheets | Appends invalid-email skipped records to `Outreach_Tracker` | No |

---

## Node-by-Node Description

### Google Sheets Trigger
Polls the `News_Trigger` tab of the linked spreadsheet every minute for newly added rows. On detection, emits all column values as a flat JSON object and triggers the rest of the workflow. No filtering or pre-processing occurs at this node.

### Normalize Input
Runs once per item. Reads every field from the trigger output, applies `.trim()` to string fields, lowercases `tone`, and provides safe defaults (`'General Announcement'` for `announcement_type`, `'Technology'` for `target_industry`, `'professional'` for `tone`). Generates a `campaign_id` with format `PR_<unix-ms>_<6-char-uppercase-random>` for tracking. Appends `created_at` in ISO 8601. All downstream nodes reference these normalised values.

### OpenRouter Chat Model (sub-node for Press Release Generator)
LLM configuration node. Model: `nvidia/nemotron-3-super-120b-a12b:free`. Attached as sub-node to the AI Agent. Handles token streaming and API communication with OpenRouter.

### Press Release Generator (AI Agent)
AI Agent node. Receives the normalised announcement fields and invokes the Nemotron-Super-120B model to produce a complete press release. The system prompt enforces AP Style, forbids markdown output, and requires strict JSON-only response. The user prompt constructs the context block from `$json` fields with `||` fallbacks. Expected output is a JSON object with 10 named fields covering headline through full formatted text.

### Parse Press Release (Code)
Reads `$json.output` (or `$json.text` as fallback). Strips any `` ```json `` / ` ``` ` markdown fences the model may include despite instructions. Attempts `JSON.parse`. On success, returns the parsed object as-is. On failure, returns `{ error: "Invalid JSON from AI", raw: <string> }` — the workflow does not halt; downstream nodes that expect specific fields (e.g. `headline`) will receive empty/undefined values.

### Google News RSS Fetch (HTTP Request)
GET request to `https://news.google.com/rss/search` with `q` set to `<target_industry> journalist reporter` (URL-encoded). Sets `User-Agent: Mozilla/5.0`. 10-second timeout. Returns raw XML string (response format: string). No API key or credential required.

### Parse RSS to Journalist Leads (Code)
Runs once for all items. Extracts the raw XML from `items[0].json.data` or `.body`. Uses regex to find all `<item>` blocks (up to 20), then extracts `title`, `link`, and `source` per block, stripping CDATA wrappers. Returns an array of up to 20 lead objects. If no items are found, returns `[{ json: { no_results: true } }]` — the workflow continues with only DB journalists.

### Fetch Journalist Seed DB
Reads all rows from the `Journalist_DB` tab (gid=1930018968) of the same spreadsheet. Each row should contain journalist fields: `name`, `email`, `publication`, `beat`, `relevance_tags`, `opt_out`, etc. Returns one item per journalist row. Runs in parallel with the RSS fetch after `Parse Press Release (Code)`.

### Merge
Combines the two incoming streams (RSS leads from input 0, DB journalists from input 1) using `mode: combine`, `combineBy: combineAll`. This produces a Cartesian product of both lists. **Important:** `combineAll` joins every item from input 0 with every item from input 1, not a simple append. Depending on list sizes, this can produce a large number of combinations. See Known Limitations.

### Loop Over Items
`splitInBatches` node with `batchSize: 5` and `reset: false`. Processes the merged journalist list 5 items at a time. Output 0 ("done") fires to `Execution Summary` when all items are exhausted. Output 1 ("loop") fires each batch to `Journalist Relevance Scorer`. The loop is closed by `Send Pitch Email` feeding back to this node's input.

### OpenRouter Chat Model1 (sub-node for Journalist Relevance Scorer)
LLM configuration node. Model: `nvidia/nemotron-nano-12b-v2-vl:free`. A smaller, faster model appropriate for the short JSON scoring task.

### Journalist Relevance Scorer
AI Agent. For each journalist in the current batch, sends announcement context (company, type, industry, details — sourced from `Normalize Input` via `$('Normalize Input').item.json`) and journalist context (name, publication, beat, article title if RSS-sourced) to the Nemotron-Nano-12B model. Returns `{ relevance_score: <int 0-100>, match_reason: <string>, suggested_angle: <string> }` as JSON.

### Parse Relevance Score
Reads `item.output`, strips markdown fences, JSON-parses the scoring result. Uses `Number()` coercion to ensure `relevance_score` is numeric (defaults to 0 on failure). Merges `relevance_score`, `match_reason`, and `suggested_angle` onto the journalist's data object using spread (`...item`) so all original fields are preserved.

### Filter Score >= 60
IF node with a single numeric condition: `$json.relevance_score >= 60`. True branch proceeds to email generation. False branch routes to `Log to Outreach Tracker`. The threshold of 60 is hardcoded in the node condition.

### Email Personalization AI
AI Agent backed by Trinity-Large-Preview (OpenRouter). Constructs the prompt from: the press release headline (via `$node["Parse Press Release (Code)"].json.headline`), company/details/spokesperson/contact email (via `$node["Normalize Input"].json`), and journalist name/publication/beat (via `$node["Fetch Journalist Seed DB"].json`) plus `$json.suggested_angle` from the scorer. Returns `{ subject_line, email_body, personalization_note }`. Subject must be under 10 words; body under 150 words.

### OpenRouter Chat Model2 (sub-node for Email Personalization AI)
LLM configuration node. Model: `arcee-ai/trinity-large-preview:free`.

### Parse Pitch Email
Five-step code node: (1) reads `item.output` or `item.text`, (2) strips markdown fences, (3) JSON-parses with a fallback on failure that substitutes a generic subject and body, (4) validates the recipient email using regex `/^[^\s@]+@[^\s@]+\.[^\s@]+$/` — checking `item.email` then `item.journalist_email`, (5) resolves recipient name from `item.name` or `item.journalist_name` with final fallback `'there'`. Sets `email_valid: true` only if both regex passes and `subject.length > 0`. Preserves all input fields on the output object.

### Execution Summary
Runs once for all items when the loop's "done" output fires. Counts items where `email_status === 'sent'` and `=== 'skipped'`. Computes average `relevance_score` across all items. Returns a single object: `{ campaign_id, total_journalists_scored, emails_sent, emails_skipped, average_relevance_score, run_completed_at, status }`. `status` is `'success'` if any email was sent, otherwise `'no_emails_sent'`.

### Log to Outreach Tracker (nodes 18 and 22)
Both nodes are identical Google Sheets `append` operations writing to the `Outreach_Tracker` tab. Node 18 handles score-gated skips (false branch of `Filter Score >= 60`). Node 22 handles email-invalid skips (false branch of `Check Email Valid`). Both write 19 columns — see the Data Transformation section for the full column mapping.

### Send Pitch Email
Gmail `send` operation. **Currently hardcoded `sendTo: storysnippents@gmail.com`** — this is a test address and does not use `$json.recipient_email` dynamically. Subject is `$json.subject_line`. Message body is `$json.email_body` with `\n` replaced by `<br>` for HTML rendering. `replyTo` is set to `$('Normalize Input').item.json.contact_email`. After sending, connects back to `Loop Over Items` input to continue the batch cycle.

### Check Email Valid
IF node with a single boolean condition: `$json.email_valid === true` (using `singleValue: true` operator). True branch goes to `Send Pitch Email`. False branch goes to `Log to Outreach Tracker2`.

---

## Node Configuration

### Google News RSS URL
```
https://news.google.com/rss/search?q={{ encodeURIComponent($('Normalize Input').item.json.target_industry + ' journalist reporter') }}&hl=en-US&gl=US&ceid=US:en
```

### Normalize Input — full code
```javascript
const item = $input.first().json;
const campaignId = `PR_${Date.now()}_${Math.random().toString(36).substr(2,6).toUpperCase()}`;
return {
  json: {
    campaign_id: campaignId,
    company_name: (item.company_name || '').trim(),
    announcement_type: (item.announcement_type || 'General Announcement').trim(),
    details: (item.details || '').trim(),
    target_industry: (item.target_industry || 'Technology').trim(),
    tone: (item.tone || 'professional').toLowerCase().trim(),
    spokesperson: (item.spokesperson || '').trim(),
    website: (item.website || '').trim(),
    contact_email: (item.contact_email || '').trim(),
    embargo_date: item.embargo_date || null,
    created_at: new Date().toISOString()
  }
};
```

### Press Release Generator — system prompt
```
You are a veteran PR copywriter with 15+ years experience.
STRICT RULES:
- Return ONLY valid JSON
- Do NOT include markdown (no ```json)
- Do NOT include explanations
- Do NOT include any text before or after JSON
- Ensure JSON is COMPLETE and VALID
- Never return empty output
Write in AP Style. Avoid buzzwords.
Output must strictly follow the schema.
```

### Press Release Generator — user prompt
```
COMPANY: {{ $json.company_name || "Unknown Company" }}
TYPE: {{ $json.announcement_type || "Announcement" }}
DETAILS: {{ $json.details || "No details provided" }}
INDUSTRY: {{ $json.target_industry || "General" }}
TONE: {{ $json.tone || "Professional" }}
SPOKESPERSON: {{ $json.spokesperson || "Company Representative" }}
WEBSITE: {{ $json.website || "Not provided" }}
EMBARGO: {{ $json.embargo_date || "Immediate Release" }}
```

### Journalist Relevance Scorer — full prompt
```
System: You are a PR targeting specialist. Output ONLY valid JSON.

User:
Score relevance of this journalist to this announcement (0-100).

ANNOUNCEMENT:
Company: {{ $('Normalize Input').item.json.company_name }}
Type: {{ $('Normalize Input').item.json.announcement_type }}
Industry: {{ $('Normalize Input').item.json.target_industry }}
Summary: {{ $('Normalize Input').item.json.details }}

JOURNALIST:
Name: {{ $json.name || $json.journalist_name || 'Unknown' }}
Publication: {{ $json.publication }}
Beat: {{ $json.beat || $json.relevance_tags || 'General Tech' }}
Article (if RSS): {{ $json.article_title || 'N/A' }}

Return ONLY:
{
  "relevance_score": <integer 0-100>,
  "match_reason": "<one sentence>",
  "suggested_angle": "<specific angle for this journalist's beat>"
}
```

### Email Personalization AI — full prompt
```
System: You write pitch emails journalists actually open. Short, specific,
no generic PR language. ONLY valid JSON.

User:
Write a personalized pitch email.

PRESS RELEASE HEADLINE: {{ $node["Parse Press Release (Code)"].json.headline }}
COMPANY: {{ $node["Normalize Input"].json.company_name }}
KEY DETAILS: {{ $node["Normalize Input"].json.details }}
SPOKESPERSON: {{ $node["Normalize Input"].json.spokesperson }}
CONTACT EMAIL: {{ $node["Normalize Input"].json.contact_email }}
TONE: {{ $node["Normalize Input"].json.tone }}

JOURNALIST NAME: {{ $node["Fetch Journalist Seed DB"].json.name }}
PUBLICATION: {{ $node["Fetch Journalist Seed DB"].json.publication }}
BEAT: {{ $node["Fetch Journalist Seed DB"].json.beat }}
SUGGESTED ANGLE: {{ $json.suggested_angle }}

Rules: Subject under 10 words. Body under 150 words.
Reference their specific beat. Single CTA: reply to this email.
Never use "I hope this finds you well" or "I'm reaching out."

Return ONLY:
{
  "subject_line": "<under 10 words>",
  "email_body": "<under 150 words, \n for line breaks>",
  "personalization_note": "<what was personalized>"
}
```

### Gmail node — parameter summary
| Parameter | Value |
|---|---|
| `sendTo` | `storysnippents@gmail.com` *(hardcoded — see Known Limitations)* |
| `subject` | `={{ $json.subject_line }}` |
| `message` | `={{ $json.email_body.replace(/\n/g, '<br>') }}` |
| `replyTo` | `={{ $('Normalize Input').item.json.contact_email }}` |

### Outreach Tracker columns written (both Log nodes)
```
campaign_id, company_name, announcement_type, journalist_name,
journalist_email, publication, relevance_score, subject_line,
email_body_preview, email_status, skip_reason, sent_at,
open_status, reply_status, followup_sent, followup_count,
pr_headline, personalization_note, last_followup_at
```

---

## Branching Logic

### Filter Score >= 60 (node 21)
| Condition | Branch | Destination |
|---|---|---|
| `$json.relevance_score >= 60` is true | True (output 0) | Email Personalization AI |
| `$json.relevance_score >= 60` is false | False (output 1) | Log to Outreach Tracker |

Threshold of 60 is hardcoded in the IF node condition. No dynamic configuration is possible without editing the node.

### Check Email Valid (node 20)
| Condition | Branch | Destination |
|---|---|---|
| `$json.email_valid === true` | True (output 0) | Send Pitch Email |
| `$json.email_valid` is false or absent | False (output 1) | Log to Outreach Tracker2 |

`email_valid` is set in `Parse Pitch Email` by a regex test on `item.email || item.journalist_email` combined with a check that `subject.length > 0`.

---

## Loop Logic

`Loop Over Items` (`splitInBatches`) processes the merged journalist list in batches of 5.

| Parameter | Value |
|---|---|
| Batch size | 5 |
| Reset on each run | false |
| Loop-back input | `Send Pitch Email` output 0 → `Loop Over Items` input 0 |
| Loop exit | `Loop Over Items` output 0 ("done") → `Execution Summary` |

**Per-iteration behaviour:** Each batch of up to 5 journalists flows through Relevance Scorer → Parse Relevance Score → Filter Score → (email path or skip path). After `Send Pitch Email` (or one of the Log nodes), execution returns to the loop to begin the next batch. The loop exits when all items have been processed, at which point `Execution Summary` runs once for all accumulated items.

**Note:** Because the Merge node uses `combineAll`, the total items entering the loop may be large (see Known Limitations). The loop processes all of them regardless of count.

---

## Variables & Expressions

Key non-trivial expressions used across the workflow:

| Node | Expression | Purpose |
|---|---|---|
| Google News RSS Fetch | `{{ encodeURIComponent($('Normalize Input').item.json.target_industry + ' journalist reporter') }}` | Builds the RSS search query from the announcement's industry field |
| Journalist Relevance Scorer | `{{ $('Normalize Input').item.json.company_name }}` (and other fields) | Back-references the normalised announcement data from within the loop |
| Email Personalization AI | `{{ $node["Parse Press Release (Code)"].json.headline }}` | Injects the AI-generated headline into the email prompt |
| Email Personalization AI | `{{ $node["Fetch Journalist Seed DB"].json.name }}` | Pulls the journalist's name from the DB node, not the current item — this will always use the **first** journalist from the DB node's output, not the per-iteration journalist (see Known Limitations) |
| Send Pitch Email | `={{ $json.email_body.replace(/\n/g, '<br>') }}` | Converts newline characters to HTML line breaks for email rendering |
| Log to Outreach Tracker | `={{ $json.email_valid ? 'sent' : 'skipped' }}` | Determines the `email_status` column value |
| Log to Outreach Tracker | `={{ $json.email_valid ? '' : ($json.recipient_email ? 'invalid_email' : 'missing_email') }}` | Determines the `skip_reason` column value |
| Execution Summary | `all[0]?.json?.campaign_id` | Retrieves campaign ID from the first processed item |

---

## Credentials Used

| Node | Credential Name | Credential Type | Service |
|---|---|---|---|
| Google Sheets Trigger | Google Sheets Trigger account | `googleSheetsTriggerOAuth2Api` | Google Sheets |
| Fetch Journalist Seed DB | Google Sheets account | `googleSheetsOAuth2Api` | Google Sheets |
| Log to Outreach Tracker | Google Sheets account | `googleSheetsOAuth2Api` | Google Sheets |
| Log to Outreach Tracker2 | Google Sheets account | `googleSheetsOAuth2Api` | Google Sheets |
| OpenRouter Chat Model | OpenRouter account for AI Journal | `openRouterApi` | OpenRouter (AI) |
| OpenRouter Chat Model1 | OpenRouter account for AI Journal | `openRouterApi` | OpenRouter (AI) |
| OpenRouter Chat Model2 | OpenRouter account for AI Journal | `openRouterApi` | OpenRouter (AI) |
| Send Pitch Email | Gmail account | `gmailOAuth2` | Gmail |

All three OpenRouter sub-nodes share the same credential (`YCawzcwbZd7VllSz`). The Google Sheets Trigger uses a separate credential type from the read/write Google Sheets nodes.

---

## Input Data

**Trigger input:** A new row from `News_Trigger` — flat JSON object with one key per column. All values are strings as returned by the Google Sheets API.

**Expected minimum payload:**
```json
{
  "company_name": "Acme SaaS",
  "announcement_type": "Product Launch",
  "details": "Acme SaaS launches AutoFlow 2.0...",
  "target_industry": "HR Tech, SaaS, Future of Work",
  "tone": "professional",
  "contact_email": "press@acmesaas.com"
}
```

Optional fields (`spokesperson`, `website`, `embargo_date`) are handled by the Normalize Input node's fallback logic.

---

## Output Data

The workflow has multiple terminal outputs:

| Output | Node | Description |
|---|---|---|
| Campaign summary | Execution Summary | `{ campaign_id, total_journalists_scored, emails_sent, emails_skipped, average_relevance_score, run_completed_at, status }` |
| Sent outreach record | Log to Outreach Tracker / Log to Outreach Tracker2 | One row appended to `Outreach_Tracker` per journalist processed |
| Delivered email | Send Pitch Email | HTML email sent to `storysnippents@gmail.com` (hardcoded) |

The `Outreach_Tracker` tab accumulates all records across runs, with `campaign_id` as the grouping key.

---

## Data Transformation

| Stage | Transformation |
|---|---|
| Normalize Input | Trims all strings; lowercases `tone`; generates `campaign_id`; sets defaults for missing optional fields; adds `created_at` |
| Parse Press Release | Converts raw LLM text string → structured JSON object with named PR fields |
| Parse RSS to Journalist Leads | Converts raw RSS XML string → array of 20 structured lead objects with normalised `title`, `publication`, `article_url` |
| Merge | Two arrays joined with `combineAll` — potentially large Cartesian product |
| Parse Relevance Score | Merges `{ relevance_score, match_reason, suggested_angle }` onto the journalist's existing object via object spread |
| Parse Pitch Email | Extracts `{ subject_line, email_body, personalization_note }` from LLM output; adds `recipient_email`, `recipient_name`, `email_valid` flags |
| Log to Outreach Tracker | Maps ~19 fields from runtime data to named spreadsheet columns; truncates `email_body_preview` to 200 chars; sets `open_status: 'pending'`, `reply_status: 'pending'`, `followup_sent: FALSE`, `followup_count: 0` as static defaults |

---

## API Integrations / External Services

| Service | Node | Method | Endpoint / Details | Auth |
|---|---|---|---|---|
| Google News RSS | Google News RSS Fetch | GET | `https://news.google.com/rss/search?q=<query>&hl=en-US&gl=US&ceid=US:en` | None (public) |
| OpenRouter AI | OpenRouter Chat Model (×3) | POST | OpenRouter inference API | API key (`openRouterApi`) |
| Google Sheets API | Google Sheets Trigger, Fetch Journalist Seed DB, Log nodes (×2) | Poll / Read / Append | Spreadsheet ID `1pTOQnzKsUaADiSifm5r4Pc64A0K5Ewn80Ek-wSLz5I8` | OAuth2 |
| Gmail API | Send Pitch Email | Send | Gmail send via OAuth2 | OAuth2 (`gmailOAuth2`) |

---

## AI/LLM Integration

Three AI Agent nodes, each with a dedicated OpenRouter LLM sub-node:

### Press Release Generator
- **Model:** `nvidia/nemotron-3-super-120b-a12b:free`
- **Role:** Generates a complete 10-field press release JSON from raw announcement data
- **System prompt key constraint:** AP Style, strict JSON-only output, no markdown, no preamble
- **Output consumed by:** `Parse Press Release (Code)` → all downstream nodes reading `.headline`, `.full_press_release`, etc.

### Journalist Relevance Scorer
- **Model:** `nvidia/nemotron-nano-12b-v2-vl:free`
- **Role:** Scores journalist-to-announcement fit 0–100; provides `match_reason` and `suggested_angle`
- **Runs:** Once per journalist, inside the batch loop
- **Output consumed by:** `Parse Relevance Score` → `Filter Score >= 60`

### Email Personalization AI
- **Model:** `arcee-ai/trinity-large-preview:free`
- **Role:** Writes a personalised pitch email (subject + body) for each journalist scoring ≥ 60
- **Key output fields:** `subject_line` (≤ 10 words), `email_body` (≤ 150 words), `personalization_note`
- **Output consumed by:** `Parse Pitch Email` → `Check Email Valid` → `Send Pitch Email`

All three models are accessed via the OpenRouter API using a single shared credential (`OpenRouter account for AI Journal`), using the `:free` tier variants. No fallback model is configured if a free-tier model is unavailable.

---

## Error Handling

| Node | Behaviour on Failure |
|---|---|
| Parse Press Release (Code) | Returns `{ error: "Invalid JSON from AI", raw: <string> }` — workflow continues but downstream nodes will receive undefined fields |
| Parse RSS to Journalist Leads (Code) | Returns `[{ json: { no_results: true } }]` — workflow continues with only DB journalists |
| Parse Relevance Score (Code) | Defaults `relevance_score` to 0 on parse failure — journalist will be filtered out by the score gate |
| Parse Pitch Email (Code) | Substitutes generic subject/body fallback; sets `email_valid: false` if no email found — journalist is routed to skip log |
| Workflow-level error workflow | Not configured (`settings` contains no `errorWorkflow` key) |
| `continueOnFail` | Not explicitly set on any node — defaults to false; unhandled node errors will halt execution |

The three Code parser nodes (5, 13, 16) each implement try/catch to handle AI output parse failures gracefully. No other nodes have explicit error handling configured.

---

## Retry Strategy

No retry configuration (`retryOnFail`, `maxTries`, `waitBetweenTries`) is set on any node in this workflow. If the OpenRouter API returns a 429 rate-limit error or the Gmail send fails transiently, the execution will fail at that node without retrying. This is a known operational gap for production use.

---

## Logging

The `Outreach_Tracker` Google Sheet tab serves as the outreach log — every journalist processed (sent or skipped) is appended as a row. The `Execution Summary` node produces a per-run aggregate but does not write to a sheet — its output is only visible in n8n's execution log. n8n's own execution history is the only other record.

---

## Execution Order

`settings.executionOrder: "v1"` — connections-based execution order. Parallel branches (RSS fetch and DB read both triggered from `Parse Press Release`) execute as independently as the n8n engine allows, then converge at `Merge`.

---

## Security Considerations

- **Trigger:** Google Sheets polling — no inbound webhook; no webhook auth considerations apply.
- **OpenRouter API key:** Shared across all three AI nodes under one credential. A compromised key would expose all AI API usage.
- **Gmail OAuth2:** Grants send access to the connected Gmail account. Scope should be limited to `gmail.send` if possible.
- **Spreadsheet access:** Both OAuth2 credentials grant broad Google Sheets access. Scope down to the specific spreadsheet ID if the Google Cloud project permits.
- **Hardcoded test recipient email** (`storysnippents@gmail.com`) in the Gmail node — if this workflow is activated as-is, all emails will go to that address regardless of journalist data. This must be changed before production use.
- *Further security review: [To be provided by workflow owner / security reviewer]*

---

## Performance Considerations

- **Merge `combineAll` risk:** If the RSS feed returns 20 leads and the `Journalist_DB` has 200 rows, `combineAll` produces 20 × 200 = 4,000 combined items entering the loop. This is almost certainly not the intended behaviour (see Known Limitations). A simple `append` mode would produce 20 + 200 = 220 items.
- **Loop batch size = 5:** Each batch makes 1 OpenRouter call (scorer) + potentially 1 more (email AI) = up to 2 LLM calls per batch. With 220 items and batch size 5, that's 44 loop iterations and up to 88 LLM calls per run. With 4,000 items, it's 800 iterations and up to 1,600 LLM calls.
- **Free-tier LLM rate limits:** OpenRouter free models have rate limits that are not configurable within n8n's current workflow. No `Wait` nodes are present to throttle between batches.
- **Execution time estimates:** [To be provided by workflow owner — requires execution data]

---

## Troubleshooting

| Symptom | Likely Cause | Where to Look |
|---|---|---|
| Workflow doesn't fire on new row | Polling interval not reached, or trigger credential expired | Check n8n execution log; re-authenticate `Google Sheets Trigger account` |
| Press release output is `{ error: "Invalid JSON from AI" }` | LLM returned markdown-wrapped JSON or incomplete output | Check `raw` field in `Parse Press Release` output; try a simpler `details` value; verify OpenRouter API key is valid |
| All journalists get score 0 | Nemotron-Nano-12B unavailable or API key exhausted | Check OpenRouter dashboard; substitute another free model in `OpenRouter Chat Model1` |
| All emails going to `storysnippents@gmail.com` instead of journalists | `sendTo` is hardcoded — known issue | Change `sendTo` in `Send Pitch Email` to `={{ $json.recipient_email }}` |
| `email_valid` is always false | `Journalist_DB` rows are missing an `email` column, or column name differs | Check column headers in `Journalist_DB`; ensure `email` field is populated |
| Outreach Tracker fills up extremely fast | Merge using `combineAll` instead of `append` — multiplying records | Change Merge node to `mode: append` |
| Loop never terminates | `Send Pitch Email` fails and does not return to loop; or loop-back connection broken | Check `Send Pitch Email` execution; consider enabling `continueOnFail` on that node |
| OpenRouter returns 429 | Free-tier rate limit hit | Add a `Wait` node (5–10 seconds) between `Parse Relevance Score` and `Journalist Relevance Scorer`, or reduce batch size |

---

## Known Limitations

1. **Hardcoded Gmail recipient.** `Send Pitch Email` has `sendTo` hardcoded to `storysnippents@gmail.com`. Emails are not sent to the journalist's actual email address. This must be changed to `={{ $json.recipient_email }}` before production use.

2. **Merge mode `combineAll` is likely incorrect.** The Merge node joins RSS leads and DB journalists using `combineAll` (Cartesian product), not `append`. This is almost certainly unintended — it multiplies the record count dramatically (e.g. 20 RSS × 200 DB = 4,000 items). Should likely be `mode: "append"` or `combineBy: "combineByPosition"`.

3. **Email Personalization AI journalist reference.** The prompt uses `$node["Fetch Journalist Seed DB"].json.name` (and `.publication`, `.beat`). This always refers to the **last output** of the `Fetch Journalist Seed DB` node — likely the same journalist for every loop iteration — rather than the current journalist being processed. The per-journalist data should come from `$json.name`, `$json.publication`, `$json.beat` (the merged/loop item).

4. **No workflow-level error handler.** If any node throws an unhandled error (e.g. Gmail auth fails, Google Sheets quota exceeded), the entire execution halts with no notification.

5. **No retry logic.** All nodes use default (no retry) settings. Transient API errors cause immediate failure.

6. **Free-tier model availability.** All three LLM models are OpenRouter free-tier (`:free` suffix). Free models may be unavailable, throttled, or deprecated without notice.

7. **RSS-only journalist leads lack contact details.** RSS leads include only `publication` and `article_title` — no `name`, `email`, or `beat`. The email validation gate will always fail for pure RSS leads unless they are enriched from the DB (which the current Merge logic would handle via `combineAll`, though unintentionally).

8. **Relevance score threshold hardcoded.** The 60-point threshold in `Filter Score >= 60` cannot be changed without editing the node. It is not driven by a configurable variable or sheet value.

9. **No duplicate-outreach prevention.** Nothing in the workflow checks whether a journalist has already been emailed in a previous campaign run. The same journalist will be pitched for every announcement.

10. **Workflow not active at time of export.** The JSON export shows `active: false`. The workflow must be manually activated in n8n before it will respond to sheet triggers.

---

## Assumptions

- The `Journalist_DB` tab contains a column named exactly `email` (and `name`, `publication`, `beat`, `relevance_tags`) — these are what the code and prompt expressions reference.
- The `target_industry` field is sufficiently descriptive for a Google News RSS search to return relevant journalist articles (e.g. "HR Tech, SaaS" works; very niche industries may return zero results).
- The `Normalize Input` node's assumption that `$input.first()` is sufficient is valid because the trigger fires one row at a time (each `rowAdded` event produces a single item).
- The three AI models (Nemotron-Super-120B, Nemotron-Nano-12B, Trinity-Large-Preview) remain available on OpenRouter's free tier. If any are deprecated, the corresponding node will need a model update.
- `combineAll` in the Merge node is assumed to be a configuration oversight (intending `append`) based on the workflow's stated purpose and the downstream behaviour described. If `combineAll` is intentional, the data model and expected record counts would need to be revised significantly.
- Execution order v1 means parallel branches (RSS fetch and DB read) may not complete in a predictable sequence; the Merge node waits for both inputs before proceeding.

---

## Version History / Change Log

_[To be provided by workflow owner]_

---

## References

- [n8n splitInBatches / Loop Over Items documentation](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.splitinbatches/)
- [n8n AI Agent node documentation](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent/)
- [OpenRouter free model directory](https://openrouter.ai/models?q=:free)
- [Google News RSS search URL format](https://news.google.com/rss/search?q=QUERY)
- _[Additional references to be provided by workflow owner]_

---

## Maintenance Notes

_[To be provided by workflow owner]_

**Suggested near-term actions before production activation:**
1. Fix `Send Pitch Email` → change `sendTo` from hardcoded address to `={{ $json.recipient_email }}`
2. Fix `Merge` node → change `combineBy` to `append` (or validate that `combineAll` is intended)
3. Fix `Email Personalization AI` prompt → change `$node["Fetch Journalist Seed DB"].json.name` references to `$json.name`, `$json.publication`, `$json.beat`
4. Add `continueOnFail: true` and retry settings to `Send Pitch Email`
5. Configure a workflow-level error handler (`settings.errorWorkflow`) to notify on failure
6. Add a Wait node between batches to manage OpenRouter free-tier rate limits
7. Activate the workflow in n8n

