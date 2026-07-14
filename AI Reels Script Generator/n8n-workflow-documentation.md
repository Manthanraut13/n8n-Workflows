# AI Reels Script Generator — n8n Workflow Documentation

**Workflow ID:** `Va329SF5ZNiZHOwB`
**Status:** Inactive (manual/testing mode)
**Execution Order:** v1

---

## 1. Overview

This workflow automatically generates a complete short-form video script package (hook, script, on-screen captions, and Instagram post caption) whenever a new idea is added to a Google Sheet. It uses four sequential AI calls powered by free OpenRouter models, then logs the results to Google Sheets and sends an email notification.

**Trigger:** New row added to Google Sheet `Reel_Ideas`
**Output:** Appended row in `Sheet2` + email notification
**AI Provider:** OpenRouter (free-tier models only)

---

## 2. Workflow Diagram

```
Google Sheets Trigger
        ↓
   Edit Fields
        ↓
AI Agent - Hook Generator ──── Model (llama-3.3-70b-instruct:free)
        ↓
AI Agent - Script Generator ── Model1 (llama-3.3-70b-instruct:free)
        ↓
AI Agent - Captions Generator ─ Model2 (llama-3.3-70b-instruct:free)
        ↓
AI Agent - Post Caption Generator ─ Model3 (gemma-3-12b-it:free)
        ↓
  Format Output Data
        ↓
  Update Google Sheets (append)
        ↓
  Send Email Notification
```

---

## 3. Node-by-Node Reference

### Node 1: Google Sheets Trigger

| Property | Value |
|---|---|
| **Type** | `n8n-nodes-base.googleSheetsTrigger` |
| **Event** | `rowAdded` |
| **Poll Interval** | Every minute |
| **Spreadsheet** | `AI Reels Script Generator` (`120qggHTYmmm8d--Di00dvuqVVh_B1n3ekku3Bietiz8`) |
| **Sheet** | `Reel_Ideas` (gid=0) |
| **Credential** | `Google Sheets Trigger account` (OAuth2) |

**Purpose:** Polls the `Reel_Ideas` sheet every minute and fires once for each new row detected.

**Output fields:** Raw row data — `Idea`, `Niche`, `Tone`, plus Google Sheets metadata (row number, etc.)

**Next node:** `Edit Fields`

---

### Node 2: Edit Fields (Set Node)

| Property | Value |
|---|---|
| **Type** | `n8n-nodes-base.set` |
| **Mode** | Manual assignment |

**Field Mappings:**

| Output Field | Expression | Type |
|---|---|---|
| `Idea` | `{{ $json.Idea }}` | string |
| `Niche` | `{{ $json.Niche }}` | string |
| `Tone` | `{{ $json.Tone }}` | string |

**Purpose:** Cleans and re-exposes the three input fields from the trigger so all downstream nodes can reference them predictably via `$('Edit Fields').item.json.*`.

**Next node:** `AI Agent - Hook Generator`

---

### Node 3: AI Agent - Hook Generator

| Property | Value |
|---|---|
| **Type** | `@n8n/n8n-nodes-langchain.chainLlm` (Basic LLM Chain) |
| **Prompt Type** | Defined (custom) |
| **Connected Model** | `Model` |

**System Message:**
```
You are a professional short-form video scriptwriter who creates high-retention hooks for Instagram Reels and YouTube Shorts. Your goal is to write hooks that stop scrolling within the first 2 seconds.
```

**User Prompt:**
```
Write ONE scroll-stopping hook for a short-form video.

Video Idea:{{ $json.Idea }}
Niche: {{ $json.Niche }}
Tone: {{ $json.Tone }}

Rules:
- 10 to 12 words maximum
- Start with a strong pattern interrupt (question, bold claim, shocking statement, or "You're doing X wrong")
- Create curiosity that forces viewers to keep watching
- Match the specified tone
- Output ONLY the hook text
```

**Output field:** `text` (the generated hook)

**Next node:** `AI Agent - Script Generator`

---

### Node 4: AI Agent - Script Generator

| Property | Value |
|---|---|
| **Type** | `@n8n/n8n-nodes-langchain.agent` (full AI Agent, not chain) |
| **Prompt Type** | Defined (custom) |
| **Connected Model** | `Model1` |
| **Optional Inputs** | Memory and Tool ports exposed but **not connected** |

**System Message:**
```
You are a viral short-form video scriptwriter who specializes in high-retention Instagram Reels and YouTube Shorts.
You write scripts that sound natural when spoken on camera and keep viewers watching until the end.
```

**User Prompt:**
```
Write a complete spoken script for a 20–40 second Instagram Reel or YouTube Short.

Hook (already created):
{{ $node["AI Agent - Hook Generator"].json.text }}

Video Idea:
{{ $('Edit Fields').item.json.Idea }}

Niche:{{ $('Edit Fields').item.json.Niche }}

Tone:{{ $('Edit Fields').item.json.Tone }}

Script Structure:
1. Hook (first 2 seconds) – Use the provided hook exactly
2. Problem / Context (5–8 seconds) – Why this matters
3. Main Content (10–20 seconds) – 2–3 key points, tips, or story beats
4. CTA / Payoff (3–5 seconds) – Strong ending that drives engagement

Rules:
- Write exactly how someone would speak on camera
- Use casual, natural, conversational language
- Use short sentences
- Add natural pauses using "..."
- Total length: 90–150 words
- Maintain the specified tone throughout
- No stage directions or emojis
- Make it binge-worthy, engaging, and shareable

Output Format:

[Hook]
(hook text)

[Script]
(full spoken script)

Begin.
```

**Output field:** `output` (the full script, including echoed hook)

**Note:** This node uses the LangChain **Agent** node type (not a simple chain), which is why it exposes Memory/Tool connectors in the canvas — but neither is wired up in this build, so it behaves as a single-turn LLM call.

**Next node:** `AI Agent - Captions Generator`

---

### Node 5: AI Agent - Captions Generator

| Property | Value |
|---|---|
| **Type** | `@n8n/n8n-nodes-langchain.chainLlm` (Basic LLM Chain) |
| **Prompt Type** | Defined (custom) |
| **Connected Model** | `Model2` |

**Prompt (single combined prompt, no separate system message):**
```
You are a video editor creating on-screen captions for a short-form video.

FULL SCRIPT:{{ $json.output }}

TASK: Extract 4-6 SHORT on-screen text captions that will appear during the video.

RULES:
- Each caption: 3-8 words maximum
- Highlight the most impactful/quotable moments
- Use power words and emotional triggers
- Format for easy reading (short phrases)
- Separate each caption with " | "

EXAMPLE OUTPUT:
Morning routines FAIL | Here's why | Willpower is LIMITED | Try this instead | Stack your habits | You'll thank me later

OUTPUT (captions only, separated by |):
```

**Input note:** `{{ $json.output }}` pulls directly from the immediately preceding node (`AI Agent - Script Generator`), since chainLlm nodes read from the main input stream by default.

**Output field:** `text` (pipe-separated captions)

**Next node:** `AI Agent - Post Caption Generator`

---

### Node 6: AI Agent - Post Caption Generator

| Property | Value |
|---|---|
| **Type** | `@n8n/n8n-nodes-langchain.chainLlm` (Basic LLM Chain) |
| **Prompt Type** | Defined (custom) |
| **Connected Model** | `Model3` |

**Prompt:**
```
You are a social media manager writing an Instagram Reel caption.

VIDEO TOPIC: {{ $('Edit Fields').item.json.Idea }}
NICHE: {{ $('Edit Fields').item.json.Niche }}
HOOK: {{ $('AI Agent - Hook Generator').item.json.text }}
TASK: Write a complete Instagram caption with hashtags.

CAPTION STRUCTURE:
1. Opening line (repurpose the hook or create intrigue)
2. 2-3 sentences expanding on the video topic
3. Call-to-action (Save this, Share with someone who needs this, Follow for more, etc.)
4. Relevant emoji (2-4 strategically placed)
5. Line break
6. 15-25 hashtags (mix of popular, niche-specific, and long-tail)

RULES:
- Conversational and engaging tone
- Front-load the most important info
- Use line breaks for readability
- Hashtags should be relevant to {{$json.content_niche}}
- Include a clear CTA

OUTPUT:
```

> ⚠️ **Known issue:** `{{$json.content_niche}}` inside the RULES section does not resolve — the correct field name in this workflow is `Niche` (from `Edit Fields`), not `content_niche`. This is a leftover reference and will output blank/undefined. See [Section 7 — Known Issues](#7-known-issues--recommended-fixes) for the fix.

**Output field:** `text` (full Instagram caption with hashtags)

**Next node:** `Format Output Data`

---

### Node 7: Format Output Data (Set Node)

| Property | Value |
|---|---|
| **Type** | `n8n-nodes-base.set` |
| **Mode** | Manual assignment |

**Field Mappings:**

| Output Field | Source Expression |
|---|---|
| `Idea` | `{{ $('Edit Fields').item.json.Idea }}` |
| `Niche` | `{{ $('Edit Fields').item.json.Niche }}` |
| `Tone` | `{{ $('Edit Fields').item.json.Tone }}` *(see note below)* |
| `hook` | `{{ $('AI Agent - Hook Generator').item.json.text }}` |
| `script` | `{{ $('AI Agent - Script Generator').item.json.output }}` |
| `post_caption` | `{{ $json.text }}` (from Post Caption Generator, the immediate input) |
| `captions` | `{{ $('AI Agent - Captions Generator').item.json.text }}` |
| `status` | `"Completed"` (static) |
| `timestamp` | `{{ $now.toString() }}` |

> ⚠️ **Known issue:** The `Tone` field expression is set to `"=FixedExpression={{ $('Edit Fields').item.json.Tone }}"`, which prepends the literal string `FixedExpression=` to the actual tone value. This is leftover placeholder text from node creation and should be cleaned up. See [Section 7](#7-known-issues--recommended-fixes).

**Purpose:** Consolidates all outputs from the four AI nodes plus original input fields into a single clean object ready for storage.

**Next node:** `Update Google Sheets`

---

### Node 8: Update Google Sheets

| Property | Value |
|---|---|
| **Type** | `n8n-nodes-base.googleSheets` |
| **Operation** | `append` |
| **Spreadsheet** | `AI Reels Script Generator` (same file as trigger) |
| **Sheet** | **`Sheet2`** (gid=426474626) — *different tab from the trigger's `Reel_Ideas` sheet* |
| **Credential** | `Google Sheets account` (OAuth2) |

**Column Mapping:**

| Sheet Column | Source |
|---|---|
| `Idea` | `{{ $json.Idea }}` |
| `Niche` | `{{ $json.Niche }}` |
| `Tone` | `{{ $json.Tone }}` |
| `Hook` | `{{ $json.hook }}` |
| `Script` | `{{ $json.script }}` |
| `Captions` | `{{ $json.captions }}` |
| `Post_Caption` | `{{ $json.post_caption }}` |
| `Status` | `{{ $json.status }}` |
| `Timestamp` | `{{ $json.timestamp }}` |

**Purpose:** Appends a new row to `Sheet2` containing the complete generated package — this acts as a results/output log, separate from the input queue sheet.

**Next node:** `Send Email Notification`

---

### Node 9: Send Email Notification

| Property | Value |
|---|---|
| **Type** | `n8n-nodes-base.gmail` |
| **Operation** | Send message |
| **To** | `storysnippents@gmail.com` (hardcoded) |
| **Credential** | `Gmail account` (OAuth2) |

**Subject:**
```
✅ New Reel Script Generated: {{ $json.Idea }}
```

**Body (HTML):**
```html
<h2>🎬 Your Reel Script is Ready!</h2>

<p><strong>Topic:</strong> {{ $json.Idea }}</p>
<p><strong>Niche:</strong> {{ $json.Niche }}</p>

<hr>

<h3>🎯 Hook (First 2 Seconds)</h3>
<p>{{ $json.Hook }}</p>

<h3>📝 Full Script</h3>
<p style="white-space: pre-wrap;">{{ $json.Script }}</p>

<h3>💬 On-Screen Captions</h3>
<p>{{ $json.Captions }}</p>

<h3>📱 Instagram Caption</h3>
<p style="white-space: pre-wrap;">{{ $json.Post_Caption }}</p>

<hr>

<p><em>Generated at: {{ $json.Timestamp }}</em></p>
```

> ⚠️ **Known issue:** This node reads `{{ $json.Hook }}`, `{{ $json.Script }}`, `{{ $json.Captions }}`, `{{ $json.Post_Caption }}` (capitalized), but the incoming data from `Update Google Sheets` actually carries the field names as written by the **Google Sheets append operation's column schema** — which does use these capitalized names (`Hook`, `Script`, `Captions`, `Post_Caption`), so this works correctly **only because** the Google Sheets node's output mirrors its column mapping. If the Sheets node's output structure ever changes, this will break silently (blank email fields).

**Purpose:** Sends a formatted HTML summary email once the row has been successfully logged.

**Next node:** None (end of workflow)

---

## 4. AI Model Configuration Summary

| Node | Model Node | Model ID | Max Tokens | Temperature |
|---|---|---|---|---|
| Hook Generator | `Model` | `meta-llama/llama-3.3-70b-instruct:free` | 100 | 0.8 |
| Script Generator | `Model1` | `meta-llama/llama-3.3-70b-instruct:free` | 400 | 0.7 |
| Captions Generator | `Model2` | `meta-llama/llama-3.3-70b-instruct:free` | 300 | 0.6 |
| Post Caption Generator | `Model3` | `google/gemma-3-12b-it:free` | 400 | 0.7 |

All models are accessed via **OpenRouter** using the credential `OpenRouter account for AI Journal`, and all four are on OpenRouter's **free tier** — no paid usage.

**Temperature logic:**
- Hook (0.8) → highest creativity, needed for punchy/varied hooks
- Script (0.7) → balanced creativity with structural consistency
- Captions (0.6) → lower creativity, prioritizes precision/extraction over invention
- Post Caption (0.7) → balanced, matches conversational copywriting needs

---

## 5. Data Flow Map

```
Trigger Row (Reel_Ideas sheet)
   Idea, Niche, Tone
        ↓
Edit Fields
   Idea, Niche, Tone (cleaned)
        ↓
Hook Generator → outputs: text (hook)
        ↓
Script Generator → outputs: output (full script)
   [reads: Edit Fields.Idea/Niche/Tone, Hook Generator.text]
        ↓
Captions Generator → outputs: text (captions)
   [reads: previous node's json.output directly]
        ↓
Post Caption Generator → outputs: text (caption+hashtags)
   [reads: Edit Fields.Idea/Niche, Hook Generator.text]
        ↓
Format Output Data → outputs: Idea, Niche, Tone, hook, script,
                               captions, post_caption, status, timestamp
   [reads from all upstream nodes by name reference]
        ↓
Update Google Sheets (append to Sheet2)
        ↓
Send Email Notification
   [reads Google Sheets node's column-mapped output]
```

---

## 6. Credentials Required

| Credential Name | Type | Used By |
|---|---|---|
| `Google Sheets Trigger account` | Google Sheets OAuth2 | Google Sheets Trigger |
| `Google Sheets account` | Google Sheets OAuth2 | Update Google Sheets |
| `Gmail account` | Gmail OAuth2 | Send Email Notification |
| `OpenRouter account for AI Journal` | OpenRouter API | Model, Model1, Model2, Model3 |

---

## 7. Known Issues & Recommended Fixes

| # | Issue | Location | Fix |
|---|---|---|---|
| 1 | `{{$json.content_niche}}` doesn't resolve (wrong field name) | AI Agent - Post Caption Generator prompt | Replace with `{{ $('Edit Fields').item.json.Niche }}` |
| 2 | `Tone` field polluted with literal text `FixedExpression=` | Format Output Data node | Change value to `={{ $('Edit Fields').item.json.Tone }}` (remove the `FixedExpression=` prefix) |
| 3 | Trigger reads from `Reel_Ideas`, but output is appended to a **different tab** (`Sheet2`) rather than writing back into the same row | Update Google Sheets node | Either (a) keep as a separate results log intentionally, or (b) switch operation to `update` and match on row number if a single unified sheet is desired |
| 4 | Email node depends on exact column names from the Sheets append step; no fallback if schema changes | Send Email Notification | Consider pulling values directly from `Format Output Data` instead of `Update Google Sheets` output to decouple email content from sheet schema |
| 5 | Script Generator uses full Agent node with unused Memory/Tool ports | AI Agent - Script Generator | Harmless as-is; can be switched to `chainLlm` for consistency with the other three nodes, or left as-is if future tool-calling is planned |

---

## 8. Free-Tier Compliance

✅ Google Sheets — free (Google account)
✅ Gmail — free (Google account)
✅ OpenRouter — all 4 models used are tagged `:free`
✅ n8n — self-hosted / free tier compatible

No paid API calls occur anywhere in this workflow as currently configured.

---

## 9. Execution Notes

- **Trigger polling:** every 1 minute — expect up to a 60-second delay between row creation and workflow start.
- **Total nodes:** 9 functional nodes + 4 language model sub-nodes = 13 nodes total.
- **Execution order setting:** `v1` (legacy execution order; sequential, top-to-bottom based on connections).
- **Workflow is currently INACTIVE** — must be toggled on in n8n for the trigger to run automatically.
