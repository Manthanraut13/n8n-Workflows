# AI Feedback Synthesis & Roadmap Prioritiser — Workflow Documentation

| Field | Value |
|---|---|
| Workflow Name | AI Feedback Synthesis & Roadmap Prioritiser |
| Workflow ID | oKC8N4zCWOloHhNE |
| Version | v1.0 (initial documentation) — internal revision `versionId: 4fa9506b-70c6-4900-ac56-bed7c47d5430` (not a human-facing version number) |
| Owner | _[To be provided by workflow owner]_ |
| Last Updated | Workflow: `updatedAt` not present in export; Document generated: 2026-07-18 |
| Active | `false` (workflow is currently deactivated in n8n) |

## Purpose

This workflow automates a weekly customer-feedback synthesis and product-roadmap prioritisation cycle. It pulls raw feedback from four Google Sheets sources, normalises and deduplicates it, sends it to an LLM (via OpenRouter) to cluster it into themes and score them, ranks the top 10 themes by a composite priority formula, and distributes the result as a Slack digest while archiving the ranked themes back to a Google Sheet.

## Business Use Case

Automates the manual, recurring work of a product/growth team collating NPS scores, support tickets, sales call notes, and community posts into a single prioritised view of "what to build next." Reduces time spent manually reading and triaging feedback, and creates a searchable historical record of ranked themes in Google Sheets. _(Inferred from workflow structure and node naming — not stated explicitly in the JSON; confirm with the workflow owner.)_

## Workflow Summary

Every Monday at 07:00 UTC, the **Weekly Trigger** fires four parallel Google Sheets reads (NPS, support tickets, call notes, community posts). Each source is normalised by its own Code node into a common `{id, source, text, score, date, user_plan}` shape, then the four branches are merged together in two sequential Merge steps. The combined list is deduplicated and capped at 500 items (with negative-sentiment "pain" items prioritised for retention), then checked by an IF node: if there's no feedback, a "no data" Slack message is sent and the run ends; otherwise the capped item set is sent to an AI Agent (backed by OpenRouter, model `nvidia/nemotron-nano-12b-v2-vl:free`) which clusters the feedback into 8–12 themes and scores each one. The AI's raw JSON response is parsed/validated, a composite priority score is computed and the themes ranked, and the top 10 are simultaneously (a) formatted into a Slack digest message and posted to Slack, and (b) split into individual rows and appended to a "history" tab in Google Sheets.

## Workflow Diagram

See the attached canvas screenshot. Shape: **fan-out / merge / branch**. Four parallel ingestion-and-normalisation branches (NPS, Support, Call Notes, Community) converge through two sequential Merge nodes into a single deduplication step, which feeds an IF node. The IF node's false branch is a short dead-end (Slack notification only); its true branch runs through the AI Agent → parser → scorer, then fans out again into two parallel terminal branches (Slack digest post, and Google Sheets history append via a Split Out/loop step).

High-level flow:

```
Weekly Trigger (Cron, Mon 07:00 UTC)
 ├─ Fetch NPS ────────────┐
 ├─ Fetch Support Issues ─┤ (each → its own Normalise Code node)
 ├─ Fetch Call Notes ─────┤
 └─ Fetch Community Posts ┘
        ↓ (2x Merge, append-style)
    Merge NPS+Support → Merge +Call Notes → Merge All Sources
        ↓
    Deduplicate + Cap (Code)
        ↓
    IF: Has Feedback?
        ├─ TRUE  → AI Agent (OpenRouter LLM) → Parse & Validate AI Response
        │            → Impact Scorer & Ranker
        │                ├─ Format Slack Message → Send Slack Digest
        │                └─ Split Top10 for Sheets Loop → Write to Sheets History
        └─ FALSE → Slack: No Data This Week (end)
```

## Trigger Details

- **Type**: Schedule Trigger (`n8n-nodes-base.scheduleTrigger`)
- **Node name**: `Weekly Trigger`
- **Schedule**: Cron expression `0 7 * * 1` — fires every **Monday at 07:00** (per the node's own note: UTC; no explicit `settings.timezone` override was found in the workflow, so the instance-default/UTC timezone applies).
- **Note on node**: "Fires every Monday 07:00 UTC. All 5 branches run in parallel from here." (author's own annotation — note the workflow actually has 4 data-source branches plus the trigger itself, not 5 independent branches downstream).
- Single entry point; no other trigger nodes exist in this workflow.

## Execution Flow

1. **Weekly Trigger** fires on schedule → triggers four parallel Google Sheets reads simultaneously.
2. **Fetch NPS (Google Sheets)** reads the `nps_feedback` document's `nps_responses`-tagged tab (`gid=0`) → **Normalise NPS** (Code) filters to the last 7 days, builds a common record shape, computes an NPS-derived score.
3. **Fetch Support Issues (Google Sheets)** reads the `support_tickets` sheet → **Normalise Support Issues** (Code) filters to last 7 days, maps `priority` (high/medium/low) to a numeric severity and a derived score.
4. **Fetch Call Notes (Google Sheets)** reads the `call_notes` sheet → **Normalise Call Notes** (Code) filters to last 7 days, concatenates pain points + feature requests, tags the record with a fixed `weight: 4` (high signal).
5. **Fetch Community Posts (Google Sheets)** reads the `community_feedback` sheet → **Normalise Community** (Code) filters to last 7 days, applies a simple keyword-based sentiment hint, tags with `weight: 2` (medium signal).
6. **Merge NPS + Support**: combines outputs of steps 2 and 3 (append mode, two inputs).
7. **Merge + Call Notes**: combines the merge from step 6 with the call-notes branch (step 4).
8. **Merge All Sources**: combines the merge from step 7 with the community branch (step 5) — this is the point where all four sources are unified into one list.
9. **Deduplicate + Cap** (Code): removes duplicate items (by `source` + first 80 characters of `text`), retains all "pain" items (`score <= 4`) preferentially, caps the total list at 500 items, and outputs a single item containing the capped `items[]` array plus a `meta` object (raw/dedup/sent counts, week label, per-source counts).
10. **IF: Has Feedback?**: checks whether `meta.sent_to_ai > 0`.
    - **TRUE** → continues to the AI Agent.
    - **FALSE** → **Slack: No Data This Week** posts a "no feedback this week" message to Slack and the run ends (no further nodes downstream).
11. **AI Agent** (LangChain Agent node, using **OpenRouter Chat Model** as its LLM): receives the capped items + metadata, clusters them into 8–12 themes, and returns a JSON object with `themes[]` and a `summary`.
12. **Parse & Validate AI Response** (Code): robustly extracts the JSON from the LLM's raw output (handles OpenRouter's `choices[0].message.content` shape, a plain `.output` field, markdown-fenced JSON, or a bare JSON object embedded in text), validates that a `themes` array exists, sanitises/clamps each theme's numeric fields, and re-attaches `meta` from the **Deduplicate + Cap** node (via `$('Deduplicate + Cap')` node-reference).
13. **Impact Scorer & Ranker** (Code): computes a composite priority score per theme (`0.40 × freq_ratio + 0.35 × impact_norm + 0.25 × urgency`), sorts themes by composite score (with impact and frequency as tiebreakers), and takes the top 10, assigning a `final_rank`.
14. From here the flow fans out into two parallel branches:
    - **14a. Format Slack Message** (Code) builds a formatted Slack message string (ranked emoji list, per-theme score/sentiment/frequency/source breakdown, quoted sample feedback, AI summary, source funnel counts) → **Send Slack Digest** posts it to Slack.
    - **14b. Split Top10 for Sheets Loop** (Split Out node) explodes the `top10` array into one item per theme → **Write to Sheets History** appends each theme as a row to a Google Sheets tab (`Sheet5`, gid `120873223`).

## Node List

| # | Node Name | Type | Purpose | Disabled |
|---|---|---|---|---|
| 1 | Weekly Trigger | Schedule Trigger | Starts the workflow every Monday 07:00 | No |
| 2 | Fetch NPS (Google Sheets) | Google Sheets (read) | Reads NPS survey responses | No |
| 3 | Fetch Support Issues (Google Sheets) | Google Sheets (read) | Reads support ticket log | No |
| 4 | Fetch Call Notes (Google Sheets) | Google Sheets (read) | Reads sales/customer call notes | No |
| 5 | Fetch Community Posts (Google Sheets) | Google Sheets (read) | Reads community post log | No |
| 6 | Normalise NPS | Code | Normalises + filters NPS rows to a week window | No |
| 7 | Normalise Support Issues | Code | Normalises + filters support rows | No |
| 8 | Normalise Call Notes | Code | Normalises + filters call-note rows | No |
| 9 | Normalise Community | Code | Normalises + filters community rows, adds sentiment hint | No |
| 10 | Merge NPS + Support | Merge | Combines NPS + Support branches | No |
| 11 | Merge + Call Notes | Merge | Combines above with Call Notes branch | No |
| 12 | Merge All Sources | Merge | Combines above with Community branch | No |
| 13 | Deduplicate + Cap | Code | Dedupes, stratified-caps at 500 items, builds meta | No |
| 14 | IF: Has Feedback? | IF | Branches on whether any feedback exists | No |
| 15 | Slack: No Data This Week | Slack (post message) | Notifies "no data" and ends the run | No |
| 16 | AI Agent | LangChain Agent | Clusters feedback into themes via LLM | No |
| 17 | OpenRouter Chat Model | LangChain LM (OpenRouter) | LLM backing the AI Agent | No |
| 18 | Parse & Validate AI Response | Code | Extracts/validates/sanitises the AI's JSON output | No |
| 19 | Impact Scorer & Ranker | Code | Computes composite score, ranks, takes top 10 | No |
| 20 | Format Slack Message | Code | Builds the formatted digest message text | No |
| 21 | Send Slack Digest | Slack (post message) | Posts the digest to Slack | No |
| 22 | Split Top10 for Sheets Loop | Split Out | Explodes top10 array into individual items | No |
| 23 | Write to Sheets History | Google Sheets (append) | Archives ranked themes to history tab | No |

## Node-by-Node Description

### Fetch NPS (Google Sheets) / Fetch Support Issues / Fetch Call Notes / Fetch Community Posts
All four read from the same spreadsheet (`AI Customer Feedback Synthesis & Product Roadmap Prioritiser`, ID `1NCJA8xuZKRIaF4ahOLnyXYqYAbt4jLXVmyaUUMKBb9c`), each pointed at a different tab (`nps_feedback`/gid=0, `support_tickets`, `call_notes`, `community_feedback`). No sheet-level filters are configured (`combineFilters: AND` with no conditions — i.e. all rows are fetched; filtering to "this week" happens downstream in the Normalise Code nodes). Per the nodes' own notes, sheets are populated via Google Forms or manual paste; the Support node explicitly notes it **replaces Freshdesk** and the Call Notes node explicitly notes it **replaces Gmail-based sales call notes**.

### Normalise NPS
Filters items to the last 7 days by `timestamp` (rows without a timestamp are kept). Builds `text` from `feedback_text` + `feature_requested`, generates an `id`, keeps the raw NPS `score` (parsed as float), and defaults `user_plan` to `'unknown'` if not present. Drops items whose resulting text is 5 characters or shorter.

### Normalise Support Issues
Same 7-day filter pattern (on `timestamp`). Cleans possibly-dirty column keys (handles a stray leading/trailing space in `issue_type`), builds `text` from issue type + description, and converts a `priority` string (`high`/`medium`/`low`) into a numeric severity (1/3/5), from which a derived `score` is calculated as `10 - severity*2` (clamped to minimum 1). `user_plan` is hardcoded to `'unknown'` for all support rows.

### Normalise Call Notes
Filters to the last 7 days by `date`. Cleans dirty keys for `pain_mentioned`/`feature_request`, concatenates them into `text`. `score` is explicitly left `null` (no numeric signal). Adds a fixed `weight: 4` field flagged in the code comment as "very important (high signal)" — note this `weight` field is **not** referenced anywhere downstream (see Known Limitations).

### Normalise Community
Filters to the last 7 days by `date`. Cleans dirty keys for `message`/`platform`. Applies a simple keyword heuristic (`bug`/`confusing`/`issue` → negative; `love`/`amazing` → positive; else neutral) to produce a `sentiment_hint` field. `score` is left `null`. Adds a fixed `weight: 2` field (also unused downstream).

### Merge NPS + Support / Merge + Call Notes / Merge All Sources
Three sequential `n8n-nodes-base.merge` nodes (typeVersion 3), each with default parameters (no configuration overrides shown in JSON — behaves as an append-style combine of the two inputs feeding it). They chain the four branches together two at a time rather than using a single 4-input merge.

### Deduplicate + Cap
Code node. Deduplicates by a composite key of `source + first 80 lowercase chars of text` (drops items with fewer than 10 text characters). Separates the deduplicated list into "pain" items (`score` not null and `≤ 4`) versus everything else, and concatenates pain-first, capping the combined list at **500 items** — meaning non-pain items are more likely to be dropped once the cap is hit. Outputs a single item: `{ items: [...], meta: { total_raw, after_dedup, sent_to_ai, week, by_source } }`. Note that each output item under `items[]` is stripped down to only `source, text, score, user_plan` — the `weight` and `sentiment_hint` fields computed upstream are dropped here and never reach the AI Agent.

### IF: Has Feedback?
Single numeric condition: `meta.sent_to_ai > 0`. True branch proceeds to AI processing; false branch routes to the "no data" Slack notification and the run terminates on that path.

### Slack: No Data This Week
Posts a message to Slack (via `select: user`, targeting `USLACKBOT`/"slackbot" as the recipient) stating no feedback was collected, including the computed week label. Uses the `Slack account` credential.

### AI Agent
LangChain Agent node (`promptType: define`). User prompt template passes the week label, total item count, per-source breakdown, and the full `items[]` array (as JSON) to the model. The system message instructs the model to act as a senior product analyst, cluster feedback into 8–12 themes, compute `frequency`, `sentiment_score` (-1 to 1), and `impact_score` (0–10, boosted for paid-plan mentions, cross-source repeats, or churn/revenue-loss language), suggest one concrete feature/fix per theme, compute a priority score using the same weighted formula later re-applied by the Impact Scorer node, and return **only** a JSON object of the shape `{ themes: [...], summary: "..." }` with no markdown or extra text. The node has no additional tools attached (no `ai_tool` connections in the JSON) — it is a single-shot LLM call, not a tool-using agent.

### OpenRouter Chat Model
LangChain LM node connected to the AI Agent as its `ai_languageModel` input. Model: `nvidia/nemotron-nano-12b-v2-vl:free`, temperature `0.2`. No credential block is visible in the truncated parameters view beyond `model`/`options`; confirm the OpenRouter API credential attached to this node when reviewing directly in n8n.

### Parse & Validate AI Response
Code node that defensively extracts JSON from the AI Agent's output, handling four possible shapes (OpenRouter's native `choices[0].message.content`, a plain `.output` string, a raw string, or an unknown object it JSON-stringifies as a last resort), then attempts direct `JSON.parse`, falling back to extracting a ```` ```json ``` ```` fenced block, then falling back to a regex match of the first `{...}` block. Throws a descriptive error if no valid JSON can be extracted, or if the parsed object lacks a `themes` array. Re-fetches `meta` from the **Deduplicate + Cap** node by name reference (with a hardcoded fallback `{ sent_to_ai: 100, week: 'unknown', by_source: {} }` if that lookup fails). Clamps each theme's `frequency` (0 to `meta.sent_to_ai`), `sentiment_score` (-1 to 1), and `impact_score` (0 to 10), and normalises `sample_quotes`/`sources` to arrays capped at 3/unlimited respectively.

### Impact Scorer & Ranker
Code node. For each theme, computes `freq_ratio` (frequency / total sent-to-AI), `urgency` (`(1 - sentiment) / 2`), and a composite score: `0.40×freq_ratio + 0.35×(impact/10) + 0.25×urgency`, scaled to 0–100 (`composite_score`) with the unrounded value kept as `composite_raw`. Also derives a display sentiment label/emoji (Positive 🟢 / Negative 🔴 / Mixed 🟡 based on ±0.3 thresholds). Sorts all themes descending by composite score (ties broken by impact, then frequency, then original order for stability), takes the top 10, and assigns `final_rank` 1–10.

### Format Slack Message
Code node that renders the top 10 themes into a single Slack-markdown message: numbered/medal emoji per rank, theme name, composite score, sentiment emoji+label, frequency, mapped source labels (NPS/Support/Reddit/Calls/Community — note "Reddit"/`reviews` and "Calls"/`sales` source keys are referenced here but are **not** produced anywhere in the ingestion branches, see Known Limitations), suggested feature, and a truncated (100-char) sample quote. Prepends a header with the week label and raw→dedup→analysed funnel counts plus the per-source breakdown, and an AI-written executive summary. Appends a footer crediting "n8n + Llama-3.3-70B" (note: this is inconsistent with the actual configured model, `nvidia/nemotron-nano-12b-v2-vl:free` — see Known Limitations).

### Send Slack Digest
Posts the formatted message (`{{ $json.message }}`) to Slack, targeting `USLACKBOT`/"slackbot", with `mrkdwn: true`. Uses the `Slack account` credential.

### Split Top10 for Sheets Loop
Splits the `top10` array field out into one item per theme (`n8n-nodes-base.splitOut`, field `top10`). Despite the node's name including "Loop," this is a single Split Out operation, not a SplitInBatches/loop node — see Loop Logic below.

### Write to Sheets History
Appends each split-out theme as a row to the `Sheet5` tab (gid `120873223`) of the same spreadsheet used for ingestion. Maps `theme, frequency, sentiment_score, impact_score, priority_rank, suggested_feature, sample_quote_1-3 (from sample_quotes[0..2]), sources (joined from sources[0..2]), composite_score, composite_raw, freq_ratio, urgency, sent_label, sent_emoji, final_rank` as explicit columns.

## Node Configuration

- **Google Sheets source document** (all read/write nodes): `1NCJA8xuZKRIaF4ahOLnyXYqYAbt4jLXVmyaUUMKBb9c` ("AI Customer Feedback Synthesis & Product Roadmap Prioritiser").
- **Read tabs**: `nps_feedback` (gid=0), `support_tickets` (gid=2143247956), `call_notes` (gid=1679537851), `community_feedback` (gid=1292812068).
- **Write tab**: `Sheet5` (gid=120873223) — note the tab name "Sheet5" is a generic default name, not a descriptive one like the read tabs; confirm this is intentional.
- **Cron**: `0 7 * * 1` (Weekly Trigger).
- **AI model**: `nvidia/nemotron-nano-12b-v2-vl:free` via OpenRouter, temperature `0.2`.
- **Composite priority formula** (applied identically in the AI system prompt and again in Impact Scorer & Ranker): `0.40 × freq_ratio + 0.35 × (impact_score/10) + 0.25 × urgency`, where `urgency = (1 - sentiment_score) / 2`.

## Branching Logic

Single conditional node: **IF: Has Feedback?** — evaluates `{{ $json.meta.sent_to_ai }} > 0` (numeric "larger than" comparison).
- **True branch** → `AI Agent` (full processing pipeline).
- **False branch** → `Slack: No Data This Week` (notification only; no further downstream nodes on this path).

## Loop Logic

No `SplitInBatches`/batch-loop node is present. **Split Top10 for Sheets Loop** is a `splitOut` node — it explodes the `top10` array into N individual items in a single pass (all items proceed downstream together in parallel), not an iterative batch loop with a defined batch size or loop-exit condition. The "Loop" in its name is a naming choice, not a functional loop construct.

## Variables & Expressions

Notable n8n expressions found in node parameters:
- `IF: Has Feedback?` condition: `={{ $json.meta.sent_to_ai }}` compared numerically to `0`.
- `Slack: No Data This Week` text: `={{ $json.meta.week }}` interpolated into the message.
- `Send Slack Digest` text: `={{ $json.message }}` (passes through the pre-formatted message from `Format Slack Message`).
- `Write to Sheets History` column mappings: multiple `={{ $json.<field> }}` expressions, including array-index access like `={{ $json.sample_quotes[0] }}` and `={{ $json.sources[0] }}\n{{ $json.sources[1] }}\n{{ $json.sources[2] }}` (newline-joined first three sources).
- `AI Agent` prompt: `={{$json.meta.week}}`, `={{$json.meta.sent_to_ai}}`, `={{ JSON.stringify($json.meta.by_source, null, 2) }}`, `={{ JSON.stringify($json.items, null, 2) }}`.
- `Parse & Validate AI Response` (Code node) uses the n8n node-reference expression `$('Deduplicate + Cap').first().json.meta` to pull metadata by node name rather than from its immediate input — worth knowing if that upstream node is ever renamed, since this reference would break.

## Credentials Used

| Node | Credential Name | Type | Service |
|---|---|---|---|
| Fetch NPS (Google Sheets) | Google Sheets account | googleSheetsOAuth2Api | Google Sheets |
| Fetch Support Issues (Google Sheets) | Google Sheets account | googleSheetsOAuth2Api | Google Sheets |
| Fetch Call Notes (Google Sheets) | Google Sheets account | googleSheetsOAuth2Api | Google Sheets |
| Fetch Community Posts (Google Sheets) | Google Sheets account | googleSheetsOAuth2Api | Google Sheets |
| Write to Sheets History | (same, via document reference — credential block not separately re-inspected but uses the same googleSheetsOAuth2Api pattern) | googleSheetsOAuth2Api | Google Sheets |
| Slack: No Data This Week | Slack account | slackApi | Slack |
| Send Slack Digest | Slack account | slackApi | Slack |
| OpenRouter Chat Model | _[credential reference not fully visible in extracted parameters — verify in n8n UI]_ | OpenRouter API | OpenRouter (LLM) |

All credentials are referenced by internal n8n ID/name only; no secret values are present in the export (as expected).

## Environment Variables

None found — no `$env.*` references appear anywhere in the workflow's expressions or code.

## Input Data

Trigger has no payload (Schedule Trigger). The effective "input" to the workflow is whatever rows currently exist in the four Google Sheets tabs at execution time — no filters are applied at the Sheets-node level, so all historical rows are fetched each run and then time-windowed to the last 7 days inside each Normalise Code node.

Expected sheet columns (per node notes):
- `nps_responses`: `timestamp, score, comment, email, plan, source, week_tag` — though the Normalise NPS code actually reads `feedback_text`, `feature_requested`, `nps_score`, `timestamp`, `email`, `plan` (a mismatch from the note's column list — see Known Limitations).
- `support_issues`: `timestamp, issue_title, description, severity (1-5), reported_by, plan` per the note, though the code reads `issue_type`, `issue_description`, `priority`, `timestamp` — another naming mismatch (see Known Limitations).
- `call_notes`: `date, contact_name, plan_type, key_points, pain_mentioned` per the note; code reads `pain_mentioned`, `feature_request`, `date`.
- `community`: `date, platform, content, author, upvotes` per the note; code reads `message`, `platform`, `date`.

## Output Data

- **Slack** (`#`-channel or DM to "slackbot" — target is `USLACKBOT`, which is unusual for a team-visible digest; confirm intended recipient/channel): either a "no data this week" notice, or a full formatted weekly digest with ranked themes, scores, and an AI summary.
- **Google Sheets** (`Sheet5` tab): one row per top-10 theme per run, with theme name, frequency, sentiment/impact/composite scores, priority rank, suggested feature, up to 3 sample quotes, sources, and supporting scoring fields — building a historical archive of weekly prioritised themes.

## Data Transformation

Significant reshaping occurs across the pipeline: four heterogeneous sheet schemas are normalised into one common shape (`Normalise *` nodes); the four branches are unified via three sequential Merge nodes; duplicates are removed and the set is stratified-capped (`Deduplicate + Cap`); the AI Agent transforms raw feedback text into structured theme objects; those are re-validated/sanitised (`Parse & Validate AI Response`); a derived composite score is computed and the list is sorted/truncated to 10 (`Impact Scorer & Ranker`); and the ranked result is rendered into both a human-readable Slack message and a flat spreadsheet-row shape.

## API Integrations / External Services

- **Google Sheets API** — read (4 source tabs) and append (1 history tab), via OAuth2 credential `Google Sheets account`.
- **Slack API** — `chat.postMessage`-style calls via the Slack node, used twice (no-data notice, and the full digest), via credential `Slack account`.
- **OpenRouter API** — LLM inference call from the `OpenRouter Chat Model` node, model `nvidia/nemotron-nano-12b-v2-vl:free`.

## Database Operations

Not applicable — no database nodes are present; Google Sheets functions as the data store.

## AI/LLM Integration

- **Node**: `AI Agent` (`@n8n/n8n-nodes-langchain.agent`), backed by `OpenRouter Chat Model` (`@n8n/n8n-nodes-langchain.lmChatOpenRouter`).
- **Model**: `nvidia/nemotron-nano-12b-v2-vl:free`, temperature `0.2`.
- **System prompt** (summarised): acts as a senior product analyst; clusters feedback into 8–12 themes; computes frequency, sentiment (-1 to 1), and impact (0–10, boosted by paid-plan/cross-source/churn signals) per theme; suggests one concrete feature per theme; ranks by the weighted composite formula; must output *only* a JSON object matching a specified `{themes: [...], summary: string}` schema, with explicit rules against inventing data and against over-weighting high-volume/low-impact noise.
- **User prompt**: injects the week label, total item count, per-source breakdown, and the full capped feedback item array as JSON.
- **Tools**: none attached — this is a single-turn structured-output call, not a tool-calling agent, despite using the Agent node type.
- **Downstream consumption**: the AI's JSON response is parsed and defensively re-validated by `Parse & Validate AI Response` (handles malformed/non-JSON output, markdown fencing, and missing-field cases) before being scored numerically by `Impact Scorer & Ranker`.

## Error Handling

No node in the export has `continueOnFail`/`onError` set (none of the JSON node definitions include these fields), and `settings` does not configure an `errorWorkflow`. The only explicit error-handling logic is inside the `Parse & Validate AI Response` Code node, which throws descriptive JavaScript errors (`[Parse Error]...`, `[Validation Error]...`) if the AI's output can't be parsed into the expected `{themes: [...]}` shape — but these are unhandled throws, meaning a malformed AI response would fail the entire workflow execution rather than degrading gracefully (e.g. falling back to a partial digest or a Slack error alert).

## Retry Strategy

No `retryOnFail`, `maxTries`, or `waitBetweenTries` settings are configured on any node in the export — including the Google Sheets reads, the Slack posts, and the OpenRouter call. No retry logic is configured anywhere in this workflow.

## Failure Notifications

None. `Slack: No Data This Week` fires on the IF node's false branch (empty feedback, not an error), not on any execution failure. There is no dedicated error-path Slack/email notification node and no configured `errorWorkflow`.

## Logging

No dedicated logging node. `Write to Sheets History` doubles as an audit trail of every week's top-10 themes, but this only captures successful runs that reach that node — it is not a full execution log. n8n's own built-in execution history (viewable in the n8n UI) is the only record of failed or partial runs.

## Execution Order

`settings.executionOrder: "v1"` — this is the legacy execution order setting in n8n (as opposed to `"v0"`), which affects how the parallel source branches and the later parallel Slack/Sheets branches are interleaved during execution. No further customisation is configured.

## Scheduling Details

Cron expression `0 7 * * 1` translates to: **every Monday at 07:00**. No `settings.timezone` override is present in the export, so the schedule runs in whatever timezone is configured at the n8n instance level (commonly UTC by default) — the node's own note explicitly states "07:00 UTC," which should be confirmed against the actual instance timezone setting.

## Dependencies

- A live, reachable **Google Sheets** document (ID `1NCJA8xuZKRIaF4ahOLnyXYqYAbt4jLXVmyaUUMKBb9c`) with the four expected read tabs populated (via Google Form intake or manual paste, per node notes) and the `Sheet5` write tab present.
- A working **Slack** workspace connection and valid `slackApi` credential.
- A working **OpenRouter** API credential with access to the `nvidia/nemotron-nano-12b-v2-vl:free` model (a free-tier model — note free-tier models can have availability/rate-limit constraints outside this workflow's control).
- No sub-workflow ("Execute Workflow") calls are present.

## Security Considerations

No webhook trigger is used (Schedule Trigger only), so webhook authentication is not applicable. All external calls rely on stored OAuth2/API credentials (Google Sheets, Slack, OpenRouter) referenced by ID — actual secret values are not present in this export, as expected. Beyond this, a full security review (credential scope/least-privilege, spreadsheet sharing permissions, Slack app permission scope) is: _[To be provided by workflow owner / security reviewer]_.

## Performance Considerations

The feedback set is capped at 500 items before being sent to the AI Agent, which bounds the size of the LLM prompt. No pagination is configured on the Google Sheets reads (all rows in each tab are fetched every run, then time-filtered client-side in the Code nodes) — this could become a performance concern as sheets grow very large. Concrete execution timing and resource usage are:

## Execution Time

_[To be provided by workflow owner — requires execution data]_

## Resource Usage

_[To be provided by workflow owner — requires execution data]_

## Testing & Validation

_[To be provided by workflow owner]_

## Troubleshooting

- **No Slack digest arrives, and no "no data" message either**: check the n8n execution log for a thrown error in `Parse & Validate AI Response` — a malformed or non-JSON AI response will halt the workflow silently (no error notification is configured).
- **Digest looks empty/thin**: check `meta.sent_to_ai` in the `Deduplicate + Cap` output — if the four source sheets had no rows within the last 7 days, very little (or nothing) will reach the AI Agent.
- **"No Data This Week" fires even though sheets have recent rows**: verify the `timestamp`/`date` columns in each sheet are parseable by JavaScript's `Date` constructor and are actually populated — the Normalise nodes treat missing/unparseable dates permissively (they keep the row), but malformed *non-empty* dates could still push rows outside the 7-day window unexpectedly.
- **Digest source labels show raw keys instead of friendly names (e.g. "reviews" instead of "🔴 Reddit")**: `Format Slack Message`'s `sourceMap` only maps `nps`, `support`, `reviews`, `sales`, `community` — but the ingestion branches only ever produce `source: 'nps' | 'support' | 'call' | 'community'`. The `call` source (from Normalise Call Notes) has no entry in `sourceMap` and will render as the raw string `call` rather than a friendly label; `reviews` and `sales` are dead entries with no producing branch. See Known Limitations.
- **Google Sheets append fails**: confirm the `Sheet5` tab (gid `120873223`) still exists and its columns haven't been renamed/reordered relative to the hardcoded field mapping in `Write to Sheets History`.

## Known Limitations

- **Column-name/notes mismatch**: each `Fetch *` node's `notes` field describes one set of expected columns (e.g. NPS: `timestamp, score, comment, email, plan, source, week_tag`), but the corresponding `Normalise *` Code node actually reads different field names (e.g. `feedback_text`, `feature_requested`, `nps_score`). This suggests the sheet schema and/or code were updated independently and the notes are stale — worth reconciling against the live sheet's actual headers.
- **Unused `weight` fields**: `Normalise Call Notes` (weight 4) and `Normalise Community` (weight 2) both compute a `weight` field intended to boost high-signal sources, but `Deduplicate + Cap` strips output items down to only `{source, text, score, user_plan}`, so `weight` never reaches the AI Agent or the scoring logic. The "weight" concept appears to be dead code.
- **Unused `sentiment_hint` field**: `Normalise Community`'s keyword-based sentiment hint is similarly dropped by `Deduplicate + Cap` and never used downstream.
- **`sourceMap` incompleteness**: `Format Slack Message`'s `sourceMap` includes `reviews` (Reddit) and `sales` (Calls) labels that no ingestion branch actually produces, while omitting a label for `call` (Call Notes), the source that actually exists. Digest output will show the raw string "call" instead of a friendly label for that source.
- **Hardcoded model credit in Slack message**: the digest footer states "Powered by n8n + Llama-3.3-70B," but the configured model is `nvidia/nemotron-nano-12b-v2-vl:free` — this footer text was not updated when the model was changed.
- **No retry logic** on any external call (Google Sheets, Slack, OpenRouter) — a transient API failure on any node will fail the entire weekly run with no automatic recovery.
- **No error notification path** — a failed run (e.g. AI response parse failure) produces no Slack/email alert; only whoever checks the n8n execution log would notice.
- **Free-tier LLM model dependency**: `nvidia/nemotron-nano-12b-v2-vl:free` is a free-tier OpenRouter model, which may carry availability, rate-limiting, or quality trade-offs versus a paid model — worth confirming this is an intentional cost/quality choice.
- **No pagination on Google Sheets reads**: all rows are always fetched from each source tab regardless of size, with time-filtering only applied afterward in code — this will not scale indefinitely as sheet history grows.
- **Slack recipient targeting**: both Slack nodes post to `USLACKBOT` ("slackbot") rather than a named channel — if the intent is a team-visible weekly digest, this configuration should be reviewed, as messages to slackbot are typically only visible to the posting user.
- **Single point of failure**: the workflow has one Google Sheets document, one Slack credential, and one LLM provider with no fallback — an outage in any one of these stops the entire weekly digest.

## Assumptions

- Assumed the Weekly Trigger's cron schedule runs in UTC as stated in the node's own note, since no explicit `settings.timezone` override exists in the export — actual behavior depends on the n8n instance's configured default timezone.
- Assumed the `Write to Sheets History` node uses the same `googleSheetsOAuth2Api` credential pattern as the other Google Sheets nodes (its full credentials block was not distinctly re-verified beyond the shared document reference).
- Assumed "the business use case" (why this automation matters) based on node naming and structure, since no sticky notes or explicit business-justification text exist in the export.
- Assumed the three sequential `Merge` nodes operate in append/combine mode based on default parameters (no explicit `mode` override is set in their `parameters` blocks).

## Version History / Change Log

_[To be provided by workflow owner]_

## References

_[To be provided by workflow owner]_

## Maintenance Notes

_[To be provided by workflow owner]_
