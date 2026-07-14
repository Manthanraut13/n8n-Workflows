# AI Meeting Intelligence — Workflow Documentation

| Field | Value |
|---|---|
| **Workflow Name** | AI Meeting Intelligence |
| **Workflow ID** | `bJLQhSX8DS5JT2P2` |
| **Version** | v1.0 (initial documentation) — `versionId: 9282a0f0-bd90-44af-90ea-58440a861bb1` |
| **Owner** | _[To be provided by workflow owner]_ |
| **Workflow Last Saved** | _[Not available in export — `updatedAt` and `createdAt` not set]_ |
| **Document Generated** | 2026-07-13 |
| **Active** | No (workflow is currently inactive in the exported state) |
| **Execution Order** | v1 |

---

## Purpose

This workflow automates the post-meeting intelligence pipeline. After a meeting ends, a transcript is submitted via HTTP webhook and the workflow validates, cleans, and enriches it before passing it to an AI agent for structured extraction. The AI produces a meeting summary, key topics, decisions, action items (with owners and deadlines), risks/blockers, and follow-up items — all in a strict JSON format. The validated AI output is then written to a Notion database page and a Slack recap message is sent to the configured channel/user.

---

## Business Use Case

> _Inferred — please confirm or replace._

Teams in medium-to-large organisations spend a significant proportion of working time in meetings, yet key decisions and action items often go uncaptured or are inconsistently documented. This workflow eliminates manual note-taking by automatically processing meeting transcripts into structured, searchable records in Notion and immediate team notifications in Slack, reducing meeting-related follow-up overhead and preventing missed commitments.

---

## Workflow Summary

A meeting transcript is received via a `POST` webhook at the path `/meeting-transcript`. The **Input Validator & Normalizer** enforces required fields and normalises the payload (email casing, date formats, participant list). The cleaned payload is passed to **Transcript Preprocessor**, which removes timestamps and filler words, chunks oversized transcripts, detects speaker names, and computes word/char counts. **Metadata Extractor** then enriches the item with derived fields: a unique `run_id`, formatted date, ISO week/month, and formatted participant string.

The enriched transcript is fed to the **AI Analysis Agent**, a LangChain-powered n8n agent backed by the **OpenRouter Chat Model** (`arcee-ai/trinity-mini:free`) and a **Structured Output Parser** that enforces the JSON schema. The agent returns a six-field structured object. The **JSON Output Parser** code node handles three possible response formats (structured LangChain output, Claude API format, or OpenAI API format), validates the schema, enriches action items with generated IDs and normalised deadlines, and sets a `parse_success` boolean flag.

An **If** gate checks `parse_success`. On `true`, the **Notion Page Creator** writes a new database page to the "Meeting Intelligence" Notion database and the **Slack Meeting Recap** node sends a Block Kit message to the configured Slack user/channel. The `false` branch is currently unconnected — see [Error Handling](#error-handling).

---

## Workflow Diagram

See the attached canvas screenshot (`AI_Meeting_Intelligence.png`).

**Shape:** Linear pipeline with one conditional branch at the end.

```
[Webhook Receiver]
       │ POST /meeting-transcript
       ▼
[Input Validator & Normalizer]   ← Code node: schema enforcement + normalisation
       │
       ▼
[Transcript Preprocessor]        ← Code node: clean text, chunk, detect speakers
       │
       ▼
[Metadata Extractor]             ← Code node: derived fields, run_id, formatted strings
       │
       ▼
[AI Analysis Agent]  ◄── [OpenRouter Chat Model]  (arcee-ai/trinity-mini:free)
       │             ◄── [Structured Output Parser]
       ▼
[JSON Output Parser]             ← Code node: multi-format parser + action item enrichment
       │
       ▼
[If]  parse_success == true?
  ├── TRUE  ──► [Notion Page Creator]  ──► [Slack Meeting Recap]
  └── FALSE ──► (unconnected — no error path configured)
```

---

## Trigger Details

| Field | Value |
|---|---|
| **Node Name** | Webhook Receiver |
| **Node Type** | `n8n-nodes-base.webhook` (v2.1) |
| **Method** | `POST` |
| **Path** | `meeting-transcript` |
| **Full Webhook URL** | `https://<your-n8n-host>/webhook/meeting-transcript` |
| **Authentication** | None configured in this export — see [Security Considerations](#security-considerations) |
| **Response Mode** | Default (responds immediately on receipt; does not wait for downstream nodes to complete unless "Respond to Webhook" node is added) |
| **Binary Data** | Not enabled |

The workflow is started whenever an HTTP `POST` request is sent to the webhook endpoint. The caller is expected to send a JSON body with meeting data. Typical callers: Zoom `recording.completed` webhook, Fireflies.ai transcript callback, Microsoft Teams Graph API notification, Google Workspace Events API, or a manual test script.

---

## Execution Flow

1. **Webhook Receiver** accepts a `POST` request to `/meeting-transcript` and passes the raw `body` object downstream.

2. **Input Validator & Normalizer** reads `$input.first().json.body` — the parsed request body. It checks that `meeting_id`, `meeting_title`, `meeting_date`, `transcript`, and `participants` are all present and non-empty, that `participants` is a non-empty array, and that `transcript` is at least 100 characters. On any failure, it throws a `VALIDATION_ERROR` which halts the execution. On success, it normalises all participant emails to lowercase, converts `meeting_date` to ISO 8601, extracts `organizer_email`, and passes a cleaned, augmented JSON item downstream.

3. **Transcript Preprocessor** takes the cleaned item, strips timestamp patterns (e.g. `[00:01:32]`), removes filler words (`um`, `uh`, `er`, `ah`, `like,`, `you know,`), collapses excess whitespace, detects unique speaker names via regex, and chunks the transcript into 80,000-character segments if it exceeds that limit. It appends `transcript_cleaned`, `transcript_chunks`, `chunk_count`, `is_chunked`, `detected_speakers`, `word_count`, `char_count`, and `preprocessed_at` to the item.

4. **Metadata Extractor** builds derived fields from the enriched item: a unique `run_id` (`run_<meeting_id>_<epoch>`), a human-readable formatted date, ISO week key (e.g. `47-2024`), month key (e.g. `2024-11`), a formatted participant list string, and participant count. It also injects three environment-variable references: `NOTION_MEETINGS_DB_ID`, `ASANA_PROJECT_ID`, and `SLACK_MEETINGS_CHANNEL` (these are embedded as literal template strings in the code — see [Environment Variables](#environment-variables)).

5. **AI Analysis Agent** receives the enriched item and constructs a prompt using `meeting_title`, `meeting_date_formatted`, `duration_minutes`, `platform`, `participant_list_formatted`, and `transcript_cleaned`. The LangChain agent calls the **OpenRouter Chat Model** with the system prompt and user message. The **Structured Output Parser** sub-node enforces the six-field JSON schema on the model's response.

6. **JSON Output Parser** receives the agent's output. It handles three response structures: `data.output` (LangChain structured path, the expected path when the Structured Output Parser is active), `data.content[0].text` (Claude direct API format), and `data.choices[0].message.content` (OpenAI direct API format). It strips markdown code fences if present, validates all six required fields exist, enriches each action item with a `task_id`, `owner_email` (resolved from the participant list), a normalised `deadline` date, and sets `parse_success: true`. If validation fails, `parse_success: false` and `parse_error` are set instead.

7. **If** evaluates `$json.parse_success === true` (boolean, strict, case-sensitive).
   - **True branch:** proceeds to output nodes.
   - **False branch:** currently unconnected — execution ends silently with no error notification.

8. **Notion Page Creator** creates a new page in the "Meeting Intelligence" Notion database (`3266ce9c-8989-801c-94d2-dd2860315dde`). The page title is `<meeting_title> -- <meeting_date>`. It maps `Platform` (select), `Duration (min)` (number), `Participants` (number), `Action Items Count` (number from `$json.output.action_items.length`), `Decision Count` (number from `$json.output.decisions.length`), `Status` (hardcoded to `Done`), `AI Summary` (rich text from `$json.output.meeting_summary`), and `Meeting Date` (rich text).

9. **Slack Meeting Recap** sends a Block Kit message to the Slack user `USLACKBOT` (slackbot). The blocks include a header, a context bar with meeting date/duration/participant count, a divider, a Summary section, an Action Items count section, and a "View in Notion" button. Values are read from `$json.name`, `$json.property_meeting_date`, `$json.property_duration_min`, `$json.property_ai_summary`, `$json.property_action_items_count`, and `$json.property_notion_url` — these are the Notion API response fields from the previous node.

---

## Node List

| # | Node Name | Type | Role | Disabled |
|---|---|---|---|---|
| 1 | Webhook Receiver | Webhook | Receives the meeting transcript POST payload | No |
| 2 | Input Validator & Normalizer | Code (JS) | Schema validation + payload normalisation | No |
| 3 | Transcript Preprocessor | Code (JS) | Cleans transcript text; detects speakers; chunks long transcripts | No |
| 4 | Metadata Extractor | Code (JS) | Builds derived fields (run_id, formatted date, week/month keys, participant string) | No |
| 5 | AI Analysis Agent | LangChain Agent | Orchestrates LLM call with structured output enforcement | No |
| 6 | OpenRouter Chat Model | LangChain LM (OpenRouter) | Provides `arcee-ai/trinity-mini:free` as the language model sub-node | No |
| 7 | Structured Output Parser | LangChain Output Parser | Enforces JSON schema on the LLM's response | No |
| 8 | JSON Output Parser | Code (JS) | Multi-format response parser; action item enrichment; sets `parse_success` | No |
| 9 | If | If | Routes on `parse_success`; true → output nodes, false → (unconnected) | No |
| 10 | Notion Page Creator | Notion | Creates a database page in the "Meeting Intelligence" Notion DB | No |
| 11 | Slack Meeting Recap | Slack | Posts a Block Kit recap message to a Slack user/channel | No |

---

## Node-by-Node Description

### Webhook Receiver

Listens on `POST /meeting-transcript`. When a request arrives, n8n parses the JSON body and passes it as `$input.first().json`. The body itself is under `.body` (i.e. `$json.body`) because n8n wraps webhook payloads. This is why the next node accesses `$input.first().json.body` rather than `$input.first().json` directly. No authentication is configured on this node in the current export.

### Input Validator & Normalizer

Reads `$input.first().json.body`. Performs three categories of checks:

- **Presence checks:** `meeting_id`, `meeting_title`, `meeting_date`, `transcript`, `participants` must all be non-empty strings/values. Throws `VALIDATION_ERROR: Missing required fields: <list>` if any fail.
- **Type checks:** `participants` must be a non-empty array; `transcript` must be ≥ 100 characters; `meeting_date` must parse to a valid `Date` object.
- **Normalisation:** participant emails are lowercased and trimmed; entries without a valid `@` are filtered out; `platform` is lowercased (defaulting to `"unknown"`); `duration_minutes` is parsed to integer; `meeting_date` is converted to ISO 8601.

Adds `participant_emails[]`, `organizer_email` (first participant with role `"organizer"`, else first in list), and `validated_at` timestamp.

### Transcript Preprocessor

Receives the normalised item. Applies sequential regex transformations to `data.transcript`:

1. Strip timestamps: `/\[?\d{1,2}:\d{2}(:\d{2})?\]?/g`
2. Strip fillers: `/\b(um|uh|er|ah|like,|you know,)\b/gi`
3. Collapse whitespace and excess blank lines.

Detects unique speaker names by matching `Firstname Lastname:` patterns at line starts. Chunks the text into 80,000-character segments if needed. Computes `word_count` and `char_count`. Appends: `transcript_cleaned`, `transcript_chunks[]`, `chunk_count`, `is_chunked`, `detected_speakers[]`, `word_count`, `char_count`, `preprocessed_at`.

### Metadata Extractor

Derives structured metadata from the validated, preprocessed item:

- `run_id`: `run_<meeting_id>_<Date.now()>` — unique per execution.
- `meeting_date_formatted`: long English locale string (e.g. `"Wednesday, November 20, 2024"`).
- `meeting_week`: ISO week and year (e.g. `"47-2024"`).
- `meeting_month`: year-month key (e.g. `"2024-11"`).
- `participant_list_formatted`: `"Name (email); Name (email); ..."`.
- `participant_count`: integer.
- Three env-variable placeholders injected as literal strings: `{{$env.NOTION_MEETINGS_DB_ID}}`, `{{$env.ASANA_PROJECT_ID}}`, `{{$env.SLACK_MEETINGS_CHANNEL}}`.

> **Note:** The env-variable references in the code are template literals, not n8n expressions — they will not resolve at runtime. See [Environment Variables](#environment-variables) for the fix required.

### AI Analysis Agent

The central intelligence node. Uses n8n's LangChain Agent with `promptType: "define"` (fully custom prompt). Constructs the user message from the enriched item fields and passes it to the **OpenRouter Chat Model** sub-node. The **Structured Output Parser** sub-node is attached to enforce the response schema. The agent has `hasOutputParser: true`, so the structured response is placed on `data.output` in the agent's output item — the primary path handled by the next node.

**System message summary:** Instructs the model to act as an expert meeting analyst, output ONLY valid JSON (no preamble, no markdown fences), and follow six strict rules: extract only what is stated; return empty arrays for absent fields; name the owner of each action item; preserve deadline phrasing verbatim; distinguish decisions from action items; flag only explicitly stated risks.

**User message template:**
```
Meeting Title: {{$json.meeting_title}}
Meeting Date: {{$json.meeting_date_formatted}}
Duration: {{$json.duration_minutes}} minutes
Platform: {{$json.platform}}
Participants: {{$json.participant_list_formatted}}

---
TRANSCRIPT:
{{$json.transcript_cleaned}}
```

### OpenRouter Chat Model

Sub-node attached to **AI Analysis Agent** on the `ai_languageModel` channel. Provides `arcee-ai/trinity-mini:free` via the OpenRouter API. Temperature is set to `0.2` (low, for deterministic factual extraction). Max tokens is `4096`. Uses the credential `OpenRouter account for AI Journal`.

### Structured Output Parser

Sub-node attached to **AI Analysis Agent** on the `ai_outputParser` channel. Infers the JSON schema from a provided example object (`jsonSchemaExample`). The schema defines:
- `meeting_summary`: string
- `key_topics`: string array
- `decisions`: string array
- `action_items`: array of `{ task, owner, deadline, priority }` objects
- `risks_or_blockers`: string array
- `followups`: string array

When this parser is active and the model complies, the structured object is available at `$json.output` in the agent's output.

### JSON Output Parser

Handles the agent's output defensively across three possible formats:

1. **`data.output`** — LangChain structured path (expected normal path with Structured Output Parser active).
2. **`data.content[0].text`** — Claude/Anthropic direct API response format.
3. **`data.choices[0].message.content`** — OpenAI direct API response format.

For cases 2 and 3, a `cleanJSON()` helper strips markdown code fences before `JSON.parse()`. If the structure is unrecognised, throws `PARSE_ERROR: Unexpected AI response structure`.

After parsing, validates all six required fields are present. If any are missing, returns `parse_success: false` with `parse_error` set. On success, enriches each action item:
- `task_id`: `task_<meeting_id>_001`, `002`, etc.
- `owner_email`: resolved by case-insensitive name match against `data.participants[]`.
- `deadline`: normalised — `"tomorrow"` → meeting date + 1 day; `"next week"` → + 7 days; valid ISO string → formatted; otherwise kept as raw string.
- `deadline_raw`: original phrasing preserved.
- `priority`: falls back to `"normal"` if absent.

Returns `ai_output` (the enriched parsed object), `parse_success: true`, `action_item_count`, and `parsed_at`.

### If

Evaluates a single boolean condition: `{{ $json.parse_success }} equals true` (strict, case-sensitive, combinator: AND).

- **Output 0 (true):** item proceeds to **Notion Page Creator**.
- **Output 1 (false):** currently unconnected. Execution terminates silently.

### Notion Page Creator

Creates a database page in the Notion database "Meeting Intelligence" (`3266ce9c-8989-801c-94d2-dd2860315dde`). Uses credential `n8n bot` (type: `notionApi`).

**Page title:** `<meeting_title> -- <meeting_date>` (both pulled from `Metadata Extractor` node via `$('Metadata Extractor').item.json.*`).

**Properties mapped:**

| Notion Property | Type | Source Expression |
|---|---|---|
| Name | title | `$('Metadata Extractor').item.json.meeting_title` |
| Platform | select | `$('Metadata Extractor').item.json.platform` |
| Duration (min) | number | `$('Metadata Extractor').item.json.duration_minutes` |
| Participants | number | `$('Metadata Extractor').item.json.participants.length` |
| Action Items Count | number | `$json.output.action_items.length` |
| Decision Count | number | `$json.output.decisions.length` |
| Status | status | `"Done"` (hardcoded) |
| AI Summary | rich_text | `$json.output.meeting_summary` |
| Meeting Date | rich_text | `$('Metadata Extractor').item.json.meeting_date` |

> **Note:** `Action Items Count` and `Decision Count` reference `$json.output.*`, which is the raw structured output from the LangChain agent. This is consistent with the `data.output` path in the JSON Output Parser — however, if the Notion node runs after the JSON Output Parser, `$json` will be the JSON Output Parser's output where the enriched data is on `$json.ai_output`, not `$json.output`. See [Known Limitations](#known-limitations).

### Slack Meeting Recap

Sends a direct message to the Slack user `USLACKBOT` (the Slack bot itself — this is likely a placeholder recipient and should be updated to a real channel or user). Uses credential `Slack account` (type: `slackApi`). Message type is `block` (Block Kit).

**Block structure:**
1. **Header:** `📋 Meeting Recap — {{ $json.name }}`
2. **Context:** `📅 <meeting_date> · ⏱️ <duration_min> min · 👥 <participants_count> participants`
3. **Divider**
4. **Section:** `*Summary*\n<ai_summary>`
5. **Section:** `*✅ Action Items (<count>)*`
6. **Actions:** Button — `View in Notion` linking to `{{ $json.property_notion_url }}`

All data fields (`$json.name`, `$json.property_meeting_date`, `$json.property_ai_summary`, etc.) are read from the Notion node's response — these are the Notion API response properties, not the meeting payload fields.

---

## Node Configuration

### Input Validator & Normalizer — Key Code

```javascript
const body = $input.first().json.body;  // ← webhook body nesting

const required = ['meeting_id', 'meeting_title', 'meeting_date', 'transcript', 'participants'];
const missing = required.filter(f => !body[f] || String(body[f]).trim() === '');
if (missing.length > 0) throw new Error(`VALIDATION_ERROR: Missing required fields: ${missing.join(', ')}`);
if (!Array.isArray(body.participants) || body.participants.length === 0)
  throw new Error('VALIDATION_ERROR: participants must be a non-empty array');
if (body.transcript.length < 100)
  throw new Error('VALIDATION_ERROR: Transcript too short to process (min 100 chars)');

const meetingDate = new Date(body.meeting_date);
if (isNaN(meetingDate.getTime())) throw new Error('VALIDATION_ERROR: Invalid meeting_date format');

// … normalisation and return
```

### Transcript Preprocessor — Key Regex Operations

```javascript
raw = raw.replace(/\[?\d{1,2}:\d{2}(:\d{2})?\]?/g, '');          // strip timestamps
raw = raw.replace(/\b(um|uh|er|ah|like,|you know,)\b/gi, '');     // strip fillers
raw = raw.replace(/[ \t]+/g, ' ').replace(/\n{3,}/g, '\n\n').trim(); // collapse whitespace

const speakers = [...new Set(
  raw.match(/^([A-Z][a-z]+ [A-Z][a-z]+):/gm)?.map(s => s.replace(':', '').trim()) || []
)];  // detect "Firstname Lastname:" patterns

const MAX_CHARS = 80000;
// chunk into 80k segments if needed
```

### AI Analysis Agent — Full System Prompt

```
You are an expert meeting analyst and executive assistant with deep expertise in
business communication, project management, and organizational decision-making.

Your task is to analyze the provided meeting transcript and extract structured
intelligence. You must respond with ONLY a valid JSON object — no preamble, no
explanation, no markdown fences, no additional text of any kind.

STRICT RULES:
1. Extract ONLY information explicitly stated in the transcript. Do NOT invent,
   infer, or hallucinate details.
2. If a field has no corresponding content in the transcript, return an empty
   array [] or empty string "".
3. For action items, identify the specific person named as responsible. If no
   person is named, use "Team" as the owner.
4. For deadlines, preserve the exact phrasing from the transcript
   (e.g., "by EOD Friday", "next sprint", "Q1 2025").
5. Distinguish between DECISIONS (things agreed upon) and ACTION ITEMS (tasks).
6. Risks and blockers must be explicitly flagged as problems or dependencies.

OUTPUT SCHEMA:
{
  "meeting_summary": "string — 3-5 sentence executive summary",
  "key_topics": ["string array — max 8 items"],
  "decisions": ["string array — phrased as completed outcomes"],
  "action_items": [
    { "task": "", "owner": "", "deadline": "", "priority": "high|normal|low" }
  ],
  "risks_or_blockers": ["string array"],
  "followups": ["string array"]
}

CONTEXT: Professional business meeting. Participants identified by "Firstname Lastname:"
at the start of speech turns. Treat content as confidential.
```

### Notion Page Creator — Property Mappings

| Property Key | Type | Value Expression |
|---|---|---|
| `Name\|title` | title | `={{ $('Metadata Extractor').item.json.meeting_title }}` |
| `Platform\|select` | select | `={{ $('Metadata Extractor').item.json.platform }}` |
| `Duration (min)\|number` | number | `={{ $('Metadata Extractor').item.json.duration_minutes }}` |
| `Participants\|number` | number | `={{ $('Metadata Extractor').item.json.participants.length }}` |
| `Action Items Count\|number` | number | `={{ $json.output.action_items.length }}` |
| `Decision Count\|number` | number | `={{ $json.output.decisions.length }}` |
| `Status\|status` | status | `Done` (literal) |
| `AI Summary\|rich_text` | rich_text | `={{ $json.output.meeting_summary }}` |
| `Meeting Date\|rich_text` | rich_text | `={{ $('Metadata Extractor').item.json.meeting_date }}` |

### OpenRouter Chat Model — Settings

| Parameter | Value |
|---|---|
| Model | `arcee-ai/trinity-mini:free` |
| Max Tokens | `4096` |
| Temperature | `0.2` |
| Credential | `OpenRouter account for AI Journal` |

---

## Branching Logic

Only one conditional node exists in this workflow.

### If — Parse Success Gate

| Field | Value |
|---|---|
| **Left Value** | `{{ $json.parse_success }}` |
| **Operator** | `boolean equals` |
| **Right Value** | `true` |
| **Case Sensitive** | Yes |
| **Combinator** | AND |

| Branch | Leads To | Condition |
|---|---|---|
| True (output 0) | **Notion Page Creator** | `parse_success` is exactly `true` |
| False (output 1) | _(unconnected)_ | `parse_success` is `false` or missing |

When `parse_success` is `false`, the false branch fires but nothing is connected — the execution ends without any error notification, Notion write, or Slack message. This is a gap that should be addressed (see [Known Limitations](#known-limitations) and [Error Handling](#error-handling)).

---

## Variables & Expressions

Key expressions used across nodes:

| Node | Expression | Computes |
|---|---|---|
| Metadata Extractor | `` `run_${data.meeting_id}_${Date.now()}` `` | Unique execution identifier |
| Metadata Extractor | `date.toLocaleDateString('en-US', {...})` | Human-readable date string |
| Metadata Extractor | `getISOWeek(date)` | ISO week number (custom function) |
| AI Agent (user prompt) | `{{$json.meeting_title}}`, `{{$json.transcript_cleaned}}` | Injects enriched fields into the LLM prompt |
| Notion Page Creator | `$('Metadata Extractor').item.json.meeting_title` | Back-references the Metadata Extractor node output |
| Notion Page Creator | `$json.output.action_items.length` | References the raw LangChain agent output |
| JSON Output Parser | `resolveEmail(ownerName, participants)` | Fuzzy name-to-email resolution |
| JSON Output Parser | `normalizeDeadline(raw, meetingDate)` | Converts relative deadline phrases to ISO dates |
| Slack Meeting Recap | `$json.property_ai_summary` | Reads Notion API response property |

---

## Credentials Used

| Node | Credential Name | Credential Type | Service |
|---|---|---|---|
| OpenRouter Chat Model | OpenRouter account for AI Journal | `openRouterApi` | OpenRouter (LLM gateway) |
| Notion Page Creator | n8n bot | `notionApi` | Notion API |
| Slack Meeting Recap | Slack account | `slackApi` | Slack Web API |

No actual secret values are present in the export — only names and internal IDs.

---

## Environment Variables

The **Metadata Extractor** node embeds three references as JavaScript template literals inside the code string. Because they are inside a code node's JS (not in an n8n expression field), they will **not** be resolved by n8n's `$env` mechanism at runtime. They will be passed downstream as literal strings like `{{$env.NOTION_MEETINGS_DB_ID}}`.

| Variable Name | Used In | Purpose |
|---|---|---|
| `NOTION_MEETINGS_DB_ID` | Metadata Extractor (injected), Notion Page Creator (hardcoded) | Notion database ID |
| `ASANA_PROJECT_ID` | Metadata Extractor (injected only — no Asana node present) | Future: Asana project for task creation |
| `SLACK_MEETINGS_CHANNEL` | Metadata Extractor (injected only — Slack node uses hardcoded user) | Future: Slack channel ID for recap |

> **Action required:** The Notion node uses a hardcoded database ID directly in its configuration (`3266ce9c-8989-801c-94d2-dd2860315dde`), not `$env`. The `NOTION_MEETINGS_DB_ID` env var in the Metadata Extractor is therefore not currently consumed by anything. If portability across environments is needed, the Notion node's Database ID should be changed to `={{ $env.NOTION_MEETINGS_DB_ID }}`.

---

## Input Data

The workflow expects a JSON `POST` body with the following shape:

```json
{
  "meeting_id": "string (required) — unique identifier from the meeting platform",
  "meeting_title": "string (required) — human-readable meeting name",
  "meeting_date": "string (required) — any parseable date/datetime string",
  "duration_minutes": "number (optional) — meeting length; defaults to 0",
  "platform": "string (optional) — e.g. 'zoom', 'google-meet', 'teams'; defaults to 'unknown'",
  "participants": [
    {
      "name": "string — full name",
      "email": "string (required) — must contain '@'",
      "role": "string (optional) — e.g. 'organizer', 'attendee'"
    }
  ],
  "transcript": "string (required) — raw meeting transcript, min 100 chars",
  "recording_url": "string (optional) — URL to the recording"
}
```

The `participants` array must have at least one entry with a valid email. Entries without `@` in the email are silently filtered out.

---

## Output Data

### Terminal Node 1: Notion Page Creator

Creates a page in the Notion "Meeting Intelligence" database. The Notion API response becomes the input for the Slack node. Key fields available downstream from the Notion response:

| Notion Response Field | Slack Block Expression |
|---|---|
| `$json.name` | Page title (meeting title + date) |
| `$json.property_meeting_date` | Meeting date display |
| `$json.property_duration_min` | Duration display |
| `$json.property_ai_summary` | Summary text |
| `$json.property_action_items_count` | Action items count |
| `$json.property_notion_url` | URL for the "View in Notion" button |

### Terminal Node 2: Slack Meeting Recap

Posts a Block Kit message to `USLACKBOT`. No data is passed further downstream — this is the terminal node of the `true` branch and the workflow.

---

## Data Transformation

The data object grows additively as it passes through each code node. Key transformations:

| Stage | Input | Output |
|---|---|---|
| Input Validator | Raw webhook `body` object | Normalised meeting payload; ISO date; lowercase emails; `organizer_email` resolved |
| Transcript Preprocessor | `transcript` (raw) | `transcript_cleaned` (noise removed); `transcript_chunks[]`; speaker list; counts |
| Metadata Extractor | Validated + preprocessed item | Derived fields: `run_id`, `meeting_date_formatted`, `meeting_week`, `meeting_month`, formatted participant string |
| JSON Output Parser | LangChain agent output (`data.output`) | `ai_output` object with enriched `action_items[]` (IDs, emails, normalised deadlines); `parse_success`; `action_item_count` |

---

## API Integrations / External Services

| Service | Node | Method / Endpoint | Auth Type | Purpose |
|---|---|---|---|---|
| OpenRouter | OpenRouter Chat Model | POST `https://openrouter.ai/api/v1/chat/completions` | API Key (`openRouterApi`) | LLM inference |
| Notion API | Notion Page Creator | POST `/v1/pages` | Internal Integration Token (`notionApi`) | Create meeting record page |
| Slack Web API | Slack Meeting Recap | POST `chat.postMessage` | Bot Token (`slackApi`) | Send meeting recap message |

---

## AI/LLM Integration

| Attribute | Value |
|---|---|
| **Framework** | n8n LangChain Agent (`@n8n/n8n-nodes-langchain.agent`, v3) |
| **Provider** | OpenRouter |
| **Model** | `arcee-ai/trinity-mini:free` |
| **Temperature** | `0.2` |
| **Max Tokens** | `4096` |
| **Output Enforcement** | Structured Output Parser (JSON Schema Example) |
| **Prompt Style** | System + User (custom `define` type) |

The AI's task is **information extraction, not generation** — it must only surface what is explicitly stated in the transcript. The system prompt enforces this with strict rules and a hardcoded output schema embedded in the prompt itself. The Structured Output Parser provides a second enforcement layer at the LangChain framework level.

The JSON Output Parser handles the case where the Structured Output Parser succeeds (output is on `data.output`, already a JS object) and two fallback paths for raw text responses from Claude or OpenAI direct APIs — making the workflow resilient to model provider changes.

**AI Output Schema:**

```json
{
  "meeting_summary": "3–5 sentence executive summary",
  "key_topics": ["topic 1", "topic 2", "..."],
  "decisions": ["decision 1", "decision 2"],
  "action_items": [
    {
      "task": "specific task description",
      "owner": "First Last",
      "deadline": "verbatim from transcript",
      "priority": "high | normal | low"
    }
  ],
  "risks_or_blockers": ["risk 1", "risk 2"],
  "followups": ["deferred item 1"]
}
```

---

## Error Handling

| Failure Point | Behaviour | Configured? |
|---|---|---|
| Missing required fields (Node 2) | Throws `VALIDATION_ERROR` — n8n marks execution as failed | ✅ Yes |
| Invalid date (Node 2) | Throws `VALIDATION_ERROR: Invalid meeting_date format` | ✅ Yes |
| Short transcript (Node 2) | Throws `VALIDATION_ERROR: Transcript too short` | ✅ Yes |
| AI response unrecognised format (Node 8) | Throws `PARSE_ERROR: Unexpected AI response structure` — n8n marks as failed | ✅ Yes |
| AI output missing schema fields (Node 8) | Sets `parse_success: false`; If-gate routes to false branch | ✅ Yes — but false branch unconnected |
| If false branch (Node 9) | Execution terminates silently — no alert sent | ❌ Not handled |
| Notion API failure (Node 10) | n8n default: marks execution as failed; no `continueOnFail` set | ❌ Not handled |
| Slack API failure (Node 11) | n8n default: marks execution as failed; no `continueOnFail` set | ❌ Not handled |
| OpenRouter rate limit / model error | n8n default retry not configured; execution fails | ❌ Not handled |

No global error workflow is configured in `settings`. No `continueOnFail` is set on any node. No retry (`retryOnFail`, `maxTries`) is configured on any node, including the HTTP-based AI call.

---

## Retry Strategy

No retry logic is configured on any node in this workflow. This means:

- If OpenRouter returns a `429 Too Many Requests` or a transient `5xx`, the execution will fail immediately.
- If the Notion API is temporarily unavailable, the execution will fail with no retry.
- If the Slack API call fails, no retry will occur.

**Recommendation:** Enable `retryOnFail: true` with `maxTries: 3` and `waitBetweenTries: 2000` on the **OpenRouter Chat Model** node and the **Notion Page Creator** node at a minimum.

---

## Failure Notifications

No failure notification nodes are present. The `false` branch of the **If** node is unconnected. When `parse_success` is `false`, execution ends silently with no alert to any system.

**Recommendation:** Connect the If node's false branch to a Slack node posting to an `#errors` or `#workflow-alerts` channel, and/or configure a global error workflow in the n8n workflow settings (`Settings → Error Workflow`).

---

## Logging

No explicit logging nodes are present (e.g. no Google Sheets append, no database insert for execution records). The only log of executions is n8n's own execution history, which is subject to the instance's execution retention settings.

---

## Execution Order

Configured as `v1` in `settings.executionOrder`. In v1 mode, when multiple branches exist, n8n executes them in the order they are defined in the connections object rather than the legacy top-to-bottom visual order. This is the recommended mode for all new workflows.

---

## Dependencies

| Dependency | Type | Used By |
|---|---|---|
| OpenRouter API | External SaaS | OpenRouter Chat Model — if unavailable, AI analysis cannot run |
| Notion API | External SaaS | Notion Page Creator — if unavailable, no meeting record is created |
| Slack API | External SaaS | Slack Meeting Recap — if unavailable, no recap message is sent |
| `arcee-ai/trinity-mini:free` model | AI model via OpenRouter | AI Analysis Agent — if model is deprecated or quota exceeded, workflow fails |
| "Meeting Intelligence" Notion DB (`3266ce9c-...`) | External resource | Notion Page Creator — if DB is deleted or unshared from the integration, creation fails |

No sub-workflows are called.

---

## Security Considerations

- **Webhook authentication:** None is configured on the **Webhook Receiver** node. Any client that knows the webhook URL can submit arbitrary payloads. In production, add header-based authentication (e.g. `X-Webhook-Secret`) or use n8n's built-in Basic/JWT auth on the webhook node.
- **Input validation:** The Input Validator & Normalizer enforces required fields and types, which provides basic protection against malformed payloads, but does not sanitise content for injection attacks.
- **Credential storage:** All credentials (OpenRouter, Notion, Slack) are stored in n8n's credential store. The export contains only names and internal IDs — no secret values are exposed.
- **Transcript data:** Meeting transcripts may contain sensitive business information. Ensure the n8n instance, OpenRouter usage, and Notion database are all governed by appropriate data retention and access control policies.

_Further sign-off: [To be provided by security reviewer]_

---

## Performance Considerations

- **AI inference latency:** The OpenRouter call to `arcee-ai/trinity-mini:free` is the dominant latency factor. Free-tier models may have variable response times. For production SLAs, consider upgrading to a paid model (`anthropic/claude-3.5-sonnet` or `openai/gpt-4o`).
- **Transcript chunking:** Transcripts over 80,000 characters are chunked, but the current workflow only passes `transcript_chunks[0]` to the AI (the full array is available but the AI agent's user prompt references `transcript_cleaned`, which is the full concatenated string). For very long transcripts (>80k chars), the full cleaned text will still be sent — chunked multi-pass processing is not yet implemented.
- **Sequential execution:** All nodes execute sequentially. There is no parallel fan-out. If additional output nodes (Asana, Gmail, Sheets) are added in the future, consider adding them in parallel branches from the Notion node to reduce total execution time.

---

## Execution Time

_[To be provided by workflow owner — requires execution data from n8n execution history]_

---

## Resource Usage

_[To be provided by workflow owner — requires execution data]_

---

## Testing & Validation

_[To be provided by workflow owner]_

**Suggested test cases:**
1. **Happy path:** valid payload with a ~500 word transcript → verify Notion page created and Slack message sent.
2. **Missing field:** omit `transcript` from payload → verify `VALIDATION_ERROR` thrown and execution marked failed.
3. **Short transcript:** send a 50-char transcript → verify minimum-length check fires.
4. **Invalid date:** send `meeting_date: "not-a-date"` → verify date validation throws.
5. **AI parse failure:** mock a model response that omits `action_items` → verify `parse_success: false` and If node routes to false branch.
6. **Long transcript:** send a 90,000-char transcript → verify `is_chunked: true`, `chunk_count: 2`, and execution completes without error.

---

## Troubleshooting

| Symptom | Likely Cause | Where to Look |
|---|---|---|
| Execution fails at Node 2 with `VALIDATION_ERROR` | Missing or empty required field in POST body | Check payload; ensure `participants` is an array with at least one `@`-containing email |
| Execution fails at Node 2 with `VALIDATION_ERROR: Transcript too short` | `transcript` is less than 100 characters | Verify the transcript was properly populated before the webhook fired |
| Execution fails with `PARSE_ERROR: Unexpected AI response structure` | AI model returned an unrecognised format | Check OpenRouter logs; verify the model `arcee-ai/trinity-mini:free` is still available |
| Notion page not created, no error shown | `parse_success` was `false` and the If false branch is unconnected | Check n8n execution log for `parse_success: false` in the JSON Output Parser output; inspect `parse_error` field |
| Slack message shows `$json.name` literally (not resolved) | Notion node failed silently or the Slack expressions reference fields not present in Notion response | Verify the Notion property names match the Notion database schema exactly |
| Notion property values not populated | Property key names in Notion node don't match actual database column names | Cross-check `key` strings (e.g. `"Platform\|select"`) against the actual Notion database property names |
| `Action Items Count` always 0 in Notion | `$json.output.action_items` is undefined at the Notion node (because JSON Output Parser puts data on `$json.ai_output`) | Update Notion expressions to reference `$json.ai_output.action_items.length` — see [Known Limitations](#known-limitations) |
| Webhook returns immediately before Notion/Slack complete | Webhook Response Mode is not set to wait for completion | Add a "Respond to Webhook" node at the end, or change response mode to "Last Node" |

---

## Known Limitations

1. **If false branch is unconnected.** When the AI output fails schema validation (`parse_success: false`), the workflow terminates silently. There is no error alert, no retry, and no record of the failure beyond n8n's execution log.

2. **Potential data reference mismatch in Notion node.** The Notion node references `$json.output.action_items.length` and `$json.output.meeting_summary`. However, the **JSON Output Parser** (the immediately preceding node in the flow) places the parsed data on `$json.ai_output`, not `$json.output`. The `$json.output` path corresponds to the raw LangChain agent output, which may still be available if n8n preserves it in the item. This should be verified in a live execution — if the Notion node is receiving data directly after the If node (which receives JSON Output Parser output), `$json.output` may be undefined and these properties would be empty.

3. **Environment variables not resolved.** The three `$env.*` references in **Metadata Extractor** are JavaScript template literals inside a code string, not n8n expression fields. They will be passed as literal strings. The Notion DB ID is hardcoded separately in the Notion node and works correctly, but `ASANA_PROJECT_ID` and `SLACK_MEETINGS_CHANNEL` will not resolve to actual values.

4. **Slack recipient is `USLACKBOT`.** The Slack node sends to Slackbot (the internal Slack test bot), not a real team channel or user. This must be updated to a real Slack user ID or channel ID before the workflow can deliver useful notifications.

5. **No retry logic on any node.** Any transient API failure (OpenRouter, Notion, Slack) will immediately fail the entire execution with no automatic recovery.

6. **Single-chunk AI processing.** Despite chunking logic in the Transcript Preprocessor, the AI Agent prompt passes `transcript_cleaned` (the full cleaned text) rather than iterating over `transcript_chunks`. For transcripts over 80,000 characters, the model may silently truncate if the OpenRouter model has a shorter context window.

7. **Asana and Gmail outputs not yet implemented.** The Metadata Extractor injects `asana_project_id` and `slack_channel_id`, and the original design specifies Asana task creation, Gmail recap, and Google Sheets archiving — but none of these nodes exist in the current workflow.

8. **No deduplication.** If the same `meeting_id` is submitted twice (e.g. a platform fires the webhook twice), the workflow will create duplicate Notion pages. No Redis or database check for deduplication is present.

---

## Assumptions

1. The caller sends the transcript in the `body` field of the POST request body (i.e. the transcript arrives at `$input.first().json.body.transcript`). This is standard for n8n webhook nodes but must match the platform's payload format.
2. Speaker names in the transcript follow the `Firstname Lastname:` pattern at the start of each line for the speaker detection regex to work.
3. The Notion database "Meeting Intelligence" has all the property names referenced in the Notion node's mapping exactly as specified (`Platform`, `Duration (min)`, `Participants`, `Action Items Count`, `Decision Count`, `Status`, `AI Summary`, `Meeting Date`).
4. The `arcee-ai/trinity-mini:free` model on OpenRouter follows the OpenAI-compatible chat completions format, as expected by the n8n OpenRouter LM node.
5. `parse_success: false` in the JSON Output Parser correctly reaches the If node's false branch — this assumes the JSON Output Parser does not throw an exception in the failure case (it returns an item with `parse_success: false` rather than throwing, which is correct per the code).
6. The Slack `property_*` field names from the Notion API response (`$json.property_ai_summary`, `$json.property_notion_url`, etc.) are resolved correctly by n8n's Notion node response mapping. If the Notion node does not automatically map these, the Slack message blocks will display `undefined`.

---

## Version History / Change Log

| Version | Date | Author | Changes |
|---|---|---|---|
| v1.0 | _[To be provided]_ | _[To be provided]_ | Initial build |

---

## References

- n8n Webhook node docs: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.webhook/
- n8n LangChain Agent docs: https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent/
- OpenRouter API docs: https://openrouter.ai/docs
- Notion API reference: https://developers.notion.com/reference/intro
- Slack Block Kit builder: https://app.slack.com/block-kit-builder

_[Additional references to be provided by workflow owner]_

---

## Maintenance Notes

_[To be provided by workflow owner]_

**Suggested maintenance items:**
- Monitor OpenRouter usage and costs if moving from `arcee-ai/trinity-mini:free` to a paid model.
- Periodically verify the Notion database property names haven't been renamed, which would silently break the Notion node's property mapping.
- Set up n8n execution log retention policy to retain failed executions for at least 30 days for debugging.
- Review and rotate the Notion integration token, Slack bot token, and OpenRouter API key on a regular cadence per the organisation's security policy.
