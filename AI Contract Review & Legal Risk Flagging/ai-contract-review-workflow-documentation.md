# AI Contract Review & Legal Risk Flagging — Workflow Documentation

| Field | Value |
|---|---|
| Workflow Name | AI Contract Review & Legal Risk Flagging |
| Workflow ID | GCqPJ6eUjV0isCbE |
| Version | v1.0 (initial documentation) — internal revision ID: `108d248c-77c5-4956-bd8a-360305ad2aef` |
| Owner | _[To be provided by workflow owner]_ |
| Workflow Last Saved | Not available in export metadata |
| Document Generated | 2026-07-12 |
| Active | No (workflow is currently inactive) |

---

## Purpose

This workflow automates the end-to-end review of contract documents received via Gmail. When an email with a PDF or DOCX attachment arrives, the workflow extracts the contract text, runs two sequential AI analysis passes (clause extraction, then risk scoring), computes a weighted composite risk score, and routes the results to Google Sheets for audit logging and Slack for team notification — all without human intervention until review is warranted.

---

## Business Use Case

Reduces the time legal and operations teams spend doing initial contract triage by automatically flagging high-risk agreements (missing clauses, unfavorable liability terms, unclear IP ownership, etc.) as soon as they arrive in the inbox. _(Inferred from workflow structure — confirm or replace with actual business justification.)_

---

## Workflow Summary

A Gmail Trigger polls the inbox every minute for emails with attachments. When an email is detected, metadata is extracted and normalised, and the attachment is re-downloaded and classified by MIME type (PDF, DOCX, or `.doc`). Depending on file type, one of three text extraction paths runs; all three converge at a **Merge Text Output** node, after which the text is cleaned, hashed for deduplication, and checked against a Google Sheet to prevent duplicate processing. If the document is new and within the 60,000-character processing limit, it is sent to a **Clause Extraction AI** agent (backed by OpenRouter) to identify standard legal clauses. A second **Risk Analysis AI** agent scores each clause category against a risk rubric. A **Risk Scoring Engine** Code node computes a weighted composite score (0–100) and assigns a tier (GREEN / AMBER / RED / CRITICAL). Finally, a **Format Summary** Code node builds a human-readable annotated summary, which is written to Google Sheets and posted to Slack. Documents that exceed the size limit are diverted to a manual-review Slack channel instead.

---

## Workflow Diagram

See attached canvas screenshot (`AI_Contract_Review___Legal_Risk_Flagging.png`).

The workflow has a **branching fan-out / merge / branch** shape:

```
Gmail Trigger
  └─▶ Format Normalizer
        └─▶ Download Attachment
              └─▶ File Type Validator ──┬─▶ PDF Text Extractor ──▶ CLEANED TEXT ──────┐
                                        ├─▶ Text Extractor    ──▶ CLEANED TEXT1 ────┤ Merge Text Output
                                        └─▶ DOCX Text Extractor ▶ CLEANED TEXT2 ───┘
                                                                                      │
                                                                                      ▼
                                                                               CONTENT HASH
                                                                                      │
                                                                                      ▼
                                                                           Deduplication Check (Sheets)
                                                                                      │
                                                                               If (duplicate?)
                                                              (new) false ────────────┘────── (dup) true → [stop]
                                                                  │
                                                           File Size Guard
                                            (>60k chars) ──┘                 └── (≤60k chars)
                                                │                                     │
                                        SLACK MESSAGE                        Clause Extraction AI
                                                │                                     │
                                   MANUAL REVIEW NOTIFICATION               PARSE OUTPUT
                                                                                       │
                                                                            Risk Analysis AI
                                                                                       │
                                                                             Parse Risk JSON
                                                                                       │
                                                                           Risk Scoring Engine
                                                                                       │
                                                                             Format Summary
                                                                                       │
                                                                           Risk Tier Router ──┬─▶ Write to Google Sheets  ──▶ Slack Alert
                                                                                             └─▶ Write to Google Sheets1 ──▶ Slack Alert1
```

---

## Trigger Details

| Property | Value |
|---|---|
| Node Name | Gmail Trigger |
| Trigger Type | Gmail IMAP polling |
| Poll Interval | Every minute (`everyMinute`) |
| Download Attachments | Enabled (`downloadAttachments: true`) |
| Label Filters | None configured (all inbox mail polled) |
| Credential | `Gmail account` (gmailOAuth2, ID: `7XNYGaBFEcVkLMju`) |

The trigger fires once per minute and returns emails with their binary attachment data. The `simple` flag is set to `false`, meaning the full email payload (headers, HTML, text, binary attachments) is returned.

> **Note:** No label filter is set in the trigger. In practice, every email with an attachment will enter the pipeline. You may want to add a `contracts-inbox` label filter to limit scope.

---

## Execution Flow

1. **Gmail Trigger** polls every minute. When a qualifying email arrives, it emits the full email payload including the binary attachment (`attachment_0`).

2. **Format Normalizer** (Set node) extracts key metadata fields from the raw Gmail payload into a clean, named structure: `contract_id`, `sender_email`, `sender_name`, `subject`, `received_at`, `attachment_id`, `attachment_name`, `mime_type`, `file_size_bytes`.

3. **Download Attachment** re-fetches the full email by `contract_id` (message ID) via the Gmail node's Get operation, ensuring the binary attachment is available for subsequent processing.

4. **File Type Validator** (Switch node) inspects `$binary.attachment_0.mimeType` and routes to one of three branches:
   - Output 0 → `application/pdf` → **PDF Text Extractor**
   - Output 1 → `application/vnd.openxmlformats-officedocument.wordprocessingml.document` → **Text Extractor**
   - Output 2 → `application/msword` → **DOCX Text Extractor**

5. **PDF Text Extractor** (Code node): decodes the base64 PDF binary and performs a UTF-8 buffer extraction (fallback method; note: does not use `pdf-parse`, so text quality depends on the PDF's encoding). Produces `extracted_text`, `char_count`, `extraction_method: "buffer_utf8_fallback"`.

6. **Text Extractor** (Code node): uses `mammoth` to extract raw text from DOCX files. Feeds into **CLEANED TEXT1**.

7. **DOCX Text Extractor** (Code node): identical mammoth-based extractor, feeding into **CLEANED TEXT2**.

8. **CLEANED TEXT / CLEANED TEXT1 / CLEANED TEXT2** (Code nodes, one per branch): normalize the extracted text — collapse whitespace, remove repeated line breaks, strip "Page N of M" markers, and trim. Produce `cleaned_contract_text` and updated `char_count`.

9. **Merge Text Output** (Merge node, `append` mode, 3 inputs): waits for whichever of the three extraction branches completes and passes the result forward as a single item.

10. **CONTENT HASH** (Code node): computes a non-cryptographic 32-bit hash (djb2-style) of `extracted_text` and attaches it as `content_hash`. This fingerprint is used for deduplication.

11. **Deduplication Check** (Google Sheets read): looks up the `content_hash` in the `Contract_Review_Log` sheet (Spreadsheet ID: `1b02bzX7kDn_s1KNOVfilKjgfG1TbJNgmHdHSKZaoEZI`). Returns any matching row if found.

12. **If** (duplicate check): compares `$json.content_hash` (from Sheets lookup result) against `$('CONTENT HASH').item.json.content_hash`. If they **match** (true branch → duplicate), execution stops. If no match (false branch → new document), execution continues.

13. **File Size Guard** (IF node): checks if `$('CONTENT HASH').item.json.char_count > 60000`.
    - **True (>60 k chars):** routes to **SLACK MESSAGE** → **MANUAL REVIEW NOTIFICATION** (posts "DROP THE CONTRACT" to Slackbot — a manual-review notification).
    - **False (≤60 k chars):** routes to the AI analysis pipeline.

14. **Clause Extraction AI** (LangChain Agent): uses the `cleaned_contract_text` from CONTENT HASH. The agent is prompted to extract 11 standard legal clause categories (contract type, parties, payment terms, liability, IP ownership, termination, renewal, governing law, confidentiality, indemnification) into a structured JSON schema. Returns the JSON object as its `output` field. Backed by **OpenRouter Chat Model** (model: `z-ai/glm-4.5-air:free`).

15. **PARSE OUTPUT** (Code node): strips markdown fences from the AI response, JSON-parses it, and detects `missing_fields` (any of the 8 required clause categories where `text === null`). Merges the parsed `clauses` object into the item.

16. **Risk Analysis AI** (LangChain Agent): receives the extracted clauses and scores each against a hard-coded risk rubric with tiers `low | medium | high | critical` per clause category. Returns scores and flags for 7 clause dimensions plus `missing_clause_penalties`, `key_concerns[]`, and `recommended_actions[]`. Backed by **OpenRouter Chat Model1** (model: `z-ai/glm-4.5-air:free`).

17. **Parse Risk JSON** (Code node): strips markdown fences, JSON-parses the risk analysis output, and attaches it as `risk_analysis` on the item.

18. **Risk Scoring Engine** (Code node): computes a weighted composite score using the risk assessment:
    - Weighted average of 6 clause risk dimensions (weights sum to 100)
    - Risk values: `low=0`, `medium=40`, `high=75`, `critical=100`
    - Missing clause penalty: `+5` per missing field
    - Final score capped at 100
    - Tier assignment: `<30 = GREEN`, `30–59 = AMBER`, `60–79 = RED`, `≥80 = CRITICAL`

19. **Format Summary** (Code node): assembles a plain-text annotated summary string (`annotated_summary`) covering the overall risk tier, composite score, per-clause risk level and flag, key concerns, and recommended actions.

20. **Risk Tier Router** (Switch node): routes based on `risk_tier`:
    - Output 0 → `AMBER` → **Write to Google Sheets** → **Slack Alert**
    - Output 1 → `RED` → **Write to Google Sheets1** → **Slack Alert1**
    - _(No branch for GREEN or CRITICAL in current configuration)_

21. **Write to Google Sheets / Write to Google Sheets1** (Google Sheets append): writes 13 columns to the `Contract_Review_Log` sheet: `contract_id`, `received_at`, `sender_email`, `attachment_name`, `content_hash`, `composite_score`, `risk_tier`, `payment_terms`, `liability`, `ip_ownership`, `missing_clauses`, `annotated_summary`, `processed_at`.

22. **Slack Alert / Slack Alert1** (Slack post): sends `annotated_summary` as a direct message to `USLACKBOT` (Slackbot). This is functionally a delivery channel — the recipient user ID should be updated to an actual team member or channel.

---

## Node List

| # | Node Name | Type | Purpose | Disabled |
|---|---|---|---|---|
| 1 | Gmail Trigger | Gmail Trigger | Entry point — polls Gmail every minute | No |
| 2 | Format Normalizer | Set | Extracts and renames email metadata fields | No |
| 3 | Download Attachment | Gmail (Get) | Re-fetches full email with attachment binary | No |
| 4 | File Type Validator | Switch | Routes by MIME type to correct extractor | No |
| 5 | PDF Text Extractor | Code (JS) | Decodes PDF binary → raw text (UTF-8 fallback) | No |
| 6 | Text Extractor | Code (JS) | Mammoth DOCX extraction (DOCX/mid-branch) | No |
| 7 | DOCX Text Extractor | Code (JS) | Mammoth DOCX extraction (`.doc` branch) | No |
| 8 | CLEANED TEXT | Code (JS) | Normalises PDF-branch extracted text | No |
| 9 | CLEANED TEXT1 | Code (JS) | Normalises Text Extractor branch text | No |
| 10 | CLEANED TEXT2 | Code (JS) | Normalises DOCX Extractor branch text | No |
| 11 | Merge Text Output | Merge (append, 3 inputs) | Collects output from all three extraction branches | No |
| 12 | CONTENT HASH | Code (JS) | Computes djb2 hash of extracted text for deduplication | No |
| 13 | Deduplication Check | Google Sheets (read) | Looks up content_hash in audit log | No |
| 14 | If | IF | Stops execution if duplicate document detected | No |
| 15 | File Size Guard | IF | Routes oversized docs (>60k chars) to manual review | No |
| 16 | SLACK MESSAGE | Code (JS) | Chunks large document (document chunker) | No |
| 17 | MANUAL REVIEW NOTIFICATION | Slack | Notifies team of oversized/unchunkable contract | No |
| 18 | Clause Extraction AI | LangChain Agent | AI pass 1: extracts legal clauses into JSON schema | No |
| 19 | OpenRouter Chat Model | OpenRouter LLM | LLM backing Clause Extraction AI (GLM-4.5-air-free) | No |
| 20 | PARSE OUTPUT | Code (JS) | Parses AI clause JSON, detects missing fields | No |
| 21 | Risk Analysis AI | LangChain Agent | AI pass 2: scores clauses against risk rubric | No |
| 22 | OpenRouter Chat Model1 | OpenRouter LLM | LLM backing Risk Analysis AI (GLM-4.5-air-free) | No |
| 23 | Parse Risk JSON | Code (JS) | Parses risk analysis JSON from AI | No |
| 24 | Risk Scoring Engine | Code (JS) | Computes weighted composite risk score + tier | No |
| 25 | Format Summary | Code (JS) | Assembles annotated plain-text summary | No |
| 26 | Risk Tier Router | Switch | Routes AMBER to path A, RED to path B | No |
| 27 | Write to Google Sheets | Google Sheets (append) | Logs AMBER-tier contract results | No |
| 28 | Write to Google Sheets1 | Google Sheets (append) | Logs RED-tier contract results | No |
| 29 | Slack Alert | Slack (post) | Sends AMBER summary to Slackbot | No |
| 30 | Slack Alert1 | Slack (post) | Sends RED summary to Slackbot | No |

---

## Node-by-Node Description

### Gmail Trigger
Polls the connected Gmail account every minute using IMAP. Returns the full email message including all headers, body (HTML and plain text), and binary attachments. The `simple: false` setting ensures the raw, unprocessed payload is returned so that metadata like `headers.from` can be parsed downstream.

### Format Normalizer
A Set node that extracts nine metadata fields from the Gmail payload into cleanly named variables. Parses the sender's email address using a regex match on `headers.from` (`/<(.+)>/`) and the sender name by splitting on `<`. These fields travel with the item through the entire pipeline for audit logging.

### Download Attachment
Fetches the email again by message ID (`contract_id`) using the Gmail Get operation with attachment download enabled. This ensures the binary attachment data is available as `attachment_0` for the MIME type check and text extraction steps, even if the trigger's payload was incomplete.

### File Type Validator
A Switch node (mode: Rules) that inspects `$binary.attachment_0.mimeType` using string-equals comparisons. Routes PDFs (output 0), DOCX files (output 1), and legacy `.doc` files (output 2) to their respective extraction nodes. If none of the three MIME types match, no output is produced and the item is silently dropped.

### PDF Text Extractor
A JavaScript Code node that converts the base64-encoded PDF binary into a Node.js Buffer and then to a UTF-8 string. **Important limitation:** this is a raw buffer-to-string conversion, not a proper PDF parsing library call — it will produce garbled text for binary-encoded PDFs and is marked `extraction_method: "buffer_utf8_fallback"`. For production use, replace with a `pdf-parse` library call.

### Text Extractor / DOCX Text Extractor
Both nodes use the `mammoth` library to extract raw text from DOCX files. `Text Extractor` handles DOCX files routed from File Type Validator output 1; `DOCX Text Extractor` handles `.doc` files from output 2. Both reference `items[0].binary.contractFile` — note that the actual binary field from Download Attachment is named `attachment_0`, not `contractFile`; this may cause a runtime error and should be verified.

### CLEANED TEXT / CLEANED TEXT1 / CLEANED TEXT2
Three near-identical Code nodes (one per extraction branch). Each normalises the text by collapsing multiple spaces to single spaces, removing repeated newlines, stripping "Page N of M" patterns, and trimming. Adds `cleaned_contract_text` and an updated `char_count`. Note: these nodes reference `$json.text` — this field should align with the output field name from the preceding extractor (ensure consistency).

### Merge Text Output
A Merge node configured in `append` mode with 3 inputs. Waits for items from CLEANED TEXT (input 0), CLEANED TEXT1 (input 1), and CLEANED TEXT2 (input 2) and passes whichever arrives downstream. In practice, only one branch fires per execution, so Merge acts as a join point rather than a true aggregation.

### CONTENT HASH
Computes a non-cryptographic 32-bit integer hash (djb2 algorithm) of `extracted_text` and stores it as `content_hash` (hex string). This is used as the deduplication key. Note: the hash is applied to `extracted_text` (raw/partially-cleaned) rather than `cleaned_contract_text`.

### Deduplication Check
A Google Sheets read operation on the `Contract_Review_Log` tab. Reads all rows without a specific filter, returning the entire log. The subsequent IF node then compares hashes client-side. On large logs, this becomes increasingly slow — consider adding a filter by `content_hash` column for efficiency.

### If (duplicate check)
Compares `$json.content_hash` (value from the Sheets row, if found) against `$('CONTENT HASH').item.json.content_hash`. The `true` branch (match = duplicate) leads nowhere, effectively stopping the workflow. The `false` branch (no match = new document) continues to File Size Guard.

### File Size Guard
Checks whether `char_count > 60000`. Oversized documents (true branch) are routed to a Slack manual-review notification. Documents within the limit (false branch) proceed to AI analysis.

### SLACK MESSAGE
Despite its name, this is a document chunker — a Code node that splits `extracted_text` into 16,000-character chunks with 800-character overlaps, producing multiple output items tagged with `chunk_index` and `total_chunks`. In the current workflow, these chunks are not consumed by the AI pipeline (the main AI path uses the full text from CONTENT HASH) — this node feeds only into the manual-review Slack notification path.

### MANUAL REVIEW NOTIFICATION
Posts the hardcoded message "DROP THE CONTRACT" to Slackbot (`USLACKBOT`) when a document exceeds the size limit. This is a minimal placeholder; it should be enhanced to include document metadata and routed to a meaningful user or channel.

### Clause Extraction AI
A LangChain Agent node configured with a detailed system prompt instructing the model to act as a "legal document analyst" and return only valid JSON. The user prompt injects the `cleaned_contract_text` from the CONTENT HASH node. It extracts 11 clause categories and identifies `missing_clauses`. Temperature: default (not explicitly set — inferred as 0 via options `{}`).

### OpenRouter Chat Model / OpenRouter Chat Model1
Two separate OpenRouter LLM nodes, each connected as the `ai_languageModel` sub-input to their respective Agent. Both use the model `z-ai/glm-4.5-air:free` (a free-tier GLM model). Both use the same credential (`OpenRouter account for AI Journal`, ID: `YCawzcwbZd7VllSz`).

### PARSE OUTPUT
Strips markdown code fences (` ```json ` patterns) from the AI's raw output, JSON-parses the result, and detects `missing_fields` by checking which of 8 required clause keys have a `null` text value. Falls back to regex extraction (`/{[\s\S]*}/`) if the first parse fails, and sets `parse_error: true` if all parsing fails. Merges the `clauses` object into the item data.

### Risk Analysis AI
A second LangChain Agent with a system prompt defining it as a "legal risk assessment engine." The user prompt includes the stringified clauses from PARSE OUTPUT and a detailed scoring rubric (7 clause dimensions × 4 risk tiers). Returns 7 risk fields, 7 flag strings, `missing_clause_penalties[]`, `key_concerns[]`, and `recommended_actions[]`.

### Parse Risk JSON
Strips markdown fences from the Risk Analysis AI's output, JSON-parses it, and attaches the result as `risk_analysis` on the item. Sets `parse_error: true` if parsing fails.

### Risk Scoring Engine
The core scoring logic in a JavaScript Code node:

| Clause | Weight |
|---|---|
| payment_terms_risk | 15% |
| liability_risk | 25% |
| ip_ownership_risk | 20% |
| termination_risk | 15% |
| renewal_risk | 10% |
| compliance_risk | 15% |

Risk tier → numeric value: `low=0`, `medium=40`, `high=75`, `critical=100`. Weighted average is computed, then a missing-clause penalty of `+5` per missing field is added, with the result capped at 100. If a risk level is absent from the risk_analysis object, it defaults to `"medium"` (score: 40). Assigns a tier label and passes `composite_score` and `risk_tier` downstream.

### Format Summary
Builds a multi-line plain-text string (`annotated_summary`) containing: overall risk tier, composite score, per-clause risk level and flag text, key concerns list, and recommended actions list. This string is used both for the Google Sheets log and the Slack notification.

### Risk Tier Router
Routes based on `risk_tier` string value:
- `"AMBER"` → output 0 → Write to Google Sheets → Slack Alert
- `"RED"` → output 1 → Write to Google Sheets1 → Slack Alert1

No routes exist for `"GREEN"` or `"CRITICAL"` — items with these tiers are silently dropped after Format Summary.

### Write to Google Sheets / Write to Google Sheets1
Both append a row to the `Contract_Review_Log` sheet (tab `gid=0`) of the spreadsheet `1b02bzX7kDn_s1KNOVfilKjgfG1TbJNgmHdHSKZaoEZI`. The column mapping is identical in both nodes. Several fields use cross-node references (`$('CLEANED TEXT').item.json.*`, `$('CONTENT HASH').item.json.*`, `$('Format Summary').item.json.*`). The `missing_clauses` column concatenates up to 5 items from the `missing_clause_penalties[]` array.

### Slack Alert / Slack Alert1
Both post `annotated_summary` as a direct message to `USLACKBOT` (the Slackbot user). Both use the same Slack credential (`Slack account`, ID: `d8m4Qm6C9v27XKtP`). In practice, Slackbot DMs are not seen by humans — the recipient should be changed to an actual team member's Slack user ID or a channel (e.g. `#legal-review`).

---

## Node Configuration

### Risk Scoring Engine — weights and risk values

```javascript
const weights = {
  payment_terms_risk: 15,
  liability_risk:     25,
  ip_ownership_risk:  20,
  termination_risk:   15,
  renewal_risk:       10,
  compliance_risk:    15   // Note: governing_law_risk is NOT in the weights map
};

const riskValues = { low: 0, medium: 40, high: 75, critical: 100 };
// Missing field default: "medium" (value 40)
// Missing clause penalty: +5 per field
// Final score: min(100, weightedAverage + penalty)
// Tiers: <30 GREEN | 30-59 AMBER | 60-79 RED | ≥80 CRITICAL
```

### Clause Extraction AI — system prompt summary
> "You are a legal document analyst. Extract contract clauses with precision. Return ONLY valid JSON. No markdown. No explanation."

Output schema: 11 top-level fields — `contract_type`, `parties`, `effective_date`, `payment_terms`, `liability`, `ip_ownership`, `termination`, `renewal`, `governing_law`, `confidentiality`, `indemnification`, `missing_clauses[]`. Each clause field contains `text` (exact quote or null) plus structured sub-fields.

### Risk Analysis AI — system prompt summary
> "You are a legal risk assessment engine. You score contract clauses against a standard template. Return ONLY valid JSON. Be deterministic."

Risk rubric embedded in user prompt:
- `payment_terms`: low ≤30 days, medium 31–60, high 61–90, critical >90 or unclear
- `liability`: low = uncapped mutual, critical = unilateral cap or none
- `ip_ownership`: low = client owns all, critical = unspecified
- `termination`: low = 30-day notice both parties, critical = no termination clause
- `renewal`: low = manual required, critical = auto with no notice period
- `governing_law`: low = favorable jurisdiction, critical = none specified
- `compliance`: low = GDPR/SOC2 referenced, critical = explicitly excludes obligations

### Google Sheets — column mapping

| Sheet Column | Source Expression |
|---|---|
| contract_id | `$('CLEANED TEXT').item.json.id` |
| received_at | `$('CLEANED TEXT').item.json.date` |
| sender_email | `$('CLEANED TEXT').item.json.to.value[0].address` |
| attachment_name | `$('CLEANED TEXT').item.json.file_name` |
| content_hash | `$('CONTENT HASH').item.json.content_hash` |
| composite_score | `$('Format Summary').item.json.composite_score` |
| risk_tier | `$('Format Summary').item.json.risk_tier` |
| payment_terms | risk + flag from Format Summary |
| liability | risk + flag from Format Summary |
| ip_ownership | risk + flag from Format Summary |
| missing_clauses | First 5 entries of `missing_clause_penalties[]` |
| annotated_summary | `$('Format Summary').item.json.annotated_summary` |
| processed_at | `$now` |

---

## Branching Logic

### File Type Validator (Switch)
| Output | Condition | Next Node |
|---|---|---|
| 0 | `mimeType === "application/pdf"` | PDF Text Extractor |
| 1 | `mimeType === "application/vnd.openxmlformats-officedocument.wordprocessingml.document"` | Text Extractor |
| 2 | `mimeType === "application/msword"` | DOCX Text Extractor |
| _(none)_ | Any other MIME type | Item is silently dropped |

### If (deduplication)
| Branch | Condition | Outcome |
|---|---|---|
| true | `content_hash` from Sheets matches current hash | Execution stops — duplicate skipped |
| false | No match found | Continues to File Size Guard |

### File Size Guard (IF)
| Branch | Condition | Next Node |
|---|---|---|
| true | `char_count > 60000` | SLACK MESSAGE → MANUAL REVIEW NOTIFICATION |
| false | `char_count ≤ 60000` | Clause Extraction AI |

### Risk Tier Router (Switch)
| Output | Condition | Next Node |
|---|---|---|
| 0 | `risk_tier === "AMBER"` | Write to Google Sheets → Slack Alert |
| 1 | `risk_tier === "RED"` | Write to Google Sheets1 → Slack Alert1 |
| _(none)_ | GREEN or CRITICAL | Item silently dropped — **known limitation** |

---

## Variables & Expressions

Key n8n expressions used across the workflow:

| Location | Expression | Purpose |
|---|---|---|
| Format Normalizer | `={{ $json.headers.from.match(/<(.+)>/)[1] }}` | Extracts email address from "Name \<email\>" format |
| Format Normalizer | `={{ $json.headers.from.split('<')[0].trim() }}` | Extracts sender display name |
| If (dedup) | `={{ $json.content_hash }}` vs `={{ $('CONTENT HASH').item.json.content_hash }}` | Cross-node reference for dedup comparison |
| File Size Guard | `={{ $('CONTENT HASH').item.json.char_count }}` | Cross-node char count lookup |
| Clause Extraction AI prompt | `{{ $('CONTENT HASH').item.json.cleaned_contract_text }}` | Injects full contract text into AI prompt |
| Risk Analysis AI prompt | `{{ JSON.stringify($json.clauses) }}` | Injects structured clauses as JSON string |
| Google Sheets | `={{ $('CLEANED TEXT').item.json.id }}` | Cross-node reference for contract ID |
| Google Sheets | `={{ $now }}` | Timestamps the log row with processing time |

---

## Credentials Used

| Node | Credential Name | Credential Type | Service |
|---|---|---|---|
| Gmail Trigger | Gmail account | gmailOAuth2 | Google / Gmail |
| Download Attachment | Gmail account | gmailOAuth2 | Google / Gmail |
| OpenRouter Chat Model | OpenRouter account for AI Journal | openRouterApi | OpenRouter AI |
| OpenRouter Chat Model1 | OpenRouter account for AI Journal | openRouterApi | OpenRouter AI |
| Deduplication Check | Google Sheets account | googleSheetsOAuth2Api | Google Sheets |
| Write to Google Sheets | Google Sheets account | googleSheetsOAuth2Api | Google Sheets |
| Write to Google Sheets1 | Google Sheets account | googleSheetsOAuth2Api | Google Sheets |
| Slack Alert | Slack account | slackApi | Slack |
| Slack Alert1 | Slack account | slackApi | Slack |
| MANUAL REVIEW NOTIFICATION | Slack account | slackApi | Slack |

---

## Input Data

**Source:** Gmail inbox — emails with PDF or DOCX attachments.

**Expected payload shape (from test pin data):**
```
{
  id: "19cd38075a065837",
  headers: { from: "Name <email@domain.com>", subject: "...", date: "..." },
  text: "email body plain text",
  html: "email body HTML",
  date: "2026-03-09T16:48:41.000Z",
  binary: {
    attachment_0: {
      mimeType: "application/pdf",
      fileName: "NDA_AcmeCorp.pdf",
      fileSize: "34.7 kB",
      data: "<base64-encoded binary>"
    }
  }
}
```

---

## Output Data

**Google Sheets (`Contract_Review_Log`):** One new row per processed contract with 13 columns as described in the Node Configuration section.

**Slack (AMBER or RED tier):** A plain-text direct message containing the full annotated summary (risk tier, score, per-clause breakdown, key concerns, recommended actions).

**Slack (oversized document):** A minimal "DROP THE CONTRACT" message to Slackbot.

---

## Data Transformation

| Stage | Transformation |
|---|---|
| Format Normalizer | Flattens nested Gmail headers → flat named fields |
| PDF/DOCX Extractors | Binary attachment → raw string text |
| CLEANED TEXT nodes | Normalises whitespace, strips page numbers |
| CONTENT HASH | Adds fingerprint field for deduplication |
| PARSE OUTPUT | AI JSON string → structured `clauses` object; detects `missing_fields[]` |
| Parse Risk JSON | AI JSON string → structured `risk_analysis` object |
| Risk Scoring Engine | 6 categorical risk labels → single numeric score (0–100) + tier label |
| Format Summary | Numeric/categorical data → human-readable annotated text report |

---

## API Integrations / External Services

| Service | Node(s) | Method / Endpoint | Auth | Purpose |
|---|---|---|---|---|
| Gmail API | Gmail Trigger, Download Attachment | IMAP poll / GET message | OAuth2 | Receive emails with attachments |
| OpenRouter | OpenRouter Chat Model, OpenRouter Chat Model1 | POST `/v1/chat/completions` (via LangChain) | API Key | LLM inference for clause extraction and risk analysis |
| Google Sheets API | Deduplication Check, Write to Google Sheets, Write to Google Sheets1 | GET (read) / POST (append) | OAuth2 | Deduplication log lookup and audit row writing |
| Slack API | Slack Alert, Slack Alert1, MANUAL REVIEW NOTIFICATION | `chat.postMessage` | Bot Token | Team notifications |

---

## AI/LLM Integration

### Model
Both AI agents use **`z-ai/glm-4.5-air:free`** via OpenRouter — a free-tier variant of Zhipu AI's GLM-4.5 model.

> ⚠️ **Free tier caveat:** Free OpenRouter models may have rate limits, lower context windows, slower latency, and no SLA. For production contract review, consider a more capable model (e.g. `anthropic/claude-3-haiku`, `openai/gpt-4o-mini`).

### Clause Extraction AI
- **Role:** Legal document analyst
- **Instruction style:** Strict JSON output only, no markdown
- **Input:** Full `cleaned_contract_text` injected into user message
- **Output:** 11-field JSON schema with exact clause quotes and structured sub-fields
- **Error handling in PARSE OUTPUT:** Markdown fence stripping → JSON.parse → regex fallback → `parse_error: true`

### Risk Analysis AI
- **Role:** Legal risk assessment engine
- **Instruction style:** Deterministic scoring, JSON only
- **Input:** Stringified `clauses` object from PARSE OUTPUT
- **Output:** 7 risk scores + 7 flags + `missing_clause_penalties[]` + `key_concerns[]` + `recommended_actions[]`
- **Error handling in Parse Risk JSON:** Same markdown-stripping → parse pattern; no fallback clause extraction

---

## Error Handling

| Node | Error Handling |
|---|---|
| All nodes | No `continueOnFail` is set on any node — an error in any node halts execution |
| Workflow level | No `errorWorkflow` is configured in `settings` |
| PARSE OUTPUT | Soft error handling: catches JSON parse errors and sets `parse_error: true`, allowing the flow to continue with degraded data |
| Parse Risk JSON | Same soft error pattern as PARSE OUTPUT |
| File Type Validator | No fallback for unsupported MIME types — items are silently dropped |
| Risk Tier Router | No route for GREEN or CRITICAL — items are silently dropped |

**Overall posture:** The workflow has minimal error resilience. A failure in any upstream node (Gmail, extractor, AI call, Sheets write) will silently fail the execution with no notification. An error workflow should be added for production use.

---

## Retry Strategy

No retry logic is configured on any node (`retryOnFail` is not set). This is particularly notable for:
- **OpenRouter AI calls** — subject to rate limits and transient failures
- **Google Sheets operations** — subject to quota limits
- **Gmail API calls** — subject to transient API errors

_For production: add `retryOnFail: true`, `maxTries: 3`, `waitBetweenTries: 5000` on the AI agent nodes and Sheets nodes at minimum._

---

## Logging

The **Write to Google Sheets** nodes serve as the primary audit log. Every successfully processed contract produces one row in `Contract_Review_Log` with contract metadata, hash, score, tier, and the full annotated summary.

Failed executions are logged only in n8n's internal execution log (if execution saving is enabled on the instance).

---

## Execution Order

From `settings.executionOrder: "v1"` — the modern connection-based execution order is used. In v1 mode, parallel branches (e.g. the three text extraction branches) run in dependency order rather than top-to-bottom canvas order.

---

## Dependencies

| Dependency | Type | Used By | Notes |
|---|---|---|---|
| Google Sheets `Contract_Review_Log` | External spreadsheet | Deduplication Check, Write to Sheets nodes | Sheet must exist with correct column headers |
| OpenRouter API | External AI API | Both AI Agent nodes | Free tier; no SLA |
| Gmail inbox | External email account | Gmail Trigger | Requires OAuth2 permissions |
| Slack workspace | External messaging | Slack nodes | Bot must be installed in workspace |
| `mammoth` library | Node.js package | Text Extractor, DOCX Text Extractor | Must be available in n8n's node_modules |

---

## Security Considerations

- **Gmail OAuth2**: Full inbox access. Consider restricting the OAuth scope to read-only if write operations are not needed.
- **Attachment download**: The workflow processes any file presented as a PDF or DOCX. No file size pre-check before download, no malware scanning.
- **Contract data sent to OpenRouter**: Full contract text (potentially including sensitive commercial terms, PII of counterparties) is transmitted to a third-party LLM API using a free-tier account. Ensure this aligns with your data handling policies and the counterparty's confidentiality expectations.
- **Slack Slackbot target**: Sending to `USLACKBOT` means messages go to the Slackbot DM, which is effectively unmonitored. Update to a real user or channel.
- **No webhook auth**: Not applicable (this is a polling trigger, not a webhook).
- **Credentials**: All credentials are referenced by name/ID only in the export — no secret values are exposed.

_[Full security review to be completed by workflow owner / security team]_

---

## Performance Considerations

- **Polling frequency**: Every 60 seconds = 1,440 Gmail API calls/day. Gmail API free quota is 1 billion units/day; each poll costs ~5 units. Well within limits for typical usage.
- **Deduplication Check reads the entire sheet**: No server-side filter. Performance degrades linearly with log size. Add a column filter on `content_hash` for large logs.
- **AI latency**: Two sequential LLM inference calls on a free-tier model. Expect 5–30 seconds per call; total pipeline latency may be 30–90 seconds per contract.
- **60,000-character limit**: Roughly 15,000–20,000 words or ~40–60 pages. Adequate for most standard contracts.
- **SLACK MESSAGE chunker**: Code present but chunks are not consumed by the AI pipeline — they go only to the manual review notification. The chunking logic (16k chars, 800-char overlap) is dormant in the main AI flow.

---

## Execution Time

_[To be provided by workflow owner — requires live execution data]_

---

## Resource Usage

_[To be provided by workflow owner — requires live execution data]_

---

## Testing & Validation

_[To be provided by workflow owner]_

**Suggested test cases:**
- Valid PDF NDA → should produce AMBER or RED output in Sheets + Slack
- Duplicate submission of same PDF → should be silently skipped at dedup step
- DOCX file → should route through Text Extractor path
- Unsupported file type (e.g. `.xlsx`) → should be silently dropped at File Type Validator
- Document > 60,000 characters → should trigger MANUAL REVIEW NOTIFICATION
- AI returns malformed JSON → should set `parse_error: true` and continue

---

## Troubleshooting

| Symptom | Likely Cause | Where to Check |
|---|---|---|
| No items processed despite emails arriving | Gmail Trigger filter or OAuth issue | n8n execution log; check Gmail credential is active |
| `undefined` for `extracted_text` in DOCX path | Binary field name mismatch — extractors reference `contractFile` but field is `attachment_0` | DOCX Text Extractor / Text Extractor Code node; update `binary.contractFile` to `binary.attachment_0` |
| Duplicate contracts still being processed | Deduplication Check not finding matching hash — check if sheet has any rows yet, or if hash field name changed | Deduplication Check node output; If node conditions |
| `parse_error: true` in Sheets log | AI returned non-JSON or heavily markdown-wrapped output | PARSE OUTPUT / Parse Risk JSON nodes; consider switching to a more instruction-following model |
| GREEN or CRITICAL contracts not logged | Risk Tier Router has no route for these tiers | Risk Tier Router Switch node; add output branches for GREEN and CRITICAL |
| Slack messages go unseen | Target is `USLACKBOT`, not a real team channel | Slack Alert nodes; update `user` to actual Slack user ID or switch to `channel` mode |
| AI agent returns empty output | OpenRouter free-tier rate limit or model unavailability | OpenRouter dashboard; add retry logic or switch model |
| `$('CLEANED TEXT').item.json.*` returns null in Sheets | Cross-node reference path issue — CLEANED TEXT only fires on the PDF branch | Verify which branch fired; consider using a single canonical cleaned-text field name |

---

## Known Limitations

1. **GREEN and CRITICAL contracts are silently dropped** — Risk Tier Router only routes AMBER and RED. Contracts scoring <30 (GREEN) or ≥80 (CRITICAL) produce no output after Format Summary.

2. **PDF text extraction is a fallback method** — Buffer UTF-8 conversion will produce garbled output for binary-encoded or scanned PDFs. `pdf-parse` or `pdfjs-dist` should be used for production.

3. **DOCX extractor references wrong binary field** — `binary.contractFile` is referenced, but the field name from Download Attachment is `attachment_0`. This will cause a runtime error on DOCX files.

4. **No fallback for unsupported MIME types** — `.xlsx`, `.txt`, `.jpg` attachments are silently dropped with no notification to the sender.

5. **No retry logic anywhere** — A transient AI or Sheets API error will silently fail the execution.

6. **No error workflow configured** — No alert is sent when a workflow execution fails.

7. **Deduplication reads entire sheet** — Will slow down on large logs; no indexed or filtered lookup.

8. **Slackbot as recipient** — Messages to `USLACKBOT` are not seen by humans in typical Slack configurations.

9. **Governing_law_risk absent from scoring weights** — The Risk Analysis AI scores governing_law_risk, but it is not included in the Risk Scoring Engine's `weights` object, so it has zero effect on the composite score.

10. **Free-tier AI model** — `z-ai/glm-4.5-air:free` has no uptime SLA and may produce inconsistent structured output.

11. **Workflow is currently inactive** — `active: false` in the export; must be activated before live use.

---

## Assumptions

1. The `mammoth` npm package is installed in the n8n instance's node environment (`npm install mammoth`).
2. The Google Sheet `Contract_Review_Log` exists at the referenced spreadsheet ID with column headers matching the mapped field names.
3. The Gmail OAuth2 credential has sufficient scope to read emails and download attachments.
4. The Slack Bot Token has `chat:write` permission for the target users/channels.
5. The `cleaned_contract_text` field referenced in the Clause Extraction AI prompt is populated from the CONTENT HASH node — this works because the AI node's prompt references `$('CONTENT HASH').item.json.cleaned_contract_text`, not the current item's `$json`.
6. Only one attachment per email is processed (`attachment_0`). Multi-attachment emails will only process the first attachment.
7. The workflow is intended to process one email per poll cycle; behaviour with multiple matching emails per poll is not documented.
8. `sender_email` expression (`headers.from.match(/<(.+)>/)[1]`) assumes the "From" header always includes an angle-bracket-enclosed address; will throw if the header is a bare address.

---

## Version History / Change Log

_[To be provided by workflow owner]_

---

## References

- OpenRouter API documentation: https://openrouter.ai/docs
- GLM-4.5-air model info: https://openrouter.ai/z-ai/glm-4.5-air
- Mammoth.js (DOCX extraction): https://github.com/mwilliamson/mammoth.js
- n8n LangChain Agent node docs: https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent/
- Google Sheets API quota: https://developers.google.com/sheets/api/limits

---

## Maintenance Notes

_[To be provided by workflow owner]_

**Suggested maintenance actions:**
- Replace `z-ai/glm-4.5-air:free` with a production-grade model when moving out of prototype
- Fix DOCX binary field reference (`contractFile` → `attachment_0`)
- Add routes for GREEN and CRITICAL in Risk Tier Router
- Configure an error workflow for execution failure notifications
- Add retry logic to AI agent nodes and Google Sheets write nodes
- Replace Slackbot recipient with actual team channel
- Add Gmail label filter to restrict intake to contract-tagged emails only
- Monitor Google Sheets log size and implement a filtered deduplication query when rows exceed ~5,000
