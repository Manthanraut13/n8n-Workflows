# Personal Career Intelligence & Skill Gap Agent — Workflow Documentation

| Field | Value |
|---|---|
| Workflow Name | Personal Career Intelligence & Skill Gap Agent |
| Workflow ID | `4ooXPtQ1ww9gBvpK` |
| Version | v1.0 (initial documentation) |
| Owner | _[To be provided by workflow owner]_ |
| Workflow Last Saved | _[Not present in export — check n8n instance]_ |
| Document Generated | 2026-07-08 |
| n8n Version ID | `07b26fc1-2c27-40ac-a106-ffe18bba7892` |
| Status | Inactive (not yet activated) |

---

## Purpose

This workflow automates weekly career intelligence gathering for an individual targeting the **AI Engineer** role in India. It scrapes live job listings from Indeed via Apify, uses an LLM to extract and cluster in-demand skills, compares those against the user's existing skill profile stored in Airtable, identifies skill gaps, and delivers a personalised weekly career digest — simultaneously to a Telegram chat and a Gmail inbox.

---

## Business Use Case

Replaces the manual, time-consuming process of scanning job boards and identifying skill gaps. Instead of spending hours weekly reviewing listings, the user receives a structured, AI-generated digest every Monday morning with the single most important skill to build that week, along with curated job links. _(Inferred from workflow design; confirm with workflow owner.)_

---

## Workflow Summary

The workflow is triggered on a weekly schedule (Mondays at 8:00 AM). It immediately fires an Apify actor to scrape up to 10 AI Engineer job listings from Indeed India and waits for the actor run to complete. Once the dataset is ready, all job records are retrieved and passed through a JavaScript Code node that consolidates descriptions into a single structured payload. An AI Agent (powered by OpenRouter's free LFM-2.5 model) analyses the consolidated job descriptions and returns a structured JSON of extracted skills ranked by market demand. A second Code node sanitises the AI's raw text output, stripping markdown fences and repairing minor JSON formatting errors, before the clean skills object is passed downstream. The workflow then fetches the user's current skill profile from a single Airtable record and runs a Gap Analysis Code node that diffs the market skills against the user's skills, sorts gaps by demand priority (high → medium → low), and finds relevant job listings that mention the top missing skill. A second AI Agent node (powered by OpenRouter's free Nemotron-Nano model) then composes a concise, emoji-rich Telegram-formatted career digest. Finally, the digest is delivered in parallel to a configured Telegram chat and a Gmail address.

---

## Workflow Diagram

See the attached canvas screenshot. The flow is **linear with a single fan-out at the terminal delivery step**:

```
Schedule Trigger
    → Job Listing Scraper (Apify)
        → Get dataset items (Apify)
            → Output parser (Code)
                → AI Agent — Skill Extraction
                  [sub: OpenRouter Chat Model]
                    → AI Agent output parser (Code)
                        → Fetch Skill Profile — Airtable
                            → Gap Analysis (Code)
                                → Compose Weekly Digest
                                  [sub: OpenRouter Chat Model1]
                                    ├→ Deliver Digest (Telegram)
                                    └→ Deliver Digest1 (Gmail)
```

There are no branches, loops, or error-handler sub-workflows configured.

---

## Trigger Details

| Property | Value |
|---|---|
| Node | `Schedule Trigger` |
| Type | `n8n-nodes-base.scheduleTrigger` v1.3 |
| Frequency | Weekly |
| Day | Monday (day index 1) |
| Hour | 08:00 |
| Minute | 00 |
| Timezone | Not explicitly set in workflow settings — defaults to n8n instance timezone. Pinned test data shows `America/New_York (UTC-04:00)`, suggesting the instance runs in US Eastern time. |

**Note:** If the n8n instance is hosted in India, the timezone should be set to `Asia/Kolkata` in n8n's global settings or the trigger node to ensure the digest arrives at a meaningful time locally.

---

## Execution Flow

1. **Schedule Trigger** fires every Monday at 08:00, emitting a timestamp payload and passing 1 item downstream.

2. **Job Listing Scraper** calls the Apify actor `MXLpngmVpE8WTESQr` (Indeed Jobs Scraper — "borderline/indeed-scraper") via the Apify n8n integration. It submits a custom JSON body requesting a search for **"AI Engineer"** in India (`country: "in"`), with `enableUniqueJobs: true` and `maxRows: 10`. The actor run executes asynchronously on Apify's infrastructure. The node waits for completion and returns the run metadata (including `storageIds.datasets.default`, the dataset ID for the scraped results).

3. **Get dataset items** takes the dataset ID from the run metadata (`$json.storageIds.datasets.default`) and retrieves all scraped job records from Apify's Dataset API. In the pinned test run this returned **10 items** — each containing the full job object (title, company, description text, location, URL, etc.).

4. **Output parser** (Code node, JavaScript) iterates over all input items and flattens them into a single output item with:
   - `totalJobs`: count of scraped jobs
   - `descriptions`: array of objects each with `title`, `company` (`companyName`), and `description` (`descriptionText`)
   
   This reduces 10 separate items to 1 consolidated item, making it suitable for a single LLM prompt.

5. **AI Agent — Skill Extraction** sends the consolidated job descriptions to the LLM with a detailed system prompt instructing it to extract technical skills, soft skills, and methodologies; count occurrences; and classify each skill as `high` / `medium` / `low` demand. The model is explicitly instructed to return **raw JSON only** (no markdown, no explanation). The model used is `liquid/lfm-2.5-1.2b-thinking:free` via OpenRouter.

6. **AI Agent output parser** (Code node, JavaScript) handles the LLM's raw text output stored in `$json.output`. It strips any markdown code fences (` ```json ` / ` ``` `), attempts `JSON.parse()`, and if that fails applies two common repair transforms (trailing comma removal, control character stripping) before a second parse attempt. On double failure it returns a safe empty-skills structure with a `parse_error` field. On success it returns the normalised fields: `skills`, `top_skills`, `technical_skills_count`, `soft_skills_count`, `highlighted_skills`.

7. **Fetch Skill Profile — Airtable** retrieves **a single record** (`recrm6gpl47geQPMH`) from a table named "Table 1" in the Airtable base "Smart Inbox Database" (`appmEJcPqEsQsVu7O`). The pinned data shows this record currently stores a single skill: Python (advanced, Programming Language, sourced from Work Experience). The operation is `get` (single record), not `list`.

8. **Gap Analysis** (Code node, JavaScript) is the workflow's core logic node. It:
   - Reads market skills from the `AI Agent output parser` node via `$('AI Agent output parser').first().json.skills`
   - Reads all Airtable records from `$(' Fetch Skill Profile — Airtable').all()` and maps skill names to lowercase
   - Filters out skills already present in the user's profile (case-insensitive)
   - Sorts remaining gaps by demand order (high → medium → low)
   - Selects the top-priority gap as `topSkill`
   - Scans the raw Apify dataset items for jobs mentioning `topSkill.name` and extracts up to 5 as `top5Jobs` (title, company, location, URL)
   - Outputs: `targetRole` (hardcoded `"AI Engineer"`), `gaps[]`, `topSkill`, `topSkillCategory` (array of categories from Airtable records), `top5Jobs[]`, `totalGaps`, `timestamp`

9. **Compose Weekly Digest** is a second AI Agent that receives the gap analysis output and composes the user-facing weekly digest. The prompt is a Telegram-optimised narrative template with sections for the top skill insight, recommended courses (if provided), and relevant job links. The model used is `nvidia/nemotron-nano-9b-v2:free` via OpenRouter (a different free model from the extraction step). Output is stored in `$json.output`.

10. **Deliver Digest** (Telegram) sends `$json.output` to a hardcoded Telegram Chat ID (`7350267365`). Runs in parallel with step 11.

11. **Deliver Digest1** (Gmail) sends `$json.output` as the message body to `storysnippents@gmail.com` with the subject line "Your Weekly Career Digest". Runs in parallel with step 10.

---

## Node List

| # | Node Name | Type | Role | Disabled |
|---|---|---|---|---|
| 1 | Schedule Trigger | Schedule Trigger | Weekly cron — initiates the workflow every Monday at 8 AM | No |
| 2 | Job Listing Scraper | Apify (community node) | Runs the Indeed Jobs Scraper actor on Apify | No |
| 3 | Get dataset items | Apify (community node) | Retrieves scraped job records from the Apify dataset | No |
| 4 | Output parser | Code (JavaScript) | Flattens 10 job items into a single structured payload for the LLM | No |
| 5 | AI Agent — Skill Extraction | LangChain Agent | Analyses job descriptions; extracts and classifies skills as JSON | No |
| 6 | OpenRouter Chat Model | LangChain LM (OpenRouter) | Sub-node — provides `liquid/lfm-2.5-1.2b-thinking:free` model to the extraction agent | No |
| 7 | AI Agent output parser | Code (JavaScript) | Sanitises and parses the LLM's raw text output into a clean skills object | No |
| 8 | Fetch Skill Profile — Airtable | Airtable | Fetches the user's current skill record from Airtable | No |
| 9 | Gap Analysis | Code (JavaScript) | Diffs market skills vs user skills; identifies and prioritises gaps; finds relevant jobs | No |
| 10 | Compose Weekly Digest | LangChain Agent | Drafts the Telegram-formatted weekly career digest from gap analysis data | No |
| 11 | OpenRouter Chat Model1 | LangChain LM (OpenRouter) | Sub-node — provides `nvidia/nemotron-nano-9b-v2:free` model to the digest agent | No |
| 12 | Deliver Digest | Telegram | Sends the digest to the configured Telegram chat | No |
| 13 | Deliver Digest1 | Gmail | Sends the digest to the configured Gmail address | No |

---

## Node-by-Node Description

### Schedule Trigger
Fires once a week on Monday at 08:00 (instance timezone). Outputs a single item with the current timestamp and date metadata. No credentials required.

---

### Job Listing Scraper
Uses the Apify n8n community node (`@apify/n8n-nodes-apify`) to call the Apify actor store and run the actor `MXLpngmVpE8WTESQr` (Indeed Jobs Scraper). The actor is configured via a `customBody` JSON string:

```json
{
  "query": "AI Engineer",
  "country": "in",
  "enableUniqueJobs": true,
  "includeSimilarJobs": false,
  "maxRows": 10
}
```

The node waits for the actor run to complete (synchronously within n8n's execution timeout). It outputs the run metadata object, most importantly `storageIds.datasets.default` which is consumed by the next node. Credentials: **Apify PromptLancer** (Apify API key).

---

### Get dataset items
Also uses the Apify n8n node, configured for the `Datasets` resource with operation `Get Items`. The dataset ID is resolved dynamically from the upstream run output:

```
datasetId = {{ $json.storageIds.datasets.default }}
```

Returns all scraped job records as individual n8n items (10 in the test run). Each item contains the full Indeed job object including `title`, `companyName`, `descriptionText`, `descriptionHtml`, `location`, `jobUrl`, `applyUrl`, `salary`, `rating`, `datePublished`, and more. Credentials: **Apify PromptLancer**.

---

### Output parser
JavaScript Code node. Aggregates all items from `Get dataset items` into one item to enable a single LLM call. Extracts only the fields needed for skill analysis:

```javascript
const jobs = $input.all();

return [{
  json: {
    totalJobs: jobs.length,
    descriptions: jobs.map(job => ({
      title: job.json.title,
      company: job.json.companyName,
      description: job.json.descriptionText
    }))
  }
}];
```

Output: 1 item with `totalJobs` (integer) and `descriptions` (array of `{title, company, description}`).

---

### AI Agent — Skill Extraction
LangChain Agent node (v3.1). Receives the consolidated job payload and sends it to the LLM with:

**System prompt:** Instructs the model to classify skills into Technical Skills (languages, frameworks, cloud, DevOps, AI/ML tools), Professional Skills (communication, leadership), and Methodologies (Agile, MLOps, CI/CD). Defines demand thresholds: `high` ≥ 50% of jobs, `medium` 20–49%, `low` < 20%. Mandates raw JSON output only — no markdown, no prose.

**User prompt (expression):**
```
Analyze the following {{ $json.totalJobs }} job listings and identify the most in-demand skills.

Job Listings:
{{ JSON.stringify($json.descriptions, null, 2) }}
```

**Expected output schema:**
```json
{
  "skills": [{"name": "Python", "category": "Programming Language", "frequency": "high", "occurrences": 8}],
  "top_skills": ["Python"],
  "technical_skills_count": 1,
  "soft_skills_count": 0,
  "highlighted_skills": ["Python"]
}
```

The agent's output is available as `$json.output` (raw text string) for the next node.

---

### OpenRouter Chat Model
LangChain sub-node attached to `AI Agent — Skill Extraction` via the `ai_languageModel` connection type. Uses model `liquid/lfm-2.5-1.2b-thinking:free` — a free reasoning model available via OpenRouter. Credentials: **OpenRouter account for AI Journal**.

---

### AI Agent output parser
JavaScript Code node that handles the inherently unpredictable output from the LLM. The agent node stores the model's text response in `$json.output`. This node:

1. Strips ` ```json ` and ` ``` ` markdown fences (regex replace)
2. Trims whitespace
3. Attempts `JSON.parse()`
4. On failure, applies repair transforms (removes trailing commas in objects/arrays, removes control characters) and retries parse
5. On double failure, returns a safe fallback object with empty arrays and a `parse_error` message, allowing downstream nodes to receive valid (if empty) data rather than crashing

Output fields: `skills[]`, `top_skills[]`, `technical_skills_count`, `soft_skills_count`, `highlighted_skills[]`, and optionally `parse_error`.

---

### Fetch Skill Profile — Airtable
Airtable node (v2.2) using the `get` operation to retrieve a **single specific record** by hardcoded record ID `recrm6gpl47geQPMH` from:

- **Base:** Smart Inbox Database (`appmEJcPqEsQsVu7O`)
- **Table:** Table 1 (`tblDIe008D0MPJcrl`)

The pinned test data shows the record contains:

```json
{
  "Skill Name": "Python",
  "Category": "Programming Language",
  "Level": "advanced",
  "Source": "Work Experience",
  "Last Updated": "2026-06-11"
}
```

**Important limitation:** Currently configured as a single-record `get`, meaning only one skill is fetched per run. The Gap Analysis node calls `.all()` on this node's output, which will return only 1 item unless the Airtable node is reconfigured to list all records. See Known Limitations.

Credentials: **Airtable Personal Access Token account**.

---

### Gap Analysis
JavaScript Code node — the analytical core of the workflow. References two upstream nodes by name using n8n's `$()` syntax:

```javascript
const marketSkills = $('AI Agent output parser').first().json.skills || [];
const skillRecords  = $(' Fetch Skill Profile — Airtable').all();
```

**Logic:**
- Maps `skillRecords` to lowercase skill names
- Filters `marketSkills` to find those not in the user's profile
- Sorts gaps by demand priority: `high (0)` → `medium (1)` → `low (2)`
- Takes `gaps[0]` as `topSkill`
- Retrieves raw Apify items from `$('Get dataset items').all()` and filters for jobs mentioning `topSkill.name` (case-insensitive substring match on `descriptionText`)
- Takes up to 5 matching jobs, mapping to `{title, company, location.fullAddress, url: jobUrl}`

**Output payload:**
```json
{
  "targetRole": "AI Engineer",
  "gaps": [...],
  "topSkill": {"name": "...", "category": "...", "frequency": "high", "occurrences": N},
  "topSkillCategory": [...],
  "top5Jobs": [{"title": "...", "company": "...", "location": "...", "url": "..."}],
  "totalGaps": N,
  "timestamp": "ISO-8601"
}
```

Note: `targetRole` is hardcoded as `"AI Engineer"` inside the code.

---

### Compose Weekly Digest
Second LangChain Agent node (v3.1). Receives the gap analysis payload and composes the user-facing message.

**System prompt:** Instructs the model to write a friendly, Telegram-optimised digest under 400 words, using emojis, with this structure:
- `📊 Your Weekly Career Digest` header
- 1-line summary
- `🎯 This week's skill to build` (≤50 words)
- `📚 Recommended courses` (bulleted links — note: no course API is connected; this section will be empty or the model will fabricate courses unless course data is added upstream)
- `💼 Relevant jobs` (3–5 job links)
- Motivational closing line

**User prompt (expression):**
```
Target Role: {{ $json.targetRole }}
Top Skill Gap: {{ $json.topSkill.name }}
Skill Category: {{ $json.gaps[2].name }}   ← Note: hardcoded index [2], not [0]
Market Demand: {{ $json.topSkill.frequency }}
Total Skill Gaps: {{ $json.totalGaps }}
Relevant Jobs: {{ JSON.stringify($json.top5Jobs, null, 2) }}
```

The model's response (plain text digest) is available as `$json.output`.

---

### OpenRouter Chat Model1
LangChain sub-node attached to `Compose Weekly Digest` via `ai_languageModel`. Uses model `nvidia/nemotron-nano-9b-v2:free` — a different free model from the extraction step, better suited for natural language generation. Credentials: **OpenRouter account for AI Journal** (same credential as the other OpenRouter node).

---

### Deliver Digest
Telegram node (v1.2). Sends the digest to a hardcoded Chat ID `7350267365`.

```
Operation: sendMessage
Chat ID: 7350267365
Text: {{ $json.output }}
```

Credentials: **Telegram account for AI Journal**.

---

### Deliver Digest1
Gmail node (v2.2). Sends the digest as a plain-text email body.

```
To: storysnippents@gmail.com
Subject: Your Weekly Career Digest
Message: {{ $json.output }}
```

Credentials: **Gmail account**. Both `Deliver Digest` and `Deliver Digest1` are connected to `Compose Weekly Digest`'s single output and execute in parallel (fan-out pattern).

---

## Node Configuration

### Apify Actor Parameters (Job Listing Scraper)

```json
{
  "query": "AI Engineer",
  "country": "in",
  "enableUniqueJobs": true,
  "includeSimilarJobs": false,
  "maxRows": 10
}
```

Actor ID: `MXLpngmVpE8WTESQr` (`borderline/indeed-scraper` from Apify Store).

### Gap Analysis — Key Code

```javascript
// Diff logic
const gaps = marketSkills
  .filter(skill => !mySkills.includes(skill.name.toLowerCase()))
  .sort((a, b) => {
    const order = { high: 0, medium: 1, low: 2 };
    return (order[a.frequency] ?? 999) - (order[b.frequency] ?? 999);
  });
```

### Compose Weekly Digest — Skill Category Bug

```javascript
// In the prompt:
Skill Category: {{ $json.gaps[2].name }}
// Should likely be:
Skill Category: {{ $json.topSkill.category }}
```

`gaps[2]` references the third gap (index 2), not the top skill's category. This is a likely bug — see Known Limitations.

---

## Variables & Expressions

| Location | Expression | What it computes |
|---|---|---|
| Get dataset items | `{{ $json.storageIds.datasets.default }}` | Apify dataset ID from the completed actor run |
| AI Agent — Skill Extraction (prompt) | `{{ $json.totalJobs }}` | Count of scraped job listings |
| AI Agent — Skill Extraction (prompt) | `{{ JSON.stringify($json.descriptions, null, 2) }}` | Full serialised array of job descriptions |
| Compose Weekly Digest (prompt) | `{{ $json.targetRole }}` | Hardcoded "AI Engineer" from Gap Analysis output |
| Compose Weekly Digest (prompt) | `{{ $json.topSkill.name }}` | Name of the highest-priority skill gap |
| Compose Weekly Digest (prompt) | `{{ $json.gaps[2].name }}` | Name of the 3rd skill gap (likely a bug — see Known Limitations) |
| Compose Weekly Digest (prompt) | `{{ $json.topSkill.frequency }}` | Demand level of top skill (high/medium/low) |
| Compose Weekly Digest (prompt) | `{{ $json.totalGaps }}` | Total number of gaps identified |
| Compose Weekly Digest (prompt) | `{{ JSON.stringify($json.top5Jobs, null, 2) }}` | Array of up to 5 relevant job listings |
| Deliver Digest | `{{ $json.output }}` | AI-generated digest text |
| Deliver Digest1 | `{{ $json.output }}` | AI-generated digest text |

---

## Credentials Used

| Node | Credential Name | Credential Type | Service |
|---|---|---|---|
| Job Listing Scraper | Apify PromptLancer | Apify API | Apify platform |
| Get dataset items | Apify PromptLancer | Apify API | Apify platform |
| OpenRouter Chat Model | OpenRouter account for AI Journal | OpenRouter API | OpenRouter (LLM proxy) |
| OpenRouter Chat Model1 | OpenRouter account for AI Journal | OpenRouter API | OpenRouter (LLM proxy) |
| Fetch Skill Profile — Airtable | Airtable Personal Access Token account | Airtable Token API | Airtable |
| Deliver Digest | Telegram account for AI Journal | Telegram API | Telegram Bot API |
| Deliver Digest1 | Gmail account | Gmail OAuth2 | Google Gmail |

---

## Input Data

| Source | Shape |
|---|---|
| Schedule Trigger | `{ timestamp, "Readable date", "Readable time", "Day of week", Year, Month, "Day of month", Hour, Minute, Second, Timezone }` |
| Apify Dataset (per job) | Full Indeed job object: `jobKey`, `title`, `jobType[]`, `descriptionText`, `descriptionHtml`, `companyName`, `location.fullAddress`, `salary`, `rating`, `jobUrl`, `applyUrl`, `datePublished`, `isRemote`, and more |
| Airtable Skill Record | `{ id, createdTime, fields: { "Skill Name", Category, Level, Source, "Last Updated" } }` |

---

## Output Data

| Terminal Node | Output |
|---|---|
| Deliver Digest (Telegram) | Text message sent to chat `7350267365` containing the AI-composed weekly digest |
| Deliver Digest1 (Gmail) | Email sent to `storysnippents@gmail.com` with subject "Your Weekly Career Digest" and the digest as the message body |

---

## Data Transformation

1. **Output parser** — 10 individual Apify job items → 1 consolidated object with a `descriptions[]` array (discards all fields except title, company, description text)
2. **AI Agent — Skill Extraction** — Free-form job description text → structured skill taxonomy JSON
3. **AI Agent output parser** — Raw LLM text response → parsed and validated JavaScript object
4. **Gap Analysis** — Market skills array + user skills record → prioritised gap list + filtered job matches
5. **Compose Weekly Digest** — Gap analysis object → human-readable Telegram-formatted plain text narrative

---

## API Integrations / External Services

| Service | Node | Method/Operation | Auth Type | Purpose |
|---|---|---|---|---|
| Apify Platform | Job Listing Scraper | Run actor (via Apify n8n node) | Apify API Key | Execute Indeed scraper |
| Apify Platform | Get dataset items | Get dataset items (via Apify n8n node) | Apify API Key | Retrieve scraped job records |
| OpenRouter API | OpenRouter Chat Model | LLM inference | API Key (Bearer) | Skill extraction via `liquid/lfm-2.5-1.2b-thinking:free` |
| OpenRouter API | OpenRouter Chat Model1 | LLM inference | API Key (Bearer) | Digest composition via `nvidia/nemotron-nano-9b-v2:free` |
| Airtable REST API | Fetch Skill Profile — Airtable | Get record | Personal Access Token | Fetch user skill profile |
| Telegram Bot API | Deliver Digest | Send message | Bot Token | Deliver weekly digest via Telegram |
| Gmail API (Google) | Deliver Digest1 | Send email | OAuth2 | Deliver weekly digest via email |

---

## AI/LLM Integration

### AI Agent — Skill Extraction

| Property | Value |
|---|---|
| Node type | `@n8n/n8n-nodes-langchain.agent` v3.1 |
| Model | `liquid/lfm-2.5-1.2b-thinking:free` via OpenRouter |
| Prompt style | Defined prompt (`promptType: "define"`) — custom user + system messages |
| Output format | Raw JSON (enforced via system prompt instructions) |
| Output field | `$json.output` (raw string) |
| Tools attached | None |
| Memory attached | None |

The system prompt is extensive (~450 words) and includes: skill taxonomy definitions, demand classification thresholds (high ≥ 50%, medium 20–49%, low < 20%), and strict output format rules (no markdown, valid `JSON.parse()`-compatible output, exact schema specified).

### Compose Weekly Digest

| Property | Value |
|---|---|
| Node type | `@n8n/n8n-nodes-langchain.agent` v3.1 |
| Model | `nvidia/nemotron-nano-9b-v2:free` via OpenRouter |
| Prompt style | Defined prompt — gap analysis data injected via expressions |
| Output format | Plain text (Telegram-formatted, emoji-rich, under 400 words) |
| Output field | `$json.output` |
| Tools attached | None |
| Memory attached | None |

The system prompt specifies a 5-section digest structure with section headers using emojis, a 400-word cap, and a friendly, encouraging tone. No course recommendation data is injected (no course API is connected), so the "Recommended courses" section of the digest will either be omitted by the model or populated with hallucinated links.

---

## Error Handling

- **No error workflow configured** — the workflow settings do not specify an `errorWorkflow`. If any node fails, the execution will halt and the failure will only appear in the n8n execution log.
- **No `continueOnFail`** set on any node — a failure in any step (Apify actor error, LLM timeout, Airtable 404, Telegram rate limit, etc.) will stop the entire execution.
- **One exception:** The `AI Agent output parser` Code node implements its own internal fault tolerance — on JSON parse failure it catches the error and returns a safe empty-skills object, allowing the workflow to continue with degraded (empty) skill data rather than crashing at this step.
- **No retry logic** is configured on the HTTP-based nodes (Apify, LLM calls, Airtable, Telegram, Gmail).

---

## Retry Strategy

No retry logic is configured on any node. The Apify actor run itself has a built-in timeout of 3600 seconds (configured in the actor options), but this is an actor-side setting, not an n8n retry. If the actor exceeds that, the n8n node will receive a failed status and propagate the error.

---

## Failure Notifications

None configured. There is no error trigger workflow or dedicated notification node for failures. Failures are only visible in the n8n execution history panel.

---

## Logging

No explicit logging to an external system. All execution data is stored in n8n's internal execution log (subject to the instance's execution log retention settings). The `Gap Analysis` node does emit a `timestamp` field in its output for traceability within a single run.

---

## Execution Order

The workflow uses `executionOrder: "v1"` (connection-based execution order, the modern n8n default). This ensures nodes execute in the order defined by their connections, and the fan-out at `Compose Weekly Digest` → `[Deliver Digest, Deliver Digest1]` will execute both delivery nodes, though the exact interleaving order of the two parallel terminal nodes is not guaranteed.

---

## Scheduling Details

| Property | Value |
|---|---|
| Schedule type | Weekly |
| Day(s) | Monday |
| Time | 08:00 |
| Repeat interval | Every 1 week |
| Instance timezone | Not explicitly set in workflow settings — inherited from n8n instance global timezone |

**Recommendation:** Set the workflow-level timezone in n8n's Schedule Trigger node or global settings to `Asia/Kolkata` if the intended audience is in India, to ensure the digest arrives on Monday morning IST.

---

## Dependencies

| Dependency | Type | Notes |
|---|---|---|
| Apify Platform | External SaaS | Must be reachable; actor `MXLpngmVpE8WTESQr` must remain in Apify Store and operational |
| Indeed (via Apify) | External website | Apify actor depends on Indeed's page structure; changes to Indeed's layout can break scraping |
| OpenRouter API | External LLM proxy | Two free models must remain available on OpenRouter; model availability may change |
| Airtable | External SaaS | Base `appmEJcPqEsQsVu7O` and record `recrm6gpl47geQPMH` must exist and be accessible |
| Telegram Bot API | External API | Bot must be active and not blocked |
| Gmail (Google OAuth2) | External API | OAuth2 token must remain valid (tokens expire; n8n auto-refreshes if the refresh token is valid) |

No sub-workflows are called.

---

## Security Considerations

- **No secrets are exposed in the workflow JSON** — all credentials are referenced by n8n credential IDs only. Actual API keys are stored in n8n's encrypted credential store.
- **Hardcoded PII:** The Gmail recipient address (`storysnippents@gmail.com`) and Telegram Chat ID (`7350267365`) are hardcoded in node parameters. Anyone with access to this workflow JSON or the n8n instance can see these values.
- **Hardcoded target role:** `"AI Engineer"` is hardcoded inside the `Gap Analysis` Code node — changing the target role requires editing the code.
- **Airtable record ID is hardcoded:** The workflow fetches one specific Airtable record by ID (`recrm6gpl47geQPMH`). If the record is deleted or the base is recreated, the node will return a 404 error.
- **No webhook authentication:** Not applicable — this workflow has no webhook trigger.
- **LLM output injection risk:** The Compose Weekly Digest agent receives job description data via the gap analysis payload. Malicious content embedded in scraped job listings could theoretically influence the LLM's output (prompt injection). The risk is low for a personal digest but worth noting.

_[For a complete security review, consult the workflow owner.]_

---

## Performance Considerations

- **Apify actor runtime:** In the pinned test run, the Indeed scraper completed in ~20 seconds (`runTimeSecs: 19.879`). This is the dominant latency in the workflow.
- **LLM calls:** Two sequential LLM calls add variable latency depending on OpenRouter's queue. Free tier models may have higher latency than paid models.
- **Batch size:** Fixed at 10 job listings (`maxRows: 10`). Increasing this improves skill coverage but increases the LLM prompt size and cost/latency.
- **Airtable:** Single-record fetch is fast; however, as the skill profile grows, the node must be reconfigured to list all records (see Known Limitations).
- **Total estimated runtime:** 30–90 seconds per execution depending on LLM response times.

---

## Execution Time

_[To be provided by workflow owner — requires execution data from n8n's execution log after live runs.]_

---

## Resource Usage

- **Apify:** Each run consumes Apify compute units. The test run used `0.0028` compute units and charged for 10 `apify-default-dataset-item` events at $0.005/event = **$0.05 per run** (~$0.20/month at weekly cadence). _Note: the `accountedChargedEventCounts` shows 0 in the test run, suggesting the credits were applied from the free tier._
- **OpenRouter:** Both models are configured as `:free` models, incurring no direct API cost.
- **Airtable:** Well within the free plan limits (1 record read per week).
- **Telegram / Gmail:** No cost.

---

## Testing & Validation

_[To be provided by workflow owner.]_

Test data is pinned on three nodes:
- **Schedule Trigger** — Pinned timestamp from 2026-06-11 (Thursday) used to test without waiting for the next Monday
- **Get dataset items** — Pinned with 10 real Indeed job records scraped during a test run (companies include Ecolab, Kensium/OmnifiCX, Infosys, TCS, Rayvector, Litmus7, InfoBeans, Zensar, Commdel)
- **Fetch Skill Profile — Airtable** — Pinned with a single skill record: Python / advanced / Programming Language

---

## Troubleshooting

| Symptom | Likely Cause | Resolution |
|---|---|---|
| Workflow does not fire on Monday | n8n instance timezone mismatch; n8n container may be sleeping (Render free tier) | Set timezone to correct region in n8n global settings; use UptimeRobot to keep Render service awake |
| `Get dataset items` returns 0 items | Apify actor failed or Indeed blocked the scraper | Check Apify console for actor run errors; verify the actor is still available in the Apify Store |
| `AI Agent output parser` returns `parse_error` | LLM returned non-JSON or malformed response despite strict prompt | Switch to a more instruction-following model on OpenRouter; add retry logic; inspect `$json.output` in execution log for the raw response |
| Gap Analysis returns 0 gaps | User profile already contains all market skills, OR Airtable returned no records | Check the Airtable node — if configured as `get` (single record), ensure the hardcoded record ID still exists; switch to `list` to fetch all skills |
| Digest contains no course recommendations | No course API is connected | Expected behaviour — the "Recommended courses" section will be absent or hallucinated; see Known Limitations |
| Telegram delivery fails | Bot token expired, bot was blocked by user, or Chat ID is wrong | Re-validate the Telegram credential in n8n; ensure the bot has not been stopped |
| Gmail fails with OAuth error | OAuth2 refresh token expired | Re-authenticate the Gmail credential in n8n (Credentials → edit → reconnect) |
| `$json.gaps[2].name` expression in Compose Weekly Digest fails | Fewer than 3 gaps were identified | The expression will throw if `gaps` has fewer than 3 elements — wrap in a safe expression or fix the bug (see Known Limitations) |

---

## Known Limitations

1. **Single Airtable record (most critical):** The `Fetch Skill Profile — Airtable` node is configured as a `get` (single record) operation with a hardcoded record ID. It will only ever fetch the one Python skill entry. To represent a multi-skill profile, the node must be changed to `list` (List Records operation) so it returns all rows in the table. Without this change, nearly all market skills will always appear as gaps regardless of the user's real profile.

2. **`gaps[2].name` bug in Compose Weekly Digest prompt:** The prompt expression `{{ $json.gaps[2].name }}` references the third skill in the gaps array and labels it "Skill Category" in the prompt — which appears to be a mistake. It likely should be `{{ $json.topSkill.category }}`. If fewer than 3 gaps exist, this expression will throw a runtime error.

3. **No course recommendations connected:** The digest prompt asks the LLM to include a "Recommended courses" section, but no course API (Udemy, Coursera, YouTube) is connected to the workflow. The LLM will either skip the section or generate plausible-sounding but potentially fabricated course names and URLs.

4. **Target role hardcoded in Gap Analysis:** `targetRole: 'AI Engineer'` is hardcoded inside the Code node. Changing the target role requires editing the code directly. Consider using an n8n variable or Airtable configuration record instead.

5. **No error workflow:** If any step fails (Apify actor timeout, LLM error, Airtable 404, network issue), the workflow silently fails with no notification to the user. An error trigger workflow should be added.

6. **OpenRouter free model reliability:** Both LLM models used are free-tier models that may be rate-limited, deprecated, or removed from OpenRouter without notice. The workflow has no fallback model configured.

7. **maxRows: 10 is a small sample:** 10 job listings is a limited sample for statistically reliable skill frequency analysis. Skills appearing in 5+ of 10 listings are classified as "high demand" — this threshold is sensitive to small sample noise.

8. **No deduplication of weekly results:** The workflow does not track which skills were surfaced in previous weeks. The user may receive the same "top skill to build" recommendation for multiple consecutive weeks if their Airtable profile is not updated after completing a course.

---

## Assumptions

1. The n8n instance timezone is or will be configured as `Asia/Kolkata` for the 8:00 AM Monday trigger to fire at the intended local time in India.
2. The Airtable base "Smart Inbox Database" is the intended location for the career skill profile, despite the base name suggesting a different primary purpose (inbox management). This may have been repurposed or is a shared base.
3. The `Fetch Skill Profile — Airtable` node is intended to eventually be changed from `get` (single record) to `list` (all records), as the Gap Analysis code already calls `.all()` on its output — indicating the developer anticipated a multi-record skill list.
4. The two different OpenRouter models (LFM-2.5 for extraction, Nemotron-Nano for composition) were chosen deliberately to use the best available free model for each task type (reasoning vs. generation).
5. The `targetRole` value of `"AI Engineer"` in the Gap Analysis code reflects the workflow owner's personal career target, not a configurable parameter.
6. Pinned test data on `Schedule Trigger` (showing Thursday, June 11 2026) indicates the workflow was tested mid-week using pinned data to bypass the schedule without waiting for Monday.

---

## Version History / Change Log

_[To be provided by workflow owner.]_

---

## References

- Apify Indeed Scraper Actor: https://console.apify.com/actors/MXLpngmVpE8WTESQr/input
- Apify n8n Node (community): https://www.npmjs.com/package/@apify/n8n-nodes-apify
- OpenRouter model list: https://openrouter.ai/models
- n8n LangChain Agent docs: https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent/
- Airtable API docs: https://airtable.com/developers/web/api/introduction

---

## Maintenance Notes

_[To be provided by workflow owner.]_

**Suggested maintenance tasks:**
- **Monthly:** Check that both OpenRouter free models (`liquid/lfm-2.5-1.2b-thinking:free`, `nvidia/nemotron-nano-9b-v2:free`) are still available on OpenRouter's model list.
- **After each weekly digest:** Update the Airtable skill profile (add newly learned skills, update proficiency levels) to keep the gap analysis accurate. Without this, the same gaps will surface every week.
- **Quarterly:** Review the Apify actor version (`0.0.170` in the test run) and update if a newer version is available or if scraping quality degrades.
- **Immediately:** Fix the `gaps[2].name` bug and reconfigure the Airtable node to `list` all records — both are blocking the workflow from functioning correctly at scale.
