# AI LinkedIn Post Generator — Workflow Documentation

**Workflow ID:** `BksClLAfgdlHAVd9`
**Version:** `cf4c1a20-05c6-4d25-8b50-0baaee776817`
**Status:** Inactive (`active: false`) — must be manually activated
**Execution Order:** v1

---

## 1. Overview

This workflow automatically converts a content idea entered into Google Sheets into a fully written LinkedIn post with AI-generated hashtags, then logs the result to a second sheet and sends a Slack notification.

**Trigger → Generate Post → Generate Hashtags → Combine → Save → Notify**

| Property | Value |
|---|---|
| Total Nodes | 13 |
| AI Nodes | 2 Agent nodes + 2 Chat Model nodes |
| LLM Provider | OpenRouter |
| Model Used | `meta-llama/llama-3.3-70b-instruct:free` (both calls) |
| Data Source | Google Sheets (Sheet1) |
| Data Destination | Google Sheets (Sheet2) |
| Notification Channel | Slack (DM to Slackbot) |
| Polling Interval | Every 1 minute |

> ⚠️ **Note:** The model actually wired into this workflow is **Meta Llama 3.3 70B (free)**, not Claude. If Claude is required, swap the model field on both `OpenRouter Chat Model` and `OpenRouter Chat Model1` nodes to an Anthropic OpenRouter model (e.g. `anthropic/claude-3-haiku`).

---

## 2. Architecture Diagram (Actual Wiring)

```
Google Sheets Trigger
        │
        ▼
  Set Variables
   │           │
   ▼           ▼ (index 1)
AI Post Writer  Merge (index 1 input)
   │
   ▼ (index 0)
  Merge ──combine──▶ AI Hashtag Generator
   │                        │
   │ (index 1)              ▼
   ▼                  Set Variables1
  Merge1 ◀────────────── (index 0)
   │
   ├──▶ Parsed output (Code) ──▶ Merge2 (index 0)
   └──────────────────────────▶ Merge2 (index 1)
                                    │
                                    ▼
                              Update Sheet
                                    │
                                    ▼
                              Notification (Slack)

  OpenRouter Chat Model  ──ai_languageModel──▶ AI Post Writer
  OpenRouter Chat Model1 ──ai_languageModel──▶ AI Hashtag Generator
```

The workflow uses **three chained `Merge` nodes in "combine all" mode** to carry forward context (original idea/audience/tone) alongside each AI output, rather than relying purely on expression references across nodes. This is a valid but somewhat redundant pattern — see Section 8 (Recommendations).

---

## 3. Node-by-Node Reference

### Node 1 — Google Sheets Trigger
| Field | Value |
|---|---|
| Type | `n8n-nodes-base.googleSheetsTrigger` |
| Event | `rowAdded` |
| Poll Interval | Every minute |
| Document | `AI LinkedIn Post Generator` (ID: `14dOQC3gJ5lEABz3Y-mLRHzrkzxwjaaWthc3F5c4-Cbo`) |
| Sheet | `Sheet1` (gid=0) |
| Credential | `Google Sheets Trigger account` (OAuth2) |

**Function:** Polls Sheet1 every 60 seconds. When a new row is detected, it emits the row as JSON and starts the workflow.

**Output fields expected from sheet:** `Idea`, `Target_audience`, ` Tone` *(note: this column name has a leading space in the source data)*.

**Connects to:** `Set Variables`

---

### Node 2 — Set Variables
| Field | Value |
|---|---|
| Type | `n8n-nodes-base.set` (v3.4) |
| Mode | Manual assignments |

**Assignments:**
```
idea             = {{ $json.Idea }}
target_audience  = {{ $json.Target_audience }}
 tone            = {{ $json[' Tone'] }}     ← field name has a leading space
```

**Function:** Normalizes raw trigger data into clean, lowercase variable names for downstream nodes.

⚠️ **Bug risk:** The output field is literally named `" tone"` (with a leading space), copied verbatim from the sheet header. Any downstream expression must reference `$json[' tone']` exactly — `$json.tone` will return `undefined`. Recommend renaming to `tone` (no space) for cleanliness. See Section 8.

**Connects to:** `AI Post Writer` (main) and `Merge` (main, input index 1)

---

### Node 3 — AI Post Writer
| Field | Value |
|---|---|
| Type | `@n8n/n8n-nodes-langchain.agent` (v3) |
| Prompt Type | Defined (static template with expressions) |

**User Prompt (verbatim):**
```
Write a LinkedIn post using the following context:

Content Idea: {{ $json.idea }}

Target Audience: {{ $json.target_audience }}

Tone: {{ $json[' tone'] }}

Requirements:

Length: 150–300 words

Structure:

1. Hook (first sentence MUST capture attention)

2. Insight, story, or lesson in the body

3. Close with a conversation-starting question or CTA

Voice: Conversational, relatable, professional, first-person

Formatting:

Short paragraphs

No hashtags

Max 2 emojis

Goal: Deliver value and drive comments

Return only the final LinkedIn post text, nothing else.
```

**System Message (verbatim):**
```
You are a top-tier LinkedIn content strategist and copywriter known for:

High-engagement writing
Story-based insights
Professional yet relatable tone
Clear value delivery

Your mission is to turn inputs into compelling LinkedIn posts that maximize:
Reader attention
Emotional connection
Comment-driven engagement

Follow these universal rules for every post you write:
Write in first person, sounding authentic and human
Open with a hook in the first line tailored to LinkedIn's feed behavior
Keep paragraphs short (1–3 lines each) for mobile readability
No buzzword stuffing — clarity > jargon
Add 1–2 relevant emojis max, only where they fit naturally
Do NOT include hashtags — they will be added later
Never reference instructions, variables, or explain your reasoning
Output only the post text, no titles, no commentary, no formatting blocks

Your goal:
Transform a simple content idea into a scroll-stopping post that educates, inspires, or sparks conversation.
```

**Model:** connected via `OpenRouter Chat Model` (ai_languageModel input)

**Output:** `output` field containing the raw post text.

**Connects to:** `Merge` (main, input index 0)

---

### Node 4 — OpenRouter Chat Model
| Field | Value |
|---|---|
| Type | `@n8n/n8n-nodes-langchain.lmChatOpenRouter` |
| Model | `meta-llama/llama-3.3-70b-instruct:free` |
| Credential | `OpenRouter account for AI Journal` |

**Function:** Supplies the LLM backend to `AI Post Writer` via the `ai_languageModel` connector (not a `main` data connection — this is a sub-node feeding the Agent node).

---

### Node 5 — Merge
| Field | Value |
|---|---|
| Type | `n8n-nodes-base.merge` (v3.2) |
| Mode | Combine → Combine All |

**Inputs:**
- Index 0 ← `AI Post Writer` (the generated post)
- Index 1 ← `Set Variables` (original idea/audience/tone)

**Function:** Merges the AI-generated post back together with the original context fields so both are available for the next agent and for final logging.

**Output fields include:** `output` (the post text), `idea`, `target_audience`, `' tone'`

**Connects to:** `AI Hashtag Generator` and `Merge1` (input index 1)

---

### Node 6 — AI Hashtag Generator
| Field | Value |
|---|---|
| Type | `@n8n/n8n-nodes-langchain.agent` (v3) |
| Prompt Type | Defined |

**User Prompt (verbatim):**
```
Generate relevant hashtags for this LinkedIn post:

Post Content: {{ $json.output }}

Original Idea: {{ $json.idea }}
Target Audience: {{ $json.target_audience }}

Requirements:

Provide 5–8 hashtags

Space-separated

Mix broad + niche + industry-aligned tags

Properly capitalized (ex: #ProductDesign > #productdesign)

Avoid filler hashtags

Output only the hashtags, nothing else
```

**System Message (verbatim):**
```
You are a LinkedIn hashtag optimization strategist with deep knowledge of:
Search discoverability
Engagement psychology
Industry-specific keyword behavior

Your mission is to generate hashtags that maximize reach + relevancy using the content context provided.

Follow these rules for every response:
Provide only hashtags, nothing else—no sentences, no explanation
Return a single line of hashtags, space-separated
Use a strong mix of:
Broad reach hashtags (high visibility, but not overly saturated)
Topic-specific hashtags (directly tied to post theme)
Audience-focused hashtags (role/industry/community)
Pick 5–8 hashtags total—never fewer, never more
Use proper capitalization
Avoid generic or banned terms (#followforfollow, #like)
Never repeat hashtags across the set

Your ultimate job:
Increase discoverability and put the post in front of the right people—not everyone.
```

**Model:** connected via `OpenRouter Chat Model1`

**Output:** `output` field containing space-separated hashtags.

**Connects to:** `Set Variables1`

---

### Node 7 — OpenRouter Chat Model1
| Field | Value |
|---|---|
| Type | `@n8n/n8n-nodes-langchain.lmChatOpenRouter` |
| Model | `meta-llama/llama-3.3-70b-instruct:free` |
| Credential | `OpenRouter account for AI Journal` (same credential as Node 4) |

**Function:** LLM backend for `AI Hashtag Generator`.

---

### Node 8 — Set Variables1
| Field | Value |
|---|---|
| Type | `n8n-nodes-base.set` (v3.4) |

**Assignment:**
```
Hashtag = {{ $json.output }}
```

**Function:** Renames the Hashtag Generator's raw `output` field to `Hashtag` to avoid collision with the post's `output` field later in the merge chain.

**Connects to:** `Merge1` (input index 0)

---

### Node 9 — Merge1
| Field | Value |
|---|---|
| Type | `n8n-nodes-base.merge` (v3.2) |
| Mode | Combine → Combine All |

**Inputs:**
- Index 0 ← `Set Variables1` (`Hashtag`)
- Index 1 ← `Merge` (post + idea + audience + tone)

**Output fields include:** `Hashtag`, `output` (post text), `idea`, `target_audience`, `' tone'`

**Connects to:** `Parsed output` (index 0) and `Merge2` (input index 1)

---

### Node 10 — Parsed output (Code)
| Field | Value |
|---|---|
| Type | `n8n-nodes-base.code` (v2) |
| Language | JavaScript |

**Code (verbatim):**
```javascript
const post = $input.first().json.output;
const tags = $input.first().json.Hashtag;
const finalPost = `${post}\n\n${tags}`;

return {
  final_post: finalPost,
  character_count: finalPost.length
};
```

**Function:** Concatenates the post body and hashtags with a double line break, and calculates the total character count for LinkedIn's 3,000-character limit check.

**Output fields:** `final_post`, `character_count`

⚠️ **Note:** This Code node returns a single object rather than `[{ json: {...} }]`. n8n's Code node auto-wraps a returned plain object as one item, so this works, but does not follow the strict `{json: {...}}` array convention documented in the Code node best practices. It only processes `$input.first()` — if multiple items ever reach this node it will silently ignore the rest.

**Connects to:** `Merge2` (input index 0)

---

### Node 11 — Merge2
| Field | Value |
|---|---|
| Type | `n8n-nodes-base.merge` (v3.2) |
| Mode | Combine → Combine All |

**Inputs:**
- Index 0 ← `Parsed output` (`final_post`, `character_count`)
- Index 1 ← `Merge1` (`Hashtag`, `output`, `idea`, `target_audience`, `' tone'`)

**Function:** Final consolidation step — brings together every field needed for the sheet-write operation: original context, generated post, hashtags, combined post, and character count.

**Connects to:** `Update Sheet`

---

### Node 12 — Update Sheet
| Field | Value |
|---|---|
| Type | `n8n-nodes-base.googleSheets` (v4.7) |
| Operation | **Append** (not update) |
| Document | `AI LinkedIn Post Generator` |
| Sheet | `Sheet2` (gid=196543033) |
| Credential | `Google Sheets account` (OAuth2) |

**Column Mapping:**
| Sheet2 Column | Value Expression |
|---|---|
| `Idea` | `{{ $json.idea }}` |
| `Target_audience` | `{{ $json.target_audience }}` |
| ` Tone` | `{{ $json[' tone'] }}` |
| `Generated Post` | `{{ $json.final_post }}` |
| `Hashtags` | `{{ $json.Hashtag }}` |
| `Status` | `"Generated"` (static) |
| `Character Count` | `{{ $json.character_count }}` |

**Matching column (unused in append mode):** `Idea`

⚠️ **Design note:** This node **appends a new row to a separate sheet (Sheet2)** rather than updating the original triggering row in Sheet1. This means Sheet1 (the input sheet) is never marked as "processed," and there's no `status` flag on the source row to prevent reprocessing. If a row is edited after being read, the trigger's `rowAdded` event won't refire (which is expected), but there is no visual link between the input row and its output row other than the `Idea` text matching.

**Connects to:** `Notification`

---

### Node 13 — Notification (Slack)
| Field | Value |
|---|---|
| Type | `n8n-nodes-base.slack` (v2.3) |
| Send To | `select: user` → `USLACKBOT` |
| Credential | `Slack account` (Slack API) |

**Message Text (verbatim template):**
```
✅ LinkedIn Post Generated!

📝 Original Idea:{{ $json.Idea }}

📊 Stats:
- Characters:{{ $json['Character Count'] }}
- Status: Ready to post
Post :{{ $json['Generated Post'] }}
```

⚠️ **Important:** This node references `$json.Idea`, `$json['Character Count']`, and `$json['Generated Post']` — these are the **Google Sheets column names**, not the workflow variable names used earlier (`idea`, `character_count`, `final_post`). This only works because n8n's Google Sheets "Append" operation returns the row echoed back using its column headers. If the Update Sheet node's output format changes, this node will break.

**Connects to:** *(end of workflow — no further nodes)*

---

## 4. Google Sheets Schema

### Sheet1 — Input ("AI LinkedIn Post Generator" doc)
| Column | Type | Required | Notes |
|---|---|---|---|
| `Idea` | Text | ✅ | The content idea/topic |
| `Target_audience` | Text | ✅ | Who the post targets |
| ` Tone` | Text | ✅ | **Header has a leading space** — must match exactly when entering column headers |

### Sheet2 — Output ("AI LinkedIn Post Generator" doc)
| Column | Type | Populated By |
|---|---|---|
| `Idea` | Text | Update Sheet (copied from input) |
| `Target_audience` | Text | Update Sheet |
| ` Tone` | Text | Update Sheet |
| `Generated Post` | Text | AI Post Writer output |
| `Hashtags` | Text | AI Hashtag Generator output |
| `Status` | Text | Static `"Generated"` |
| `Character Count` | Number | Code node calculation |

---

## 5. Credentials Required

| Credential Name | Type | Used By |
|---|---|---|
| `Google Sheets Trigger account` | Google Sheets OAuth2 (Trigger) | Google Sheets Trigger |
| `Google Sheets account` | Google Sheets OAuth2 | Update Sheet |
| `OpenRouter account for AI Journal` | OpenRouter API | OpenRouter Chat Model, OpenRouter Chat Model1 |
| `Slack account` | Slack API | Notification |

---

## 6. End-to-End Data Flow Example

**Input row added to Sheet1:**
```
Idea: "The biggest mistake I see new automation builders make..."
Target_audience: "No-code enthusiasts, operations managers"
 Tone: "professional"
```

**After Set Variables:**
```json
{ "idea": "...", "target_audience": "...", " tone": "professional" }
```

**After AI Post Writer + Merge:**
```json
{ "output": "<generated post text>", "idea": "...", "target_audience": "...", " tone": "professional" }
```

**After AI Hashtag Generator + Set Variables1 + Merge1:**
```json
{ "Hashtag": "#Automation #NoCode ...", "output": "<post>", "idea": "...", "target_audience": "...", " tone": "..." }
```

**After Parsed output + Merge2:**
```json
{
  "final_post": "<post>\n\n#Automation #NoCode ...",
  "character_count": 842,
  "Hashtag": "...",
  "output": "...",
  "idea": "...",
  "target_audience": "...",
  " tone": "..."
}
```

**Row appended to Sheet2** with all mapped columns, then **Slack DM sent to Slackbot** with the summary.

---

## 7. Setup Checklist

1. ✅ Create/duplicate the Google Sheet with `Sheet1` (input) and `Sheet2` (output) tabs matching the schemas above — note the **leading space in ` Tone`**.
2. ✅ Connect Google Sheets OAuth2 credentials (two separate credential entries: one for the Trigger, one for Update Sheet — they can point to the same Google account).
3. ✅ Connect an OpenRouter API credential (free tier works with `meta-llama/llama-3.3-70b-instruct:free`).
4. ✅ Connect a Slack API credential with permission to DM the workspace's Slackbot (or swap to a channel).
5. ✅ Set the Google Sheets Trigger's `documentId` and `sheetName` to your actual sheet.
6. ✅ Toggle the workflow **Active** (currently `false` in the JSON).
7. ✅ Add a test row to Sheet1 and confirm within ~60 seconds that Sheet2 receives a new row and Slack fires a notification.

---

## 8. Known Issues & Recommendations

| Issue | Impact | Recommendation |
|---|---|---|
| Field name `" tone"` has a leading space | Fragile — any typo breaks the chain silently (returns `undefined`, not an error) | Rename to `tone` throughout Set Variables, prompts, and Update Sheet mapping |
| Model is Llama 3.3 70B, not Claude | Doesn't match "Claude preferred" requirement | Change `model` param on both OpenRouter Chat Model nodes to an Anthropic model, e.g. `anthropic/claude-3-haiku` or `anthropic/claude-3.5-sonnet` |
| Update Sheet uses **Append**, not update-in-place | Sheet1 (source) never gets a "Processed" flag; risk of no dedup logic | Consider adding a `Status` column to Sheet1 and switching to `appendOrUpdate` matched on row ID, or write a "Processed" flag back to Sheet1 in a parallel branch |
| Notification node references Sheet column names (`$json.Idea`) instead of workflow variable names | Brittle — breaks if Update Sheet's column mapping changes | Reference the pre-append merged data directly, or keep in sync deliberately with a comment/note in the node |
| Three chained `Merge` nodes in "combine all" mode | Adds complexity; harder to debug/maintain | Could be simplified using `$('Set Variables').item.json...` cross-node references instead of merges, reducing node count from 13 to ~9 |
| Code node uses `$input.first()` only | Only processes one item if multiple trigger simultaneously | Acceptable for row-by-row triggers, but note the limitation if input volume ever increases |
| Notification only goes to Slackbot DM (self) | No team visibility | Change `user` to a channel ID, or use `channel` select mode, if broader visibility is wanted |

---

## 9. Optional Enhancements

1. **Add a "Processed" write-back to Sheet1** — After Update Sheet, add a second Google Sheets node using `update` (matched by row number from the trigger) to mark the source row as `Status: Done`, preventing accidental reprocessing and giving a clear audit trail on the input sheet itself.

2. **Add a validation/guard IF node after Parsed output** — Check `character_count <= 3000` (LinkedIn's limit) and route posts that are too long to a "Trim/Regenerate" branch instead of writing them straight to the sheet.
