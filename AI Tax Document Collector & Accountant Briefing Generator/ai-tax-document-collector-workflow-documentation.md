# AI Tax Document Collector & Accountant Briefing Generator — Workflow Documentation

| Field | Value |
|---|---|
| **Workflow Name** | AI Tax Document Collector & Accountant Briefing Generator |
| **Workflow ID** | `IpWjx8vdr81qW6uq` |
| **n8n Version ID** | `cea70f1a-7116-41e1-b698-a113f0ab8dde` _(internal revision hash, not a semantic version)_ |
| **Version** | v1.0 _(initial documentation)_ |
| **Owner** | _[To be provided by workflow owner]_ |
| **Workflow Status** | Inactive _(not yet activated on n8n instance)_ |
| **Last Saved** | _Not recorded in export_ |
| **Doc Generated** | 2026-07-14 |
| **Execution Order** | v1 (connection-based) |

---

## Purpose

This workflow automates the full tax document collection and accountant briefing pipeline. It collects financial documents from two sources — a Google Drive folder and Gmail attachments — normalises and extracts their content, uses an AI agent to classify each document into a financial category, checks collected documents against a required tax checklist, sends an automated email reminder when items are missing, and finally uses a second AI agent to generate a professional, six-section accountant briefing that is saved back to Google Drive and emailed to the accountant.

---

## Business Use Case

> _Inferred from workflow structure — confirm or replace with actual business context._

This workflow targets small business owners and sole traders preparing for an annual tax filing. It eliminates the manual effort of gathering documents, chasing clients for missing paperwork, and writing summary reports — tasks that typically consume several hours per client per tax season. By automating collection, classification, gap detection, and briefing generation, it lets an accountant focus on review and advisory work rather than administrative chasing.

---

## Workflow Summary

The workflow fires every Monday at 09:00. It simultaneously scans a designated Google Drive folder for all files and fetches recent Gmail messages that contain attachments with tax-related subject keywords. Binary file content is extracted from both sources and passed through separate normalisation Code nodes before being merged into a single item stream. A Filter node removes any empty placeholder items, then an Edit Fields node stamps uniform metadata on every document. Documents then enter a Loop Over Items node (batch size 5) where each batch is fed to an AI Categorization Agent backed by an OpenRouter free model; the agent classifies each file and extracts key financial fields (amount, vendor, date), and the result is parsed and written as a row to the Google Sheets document tracker. Once all documents are processed, an Aggregate node collects them; a Checklist Engine Code node counts documents per category, computes financial totals, and identifies what is missing against a hardcoded required-document list. The checklist is expanded into rows and written to a second Sheets tab. An IF node then routes: if documents are missing, a formatted HTML reminder email is sent to the client; in parallel (false branch), a second Code node builds a structured prompt that a Briefing Generation Agent uses to produce a formal six-section accountant report. The briefing text is extracted, encoded to binary, uploaded to Google Drive as a `.txt` file, the run is logged to a third Sheets tab, and a final email containing a briefing preview is dispatched to the accountant.

---

## Workflow Diagram

See the attached canvas screenshot. The workflow has the following shape:

```
                    ┌─────────────────────┐
                    │   Schedule Trigger   │
                    └────────┬────────────┘
               ┌────────────┘ └────────────────┐
               ▼                               ▼
     ┌──────────────────┐          ┌────────────────────┐
     │ GDrive List Files│          │ Gmail Fetch Emails  │
     └────────┬─────────┘          └──────────┬─────────┘
              ▼                               ▼
     ┌──────────────────┐          ┌────────────────────┐
     │  Download file   │          │  Normalize Gmail    │
     └────────┬─────────┘          └──────────┬─────────┘
              ▼                               │
     ┌─────────────────────┐                 │
     │Normalize documents  │                 │
     │     data            │                 │
     └────────┬────────────┘                 │
              └──────────────┬───────────────┘
                             ▼
                    ┌────────────────┐
                    │     Merge      │
                    └───────┬────────┘
                            ▼
                    ┌────────────────┐
                    │     Filter     │
                    └───────┬────────┘
                            ▼
                    ┌─────────────────────┐
                    │ Edit Fields:Metadata│
                    └───────┬─────────────┘
                            ▼
                    ┌────────────────────┐
                    │  Loop Over Items   │◄──────────────────────┐
                    └───┬────────────────┘                        │
                [done]  │  [loop]                                 │
                        ▼                                         │
               ┌─────────────────────────┐                       │
               │ AI Categorization Agent │                        │
               │  + OpenRouter Model     │                        │
               └────────────┬────────────┘                       │
                            ▼                                     │
                   ┌────────────────┐                            │
                   │Parse AI Response│                            │
                   └───────┬─────────┘                           │
                           ▼                                      │
               ┌───────────────────────────┐                     │
               │ Sheets: Upsert Doc Row    │─────────────────────┘
               └───────────────────────────┘
        ▼ (done output from Loop)
 ┌──────────────────────┐
 │ Aggregate All Results│
 └──────────┬───────────┘
            ▼
 ┌───────────────────────────┐
 │ Checklist Engine (Code)   │
 └────────────┬──────────────┘
              ▼
 ┌────────────────────┐
 │   Structure data   │
 └────────┬───────────┘
          ▼
 ┌─────────────────────────────┐
 │Sheets: Update Checklist Tab │
 └────────────┬────────────────┘
              ▼
 ┌──────────────────────────┐
 │   IF: Missing Documents? │
 └───┬──────────────────────┘
 [true]                [false]
   ▼                      ▼
┌──────────────┐  ┌──────────────────────────┐
│Gmail: Send   │  │ Code: Build Briefing     │
│Reminder      │  │ Prompt                   │
└──────────────┘  └──────────────┬───────────┘
                                 ▼
                  ┌──────────────────────────────┐
                  │ AI Briefing Generation Agent  │
                  │  + OpenRouter Model1          │
                  └────────────┬─────────────────┘
                               ▼
                  ┌──────────────────────────┐
                  │ Code: Extract Briefing   │
                  │ Text                     │
                  └──────────────┬───────────┘
                                 ▼
                  ┌──────────────────────────┐
                  │   Convert to Binary      │
                  └──────────────┬───────────┘
                                 ▼
                  ┌────────────────────────────────────┐
                  │ Google Drive: Save Briefing as     │
                  │ Text File                          │
                  └────────────────┬───────────────────┘
                                   ▼
                       ┌───────────────────┐
                       │     Log Run       │
                       └──────────┬────────┘
                                  ▼
                       ┌──────────────────────────┐
                       │ Gmail: Send Final Report  │
                       └──────────────────────────┘
```

---

## Trigger Details

| Field | Value |
|---|---|
| **Trigger Node** | `Schedule Trigger` |
| **Type** | Schedule / Cron |
| **Node Type** | `n8n-nodes-base.scheduleTrigger` v1.3 |
| **Schedule** | Every week on **Monday** at **09:00** (server timezone) |
| **Activation** | Automatic — fires without any manual input |
| **Timezone** | Inherits n8n instance default (no override set in workflow settings) |

---

## Scheduling Details

The workflow uses a weekly rule: `field: "weeks"`, `triggerAtDay: [1]` (Monday = 1), `triggerAtHour: 9`. No cron expression is stored — this is the Schedule Trigger UI format. No explicit timezone is set in `settings.timezone`, so the n8n server's system timezone governs the schedule; confirm this is the expected timezone for the intended client population.

---

## Execution Flow

1. **Schedule Trigger** fires every Monday at 09:00 and fans out to two parallel branches simultaneously:

   **Branch A — Google Drive:**
   2. **GDrive List Files** searches the configured Drive folder (`1dyFx4IlKAUcdXngaKhL23lbuboaTOKNM`, cached as folder "n8n") for all files (all types), returning all results with no limit.
   3. **Download file** downloads the binary content of each file returned, using `$json.id` as the file identifier.
   4. **Normalize documents data** processes each downloaded binary: CSV files are parsed into structured JSON arrays; TXT files are read as plain text; PDFs get a raw byte-string preview (first 2,000 chars); all other types produce an "unsupported" marker. Outputs `{ fileName, type, data|content|raw_preview }`.

   **Branch B — Gmail:**
   5. **Gmail Fetch Emails** fetches up to 50 messages matching `has:attachment (subject:tax OR subject:invoice OR subject:receipt OR subject:statement)` received in the last 6 months.
   6. **Normalize Gmail** iterates all binary properties on each message, filters to six supported MIME types (PDF, CSV, JPEG, PNG, XLS, XLSX), and produces one flat item per attachment with standard metadata (`id`, `name`, `mimeType`, `createdTime`, `size`, `source: 'gmail'`). Emits `[{ json: { _empty: true } }]` if no qualifying attachments are found.

   **Merge:**
   7. **Merge** (Append mode) combines Branch A and Branch B outputs into a single flat stream.
   8. **Filter** removes any items where `$json._empty` exists — this cleans up the empty-guard item from Normalize Gmail.

   **Metadata & Loop:**
   9. **Edit Fields: Metadata** stamps a uniform 12-field schema on every item: `doc_id`, `file_name`, `file_extension`, `mime_type`, `upload_date`, `source`, `size_kb`, `email_subject`, `drive_link`, `status: "pending"`, `data`, `raw_preview`.
   10. **Loop Over Items** begins a batch loop (size 5). The **loop output (main:1)** processes the current batch; the **done output (main:0)** fires once all batches are consumed.

   **Per-batch AI categorization (iterates until all documents processed):**
   11. **AI Categorization (OpenRouter)** — an n8n AI Agent node — receives each item with its metadata and extracted content. The `OpenRouter Chat Model` sub-node (model: `openrouter/free`) provides the LLM. The agent is instructed to return a strict JSON object: category, confidence (0–100), extracted data (amount, total_amount, date, vendor, account_number, transaction_count), and flags (is_blurry, is_incomplete, multiple_transactions).
   12. **Parse AI Response** reads `$json.output` (the Agent's output field), strips any markdown fences, JSON-parses the result, and maps fields to the standard schema. Falls back to `{ category: 'unknown', confidence: 0, notes: 'parse error' }` on any failure. Also double-parses output to extract `flags.is_incomplete` and `flags.multiple_transactions`.
   13. **Google Sheets: Upsert Document Row** appends a 17-column row to Sheet1 of the tracking spreadsheet, combining original metadata (referenced via `$('Loop Over Items').item.json.*`) with the AI classification result.
   14. Output of Sheets node **loops back to Loop Over Items input 0** — this is the loop-back connection that drives the batch iteration. Steps 11–14 repeat until all batches are done.

   **Post-loop analysis:**
   15. **Aggregate All Results** (done output from Loop) collects all processed items into `{ all_documents: [...] }` — a single item containing the full document array.
   16. **Checklist Engine (Code Node)** counts documents per category, computes `total_expenses` (sum of receipt + invoice amounts), `total_income` (sum of payroll amounts), `net_position`, a per-vendor spend map, and flags any document with `extracted_amount > 10000` as an anomaly. Compares counts against five hardcoded requirements (see Branching Logic). Outputs a rich summary object including `checklist`, `missing_items`, `complete_items`, `has_missing`, and financial totals.
   17. **Structure data** expands the `checklist` array into individual items (one per required category) so the Sheets node can append one row per category.
   18. **Google Sheets: Update Checklist Tab** appends the expanded checklist rows to Sheet2 (`Sheet2`, gid `666909649`), writing `run_date`, `key`, `label`, `required`, `found`, `complete` (YES/NO), `missing_count`.
   19. **IF: Missing Documents?** evaluates whether `$('Checklist Engine (Code Node)').item.json.all_documents[1].missing_flag` is boolean true.

   **IF TRUE (missing documents detected):**
   20. **Gmail: Send Missing Document Reminder** sends an HTML email to `$env.CLIENT_EMAIL` listing each missing document type in a formatted table with required/found/still-needed counts.

   **IF FALSE (all documents collected):**
   21. **Code: Build Briefing Prompt** constructs a richly structured prompt string injecting the full financial summary (income, expenses, net position, top vendors, anomalies, document list capped at 25 items) into a template that instructs the LLM to produce six named sections.
   22. **AI Briefing Generation (OpenRouter)** — a second AI Agent — receives the prompt. The `OpenRouter Chat Model1` sub-node (model: `nvidia/nemotron-3-nano-30b-a3b:free`, max tokens: 2000, temperature: 0.2) generates the formal briefing.
   23. **Code: Extract Briefing Text** reads `$json.output`, derives a `doc_title` (`"Tax Briefing – <Month Year>"`), and emits either the clean briefing text or an error string with `generation_error: true`.
   24. **Convert to Binary** encodes `briefing_text` as a UTF-8 base64 binary property named `briefing_binary` so it can be uploaded as a file.
   25. **Google Drive: Save Briefing as Text File** uploads the binary to the same Drive folder (`1dyFx4IlKAUcdXngaKhL23lbuboaTOKNM`), naming it `<doc_title>.txt`.
   26. **Log Run** appends a summary row to the "Log Run" sheet (gid `1548642444`): `run_date`, `missing_count`, `briefing_generated` (full briefing text — see Known Limitations), `doc_title`.
   27. **Gmail: Send Final Report** sends a preview email to `storysnippents@gmail.com` with the first 1,200 characters of the briefing and a reference to the saved Drive file.

---

## Node List

| # | Node Name | Type | Role / Purpose | Disabled |
|---|---|---|---|---|
| 1 | Schedule Trigger | Schedule Trigger | Weekly cron entry point | No |
| 2 | GDrive List Files | Google Drive | Search Drive folder for all files | No |
| 3 | Download file | Google Drive | Download binary content of each file | No |
| 4 | Normalize documents data | Code (JS) | Extract text/CSV/PDF content from Drive binaries | No |
| 5 | Gmail Fetch Emails | Gmail | Fetch emails with tax-related attachments | No |
| 6 | Normalize Gmail | Code (JS) | Flatten Gmail attachments into standard items | No |
| 7 | Merge | Merge | Combine Drive and Gmail document streams | No |
| 8 | Filter | Filter | Remove empty placeholder items | No |
| 9 | Edit Fields: Metadata | Edit Fields (Set) | Stamp uniform 12-field metadata schema | No |
| 10 | Loop Over Items | Split In Batches | Batch-process documents (size 5) with loop-back | No |
| 11 | OpenRouter Chat Model | OpenRouter LM | LLM sub-node for categorization agent | No |
| 12 | AI Categorization (OpenRouter) | AI Agent | Classify document + extract financial fields | No |
| 13 | Parse AI Response | Code (JS) | Parse and map AI agent output; handle errors | No |
| 14 | Google Sheets: Upsert Document Row | Google Sheets | Append categorized doc row to Sheet1 | No |
| 15 | Aggregate All Results | Aggregate | Collect all processed items into single array | No |
| 16 | Checklist Engine (Code Node) | Code (JS) | Count by category, compute totals, flag missing | No |
| 17 | Structure data | Code (JS) | Expand checklist array into one item per category | No |
| 18 | Google Sheets: Update Checklist Tab | Google Sheets | Append checklist status rows to Sheet2 | No |
| 19 | IF: Missing Documents? | IF | Route based on whether required docs are missing | No |
| 20 | Gmail: Send Missing Document Reminder | Gmail | Email client listing missing document categories | No |
| 21 | Code: Build Briefing Prompt | Code (JS) | Construct structured LLM prompt with all financials | No |
| 22 | OpenRouter Chat Model1 | OpenRouter LM | LLM sub-node for briefing agent (Nemotron 30B) | No |
| 23 | AI Briefing Generation (OpenRouter) | AI Agent | Generate six-section accountant briefing | No |
| 24 | Code: Extract Briefing Text | Code (JS) | Extract briefing text from agent output field | No |
| 25 | Convert to Binary | Code (JS) | Encode briefing text to base64 binary | No |
| 26 | Google Drive: Save Briefing as Text File | Google Drive | Upload briefing as `.txt` to Drive folder | No |
| 27 | Log Run | Google Sheets | Append run summary row to Log Run sheet | No |
| 28 | Gmail: Send Final Report | Gmail | Email briefing preview to accountant | No |

---

## Node-by-Node Description

### Schedule Trigger
Fires the workflow automatically every Monday at 09:00 using a weekly rule (`field: weeks`, day: Monday, hour: 9). No payload — the trigger outputs an empty item `{}`. The schedule is controlled by the n8n server timezone since no override is configured.

### GDrive List Files
Searches Google Drive using the `fileFolder` resource with `searchMethod: query` in "My Drive", filtered by folder ID `1dyFx4IlKAUcdXngaKhL23lbuboaTOKNM` (cached display name: "n8n"), searching all file types, with `returnAll: true`. Returns all file metadata objects including `id`, `name`, `mimeType`, `createdTime`, `size`, `webViewLink`. This is the source node for Drive-originated documents.

### Download file
Downloads the binary content of each file listed by GDrive List Files. Uses `operation: download` and resolves the file ID dynamically via `={{ $json.id }}`. Outputs a binary property named `data` containing the raw file bytes alongside the file's metadata. Required to enable content extraction in the next Code node.

### Normalize documents data
A Code node (Run Once for Each Item pattern via `items` array) that processes each downloaded Drive binary. **CSV files** are parsed into a JSON array using a simple header-row split. **TXT files** are read as plain UTF-8 text. **PDF files** receive a best-effort raw byte-string preview (first 2,000 chars — not true PDF text extraction; binary data will be mostly illegible for scanned PDFs). **Other types** produce `{ type: "unknown" }`. Errors during processing produce `{ fileName, error }`. Passes `{ fileName, type, data|content|raw_preview }` downstream.

### Gmail Fetch Emails
Fetches up to 50 Gmail messages matching the query `has:attachment (subject:tax OR subject:invoice OR subject:receipt OR subject:statement)`, filtering for both read and unread messages received in the last 6 months (`$now.minus({months: 6}).toISO()`). Uses Gmail OAuth2 credential "Gmail account". Outputs full message objects including headers (`Subject`, `From`, `internalDate`) and binary properties for each attachment (named `attachment_0`, `attachment_1`, etc.).

### Normalize Gmail
A Code node that iterates all binary properties on each Gmail message item. It filters to six supported MIME types (PDF, CSV, JPEG, PNG, application/vnd.ms-excel, XLSX). For each qualifying attachment it constructs a standard item with fields: `id` (`gmail_<msgId>_<idx>`), `name`, `mimeType`, `createdTime` (from `msg.internalDate`), `size`, `emailId`, `emailSubject`, `emailFrom`, `source: 'gmail'`, `binaryPropertyName`. Only the specific attachment binary is forwarded (not all message binaries). Returns `[{ json: { _empty: true } }]` as a guard when no qualifying attachments exist.

### Merge
Combines the output of Normalize documents data (input 0) and Normalize Gmail (input 1) using Append mode. No configuration parameters — simply concatenates both item arrays into a single stream. Output: all Drive document items followed by all Gmail attachment items.

### Filter
Passes only items where `$json._empty` **does not exist** (operator: `notExists`, type: `string`, loose type validation). This removes the `{ _empty: true }` sentinel item emitted by Normalize Gmail when no attachments were found, preventing empty items from polluting downstream processing.

### Edit Fields: Metadata
An Edit Fields (Set) node in Manual Mapping mode that strips all upstream fields and outputs exactly 12 standardised fields per document:
- `doc_id` — from `$('Download file').item.json.id`
- `file_name` — from `$('Download file').item.binary.data.fileName`
- `file_extension` — lower-cased last segment of `fileExtension` split on `.`
- `mime_type` — from `$('Download file').item.binary.data.mimeType`
- `upload_date` — first 10 chars of `$json.createdTime` (YYYY-MM-DD) or `"unknown"`
- `source` — `$('Download file').item.binary.data.source ?? 'google_drive'`
- `size_kb` — `Math.round(Number($json.size ?? 0) / 1024)`
- `email_subject` — `$json.emailSubject ?? ''`
- `drive_link` — `$json.webViewLink ?? ''`
- `status` — literal `"pending"`
- `data` — `$json.data` (structured CSV data if applicable)
- `raw_preview` — `$json.raw_preview` (PDF byte preview if applicable)

### Loop Over Items
Processes the document stream in batches of 5 (`batchSize: 5`, `reset: false`). **Main output 0 ("done")** fires after all batches are exhausted and routes to Aggregate All Results. **Main output 1 ("loop")** carries the current batch and routes to AI Categorization. The loop-back is implemented by connecting Google Sheets: Upsert Document Row back to Loop Over Items input 0 — after each Sheets write, the loop fetches the next batch.

### OpenRouter Chat Model
An LLM sub-node connected to AI Categorization (OpenRouter) via the `ai_languageModel` port. Uses `openrouter/free` model (routes to the best currently available free model on OpenRouter). No temperature or max-token overrides — uses OpenRouter defaults. Credential: "OpenRouter account for AI Journal".

### AI Categorization (OpenRouter)
An AI Agent node (`promptType: define`) that classifies each document using the attached OpenRouter model. Receives document metadata and content via a dynamic user prompt; applies a strict system message requiring JSON-only output.

**System message instructs the model to:**
- Classify into one of: `bank_statement`, `invoice`, `receipt`, `payroll`, `tax_document`, `unknown`
- Return a precise JSON object with `category`, `confidence` (0–100), `document_type_reason`, `extracted_data` (amount, total_amount, date, vendor, account_number, transaction_count), and `flags` (is_blurry, is_incomplete, multiple_transactions)
- Never hallucinate; use `null` for missing fields; no markdown

**User prompt injects:** `file_name`, `mime_type`, `source`, and the extracted document content (`$json.raw_preview || $json.data`). Passes its output as `$json.output` to downstream nodes.

### Parse AI Response
A Code node that reads the AI agent's `$json.output` field, strips any residual markdown code fences (` ```json ` / ` ``` `), and JSON-parses the result. Maps `extracted_data.amount` → `extracted_amount`, `extracted_data.date` → `doc_period`, etc. Double-parses output to extract `flags.is_incomplete` and `flags.multiple_transactions`. On any parse failure, falls back to `{ category: 'unknown', confidence: 0, notes: 'Parse error: <message>' }`. Merges original metadata fields with AI results and sets `status: 'categorized'`, `missing_flag: false`, `processed_at: <ISO timestamp>`.

### Google Sheets: Upsert Document Row
Appends one row per processed document to **Sheet1** (`gid=0`) of spreadsheet `1OaVVOh-KFPUjvRiYyucY2dqYoaTCD-TVTKpiSNuL7sw`. Uses `mappingMode: defineBelow` with 17 manually mapped columns. Metadata columns reference `$('Loop Over Items').item.json.*` to pull from the pre-loop item; AI result columns reference `$json.*` from the current item. The output of this node connects back to Loop Over Items input 0, completing the loop. Uses credential "Google Sheets account".

**Columns written:** `doc_id`, `document_name`, `file_type`, `source`, `upload_date`, `category`, `sub_category`, `status`, `missing_flag`, `extracted_amount`, `currency`, `vendor`, `doc_period`, `confidence`, `notes`, `drive_link`, `last_updated` (`$now.format('yyyy-MM-dd')`).

### Aggregate All Results
Receives the **done output** from Loop Over Items and aggregates all items into a single output item using `aggregateAllItemData` mode, placing the array in `destinationFieldName: "all_documents"`. Output: `{ all_documents: [ ...all 28 document objects... ] }`.

### Checklist Engine (Code Node)
The core business-logic node. Reads `all_documents` and:
1. Counts documents by category into `counts` object
2. Checks each of five required categories against minimum thresholds
3. Computes `total_expenses` (sum of receipt + invoice `extracted_amount` where > 0)
4. Computes `total_income` (sum of payroll `extracted_amount` where > 0)
5. Builds a `vendor_summary` object (cumulative spend per vendor)
6. Flags any document with `extracted_amount > 10000` as an anomaly
7. Emits a single rich item with: `all_documents`, `checklist`, `missing_items`, `complete_items`, `has_missing`, `total_docs`, `total_expenses`, `total_income`, `net_position`, `category_counts`, `vendor_summary`, `anomalies`, `run_date` (today YYYY-MM-DD)

**Required document thresholds (hardcoded):**
| Category Key | Label | Minimum |
|---|---|---|
| `bank_statement` | Bank Statements (12 months) | 12 |
| `payroll` | Payroll / Salary Reports | 4 |
| `invoice` | Vendor Invoices | 1 |
| `receipt` | Expense Receipts | 1 |
| `tax_document` | Tax Forms (W-2, 1099, GST) | 1 |

### Structure data
Expands `checklist` (an array inside a single item) into multiple individual items — one per required category — so the subsequent Google Sheets Append node writes one row per category rather than trying to write an array. Maps `complete` boolean to `"YES"` / `"NO"` string. Passes through `run_date`.

### Google Sheets: Update Checklist Tab
Appends checklist status rows to **Sheet2** (`gid=666909649`) of the same spreadsheet. Columns: `run_date`, `key`, `label`, `required`, `found`, `complete` (YES/NO), `missing_count`. Each workflow run adds 5 new rows (one per required category). Uses credential "Google Sheets account".

### IF: Missing Documents?
Evaluates whether `$('Checklist Engine (Code Node)').item.json.all_documents[1].missing_flag` is boolean `true` (operator: `boolean → true → singleValue: true`, strict type validation). **True branch (main:0)** → Gmail reminder. **False branch (main:1)** → Build Briefing Prompt. See Known Limitations for a note on the condition expression.

### Gmail: Send Missing Document Reminder
Sends an HTML email to `$env.CLIENT_EMAIL` (environment variable). Subject: `⚠️ Missing Tax Documents – Action Required (<Month Year>)`. Body is a formatted HTML table listing each missing category with Required / Found / Still Needed columns. Also shows total documents collected so far. Attribution footer is disabled (`appendAttribution: false`). Uses credential "Gmail account".

### Code: Build Briefing Prompt
Constructs a multi-section structured prompt string and appends it as `briefing_prompt` to the item. Injects: run date, total docs, category breakdown, income/expenses/net position, top 10 vendors sorted by descending spend, complete checklist items, missing item details, anomalies (amounts > $10,000), and a document list (first 25 documents: name, category, vendor, amount, period). Instructs the LLM to produce exactly six sections: EXECUTIVE SUMMARY, INCOME ANALYSIS, EXPENSE ANALYSIS, DOCUMENT STATUS, FLAGS AND ANOMALIES, RECOMMENDED NEXT STEPS.

### OpenRouter Chat Model1
LLM sub-node for the Briefing Generation Agent. Uses `nvidia/nemotron-3-nano-30b-a3b:free` (Nemotron 30B free tier), `max_tokens: 2000`, `temperature: 0.2`. Same OpenRouter credential as the categorization model.

### AI Briefing Generation (OpenRouter)
Second AI Agent node. System message instructs the model to act as a senior certified accountant, use formal language, structure output in numbered sections, and avoid any JSON or meta-commentary in the output. User prompt dynamically injects `JSON.stringify($json, null, 2)` (the entire item object including `briefing_prompt`). Stores generated text in `$json.output`.

### Code: Extract Briefing Text
Reads `$json.output` from the Briefing Agent. Derives `doc_title` as `"Tax Briefing – <Month Year>"` using `Date.toLocaleDateString`. On empty content, emits error string and sets `generation_error: true`. On success, emits `{ briefing_text, doc_title, generation_error: false, char_count, generated_at }`.

### Convert to Binary
Iterates all items; for each, encodes `briefing_text` as UTF-8 base64 and attaches it as a binary property `briefing_binary` with `mimeType: 'text/plain'`, `fileName: <doc_title>.txt`, `fileExtension: 'txt'`. Skips (passes through with error flag) items with empty briefing text.

### Google Drive: Save Briefing as Text File
Uploads the `briefing_binary` binary property to Google Drive in folder `1dyFx4IlKAUcdXngaKhL23lbuboaTOKNM` (same folder as the tax documents input), naming the file `<doc_title>.txt`. Uses `inputDataFieldName: "briefing_binary"`. Uses credential "Google Drive account".

### Log Run
Appends a run summary row to the **"Log Run"** sheet (`gid=1548642444`). Maps four columns: `run_date` (from `$('Code: Build Briefing Prompt').item.json.run_date`), `missing_count` (from Build Briefing Prompt item), `briefing_generated` (mapped to the full briefing text from `$('Code: Extract Briefing Text').item.json.briefing_text`), `doc_title`. Note: `total_docs`, `total_income`, `total_expenses` columns are present in the sheet schema but marked `removed: true` — they are not populated. Uses credential "Google Sheets account".

### Gmail: Send Final Report
Sends an HTML email to the hardcoded address `storysnippents@gmail.com`. Subject: `<doc_title> – Ready for Review`. Body includes a summary list (missing documents count), a `<pre>`-formatted preview of the first 1,200 characters of `$json.briefing_generated` (note: this references a field that may not be set — see Known Limitations), and a note that the full file is saved to Google Drive.

---

## Node Configuration

### AI Categorization (OpenRouter) — Full Prompts

**System Message:**
```
You are an expert financial document classification and data extraction system.

Your job is to:
1. Classify the document into ONE category:
   - bank_statement
   - invoice
   - receipt
   - payroll
   - tax_document
   - unknown

2. Extract structured financial information.

STRICT RULES:
- Respond ONLY in valid JSON
- NO markdown
- NO explanations
- NO extra text
- If data is missing, return null
- Do NOT hallucinate values
- Keep numbers as numbers (no currency symbols)

OUTPUT FORMAT:
{
  "category": "",
  "confidence": 0-100,
  "document_type_reason": "",
  "extracted_data": {
    "amount": number|null,
    "total_amount": number|null,
    "date": "YYYY-MM-DD"|null,
    "vendor": string|null,
    "account_number": string|null,
    "transaction_count": number|null
  },
  "flags": {
    "is_blurry": false,
    "is_incomplete": false,
    "multiple_transactions": false
  }
}
```

**User Prompt:**
```
Analyze the following document and extract structured financial data.

FILE NAME: {{ $json.file_name }}
MIME TYPE: {{ $json.mime_type }}
SOURCE: {{ $json.source }}

DOCUMENT CONTENT:
{{ $json.raw_preview || $json.data }}

INSTRUCTIONS:
- Identify document type
- Extract key financial details
- If multiple transactions exist, set "multiple_transactions": true
- If content is unclear, set "is_incomplete": true

Return ONLY JSON.
```

### AI Briefing Generation (OpenRouter) — Full Prompts

**System Message:**
```
You are a senior certified accountant preparing a professional tax briefing document for a client.

Your job is to:
- Analyze structured financial data
- Summarize key financial insights
- Highlight missing or incomplete documents
- Identify anomalies or risks
- Provide clear next steps for the accountant

STRICT RULES:
- Use formal, professional language
- Structure output with numbered sections
- Keep it concise but informative
- Do NOT include JSON
- Do NOT include explanations outside the document

OUTPUT STRUCTURE:
1. Executive Summary
2. Financial Overview
3. Document Summary
4. Missing or Incomplete Documents
5. Key Observations / Anomalies
6. Recommendations / Next Steps
```

**User Prompt:**
```javascript
// Expression (the entire prompt is JavaScript template literal)
`Generate a tax briefing document based on the following processed financial data.

DATA:
${JSON.stringify($json, null, 2)}

INSTRUCTIONS:
- Aggregate totals where possible
- Identify missing categories (bank statements, payroll, invoices, etc.)
- Highlight low confidence or incomplete data
- Provide actionable recommendations

Generate the final accountant-ready briefing.`
```
> Note: This prompt injects the **entire current item JSON** (`$json`) via `JSON.stringify`. Depending on document volume, this can be very large. See Performance Considerations.

### Gmail: Send Missing Document Reminder — Email Body
```html
<h2 style="color:#333">Tax Document Collection — Missing Items</h2>
<p>Hi,</p>
<p>We reviewed your uploaded tax documents for <strong>{{ $now.format('MMMM yyyy') }}</strong>
and the following items are still missing:</p>
<table border="1" cellpadding="8" cellspacing="0" style="border-collapse:collapse;width:100%">
  <tr style="background:#f5f5f5">
    <th>Document Type</th><th>Required</th><th>Found</th><th>Still Needed</th>
  </tr>
  {{ $json.missing_items.map(i =>
    `<tr><td>${i.label}</td><td>${i.required}</td><td>${i.found}</td>
    <td><strong>${i.missing_count}</strong></td></tr>`).join('') }}
</table>
<p>Please upload the missing files to your shared Google Drive folder or reply with attachments.</p>
<p>Documents collected so far: <strong>{{ $json.total_docs }}</strong></p>
<p>Thank you,<br><em>Tax Prep Automation</em></p>
```

### Edit Fields: Metadata — Key Expressions
| Output Field | Expression |
|---|---|
| `doc_id` | `={{ $('Download file').item.json.id }}` |
| `file_name` | `={{ $('Download file').item.binary.data.fileName }}` |
| `file_extension` | `={{ $('Download file').item.binary.data.fileExtension.split('.').pop().toLowerCase() }}` |
| `upload_date` | `={{ $json.createdTime ? $json.createdTime.substring(0, 10) : 'unknown' }}` |
| `source` | `={{ $('Download file').item.binary.data.source ?? 'google_drive' }}` |
| `size_kb` | `={{ Math.round(Number($json.size ?? 0) / 1024) }}` |

### Google Sheets: Upsert Document Row — Column Mapping (abridged)
| Sheet Column | Expression |
|---|---|
| `doc_id` | `={{ $('Loop Over Items').item.json.doc_id }}` |
| `document_name` | `={{ $('Loop Over Items').item.json.file_name }}` |
| `category` | `={{ $json.category }}` |
| `extracted_amount` | `={{ $json.extracted_amount }}` |
| `confidence` | `={{ $json.confidence }}` |
| `last_updated` | `={{ $now.format('yyyy-MM-dd') }}` |

---

## Branching Logic

### IF: Missing Documents?

| Condition | Expression | Operator |
|---|---|---|
| Check missing_flag | `$('Checklist Engine (Code Node)').item.json.all_documents[1].missing_flag` | `boolean → is true` |

| Branch | Destination | Meaning |
|---|---|---|
| **TRUE (main:0)** | Gmail: Send Missing Document Reminder | At least one required category is below its minimum count |
| **FALSE (main:1)** | Code: Build Briefing Prompt | All required categories meet their minimums |

> ⚠️ **See Known Limitations** — this condition checks `all_documents[1].missing_flag` (index 1 of the raw document array), not the checklist engine's `has_missing` summary field. The condition may not behave as intended in all cases.

---

## Loop Logic

### Loop Over Items (Split In Batches)

| Parameter | Value |
|---|---|
| **Batch Size** | 5 |
| **Reset** | false |
| **Loop output** | main:1 → AI Categorization (OpenRouter) |
| **Done output** | main:0 → Aggregate All Results |

**How the loop works:**
- On each iteration, main:1 sends the next 5 document items to the AI pipeline (AI Categorization → Parse AI Response → Google Sheets: Upsert Document Row)
- The Sheets node's output connects **back** to Loop Over Items **input 0** — this is the loop-back connection visible in the canvas screenshot
- When all batches are consumed, the done output (main:0) fires once and routes to Aggregate All Results

**Loop-exit condition:** All items from Edit Fields: Metadata have been processed (n8n's SplitInBatches internally tracks the position).

---

## Variables & Expressions

### Notable expressions

| Location | Expression | Purpose |
|---|---|---|
| Gmail Fetch Emails | `$now.minus({months: 6}).toISO()` | Dynamic "received after" date — last 6 months |
| Edit Fields: Metadata | `$('Download file').item.json.id` | Cross-node reference: read Drive file ID from Download file node |
| Edit Fields: Metadata | `$json.createdTime.substring(0, 10)` | Truncate ISO timestamp to date-only string |
| Edit Fields: Metadata | `$json.size ?? 0` | Nullish coalescing guard for missing size field |
| Sheets: Upsert Doc Row | `$('Loop Over Items').item.json.doc_id` | Reference the batch item's metadata via Loop node name |
| Sheets: Upsert Doc Row | `$now.format('yyyy-MM-dd')` | Today's date in Luxon format for last_updated column |
| IF: Missing Documents? | `$('Checklist Engine (Code Node)').item.json.all_documents[1].missing_flag` | Cross-node boolean check (see Known Limitations) |
| Gmail: Send Reminder | `$now.format('MMMM yyyy')` | Month and year for email subject |
| Gmail: Send Reminder | `$json.missing_items.map(i => ...)` | Inline JS template literal to build HTML table rows |
| AI Briefing (user prompt) | `JSON.stringify($json, null, 2)` | Serialises entire item to JSON string for LLM injection |
| Log Run | `$('Code: Build Briefing Prompt').item.json.run_date` | Cross-node reference for run date |
| Drive: Save Briefing | `={{ $json.doc_title }}.txt` | Dynamic filename from Extract Briefing Text output |

### Environment Variables used

| Variable | Node | Purpose |
|---|---|---|
| `$env.CLIENT_EMAIL` | Gmail: Send Missing Document Reminder | Recipient address for the missing document reminder email |

---

## Credentials Used

| Node | Credential Name | Type | Service |
|---|---|---|---|
| GDrive List Files | Google Drive account | `googleDriveOAuth2Api` | Google Drive API v3 |
| Download file | Google Drive account | `googleDriveOAuth2Api` | Google Drive API v3 |
| Google Drive: Save Briefing as Text File | Google Drive account | `googleDriveOAuth2Api` | Google Drive API v3 |
| Gmail Fetch Emails | Gmail account | `gmailOAuth2` | Gmail API v2 |
| Gmail: Send Missing Document Reminder | Gmail account | `gmailOAuth2` | Gmail API v2 |
| Gmail: Send Final Report | Gmail account | `gmailOAuth2` | Gmail API v2 |
| Google Sheets: Upsert Document Row | Google Sheets account | `googleSheetsOAuth2Api` | Google Sheets API v4 |
| Google Sheets: Update Checklist Tab | Google Sheets account | `googleSheetsOAuth2Api` | Google Sheets API v4 |
| Log Run | Google Sheets account | `googleSheetsOAuth2Api` | Google Sheets API v4 |
| OpenRouter Chat Model | OpenRouter account for AI Journal | `openRouterApi` | OpenRouter API |
| OpenRouter Chat Model1 | OpenRouter account for AI Journal | `openRouterApi` | OpenRouter API |

> All credentials are referenced by name and internal ID only. No secret values are stored in the workflow export.

---

## Input Data

**Trigger:** No input payload — the workflow starts on schedule with an empty item.

**Drive documents:** The workflow scans folder `1dyFx4IlKAUcdXngaKhL23lbuboaTOKNM`. Files can be of any type; only PDF, CSV, TXT, JPEG, PNG, XLS, XLSX receive meaningful content extraction.

**Gmail attachments:** Messages matching `has:attachment (subject:tax OR subject:invoice OR subject:receipt OR subject:statement)` received in the last 6 months. Attachments must be PDF, CSV, JPEG, PNG, XLS, or XLSX to be processed; other types are silently skipped.

**Expected input document shape (post-normalisation):**
```json
{
  "doc_id": "1XkAbCd9Ef2GhIj",
  "file_name": "Jan_BankStatement_2024.pdf",
  "file_extension": "pdf",
  "mime_type": "application/pdf",
  "upload_date": "2024-01-15",
  "source": "google_drive",
  "size_kb": 200,
  "email_subject": "",
  "drive_link": "https://drive.google.com/file/d/.../view",
  "status": "pending",
  "data": null,
  "raw_preview": "...first 2000 bytes of PDF..."
}
```

---

## Output Data

| Output | Destination | Content |
|---|---|---|
| Document tracker rows | Google Sheets — Sheet1 | 17-column row per document: category, vendor, amount, confidence, status, etc. |
| Checklist status rows | Google Sheets — Sheet2 | 7-column row per required category: found vs required counts, complete YES/NO |
| Run log row | Google Sheets — Log Run sheet | run_date, missing_count, briefing text, doc_title |
| Missing doc reminder | Gmail → `$env.CLIENT_EMAIL` | HTML table listing unfulfilled document categories |
| Briefing `.txt` file | Google Drive (same folder as input) | Six-section formal accountant briefing as plain text |
| Final report email | Gmail → `storysnippents@gmail.com` | Briefing preview (first 1,200 chars) + Drive file reference |

---

## Data Transformation

1. **Drive binary → structured content**: Normalize documents data decodes base64 binary data and extracts text, CSV arrays, or raw byte strings depending on MIME type.
2. **Gmail message → flat attachment item**: Normalize Gmail flattens multi-attachment messages into one item per attachment, synthesising a compound `id` (`gmail_<msgId>_<idx>`).
3. **Heterogeneous sources → uniform schema**: Edit Fields: Metadata produces a consistent 12-field schema regardless of whether a document originated from Drive or Gmail.
4. **Free-text AI output → structured fields**: Parse AI Response strips markdown, JSON-parses the agent's text output, and maps nested fields (`extracted_data.amount`) to flat schema fields (`extracted_amount`).
5. **Document array → financial summary**: Checklist Engine aggregates by category, sums amounts, groups by vendor, and computes net position.
6. **Checklist array → individual rows**: Structure data explodes a single item containing an array into multiple items for Sheets append.
7. **Text string → binary file**: Convert to Binary encodes `briefing_text` as UTF-8 base64 for Drive upload.

---

## AI/LLM Integration

### Categorization Agent

| Property | Value |
|---|---|
| Node | AI Categorization (OpenRouter) |
| Model sub-node | OpenRouter Chat Model |
| Model ID | `openrouter/free` |
| Temperature | Default (not set) |
| Max tokens | Default (not set) |
| Output field | `$json.output` |
| Output format | JSON (enforced via system message) |

The agent receives document filename, MIME type, source, and extracted content. It returns a structured JSON object with category, confidence, extracted financial data, and quality flags. The categorization is based primarily on filename and limited text content (PDF content quality is low without a proper parser).

### Briefing Generation Agent

| Property | Value |
|---|---|
| Node | AI Briefing Generation (OpenRouter) |
| Model sub-node | OpenRouter Chat Model1 |
| Model ID | `nvidia/nemotron-3-nano-30b-a3b:free` |
| Temperature | 0.2 (low — produces consistent formal prose) |
| Max tokens | 2,000 |
| Output field | `$json.output` |
| Output format | Plain prose (no JSON) |

The agent receives the full structured financial summary as a JSON string in the user prompt and produces a six-section formal narrative briefing. The 0.2 temperature setting biases output towards deterministic, formal language suitable for professional documents.

---

## Error Handling

| Node | Error Handling Configured | Behaviour |
|---|---|---|
| All nodes | `continueOnFail` not set (defaults to OFF) | Any unhandled error halts the workflow execution at that node |
| Normalize documents data | `try/catch` in Code node | File parse errors produce `{ fileName, error: message }` item instead of crashing |
| Normalize Gmail | Guard return | Missing/unsupported attachments produce `{ _empty: true }` rather than nothing |
| Parse AI Response | `try/catch` + fallback object | JSON parse failures produce `{ category: 'unknown', confidence: 0 }` and continue |
| Code: Extract Briefing Text | Empty content check | Empty LLM output produces error string and `generation_error: true`; does not crash |
| Convert to Binary | Empty text guard | Empty briefing text produces `{ error: 'Empty briefing text' }` and skips binary conversion |

**No workflow-level error workflow is configured** (`settings.errorWorkflow` is not set). If any node outside a Code node's try/catch throws an unhandled exception, the execution fails silently (no alert is sent unless the n8n instance is configured to notify on failure globally).

---

## Retry Strategy

**No retry configuration** is present on any node in this workflow. The HTTP calls to OpenRouter (via the AI Agent nodes) have no `retryOnFail`, `maxTries`, or `waitBetweenTries` settings. If OpenRouter returns a 429 (rate limit) or 5xx error, the workflow will fail at that iteration with no automatic recovery.

---

## Logging

Each workflow run appends:
- Rows to Sheet1 for every processed document (document tracker)
- 5 rows to Sheet2 (checklist status, one per required category)
- 1 row to the "Log Run" sheet with run_date, missing_count, briefing text, and doc_title

n8n's own execution log also records the run (retention depends on instance configuration).

---

## Execution Order

Execution order is **v1 (connection-based)** as set in `settings.executionOrder: "v1"`. Branches from a single node execute in parallel (GDrive List Files and Gmail Fetch Emails run concurrently from Schedule Trigger). Downstream nodes wait for all required inputs before firing.

---

## File Operations

| Node | Operation | Drive / Location | Files |
|---|---|---|---|
| GDrive List Files | List / Search | My Drive › folder `1dyFx4IlKAUcdXngaKhL23lbuboaTOKNM` | All files — returns metadata only |
| Download file | Download binary | My Drive | Each file listed by GDrive List Files |
| Google Drive: Save Briefing as Text File | Upload binary | My Drive › folder `1dyFx4IlKAUcdXngaKhL23lbuboaTOKNM` | `<Tax Briefing – Month Year>.txt` |

> The briefing output is saved to the **same folder** as the input tax documents. Consider creating a separate output folder to avoid the Drive search node picking up previously generated briefings on subsequent runs.

---

## API Integrations / External Services

| Service | Node(s) | Endpoint / Operation | Auth | Purpose |
|---|---|---|---|---|
| Google Drive API v3 | GDrive List Files, Download file, Save Briefing | Drive file search & download, file upload | OAuth2 | Source document retrieval + briefing upload |
| Gmail API v2 | Gmail Fetch Emails, Send Reminder, Send Final Report | `messages.list`, `messages.get`, `messages.send` | OAuth2 | Email attachment ingestion + outbound notifications |
| Google Sheets API v4 | Upsert Doc Row, Update Checklist, Log Run | `spreadsheets.values.append` | OAuth2 | Persistent logging and checklist tracking |
| OpenRouter API | OpenRouter Chat Model, OpenRouter Chat Model1 | `POST /v1/chat/completions` | API Key | Free-tier LLM inference for categorization and briefing |

---

## Security Considerations

- **OAuth2 for all Google APIs** — credentials are stored in n8n's credential store, not in the workflow JSON. Standard OAuth2 token refresh applies.
- **OpenRouter API key** — stored as a named credential. The free tier account (`openRouterApi`) is named "OpenRouter account for AI Journal" — verify this is not a personal account shared with other workflows.
- **Hardcoded recipient email** — `storysnippents@gmail.com` is hardcoded in Gmail: Send Final Report. This should be moved to `$env.ACCOUNTANT_EMAIL` for production use.
- **Drive folder ID is hardcoded** — if the folder is shared publicly or with unintended parties, all tax documents in it are accessible. Confirm Drive folder sharing settings.
- **No webhook involved** — the schedule trigger has no inbound authentication surface.
- **AI data handling** — document content (financial data, vendor names, amounts) is sent to OpenRouter's API. Review OpenRouter's data retention and privacy policies before using with real client data. _[Security sign-off required by workflow owner]_

---

## Performance Considerations

- **Batch size of 5** — with 28 documents, this produces 6 iterations. Each iteration involves an LLM API call; total categorization time scales linearly with batch count and API response latency.
- **PDF content quality** — the current PDF extraction is raw byte string (first 2,000 chars), which is largely unreadable binary data for most PDFs. The AI will rely primarily on filename for classification. A proper PDF parser (e.g. calling a cloud OCR service via HTTP Request) would significantly improve categorization accuracy.
- **Briefing prompt size** — `JSON.stringify($json, null, 2)` injects the entire checklist engine output including all documents. For large document sets (100+ docs), this may exceed the model's context window or produce very long prompts that slow inference.
- **No parallelism in the AI loop** — documents are processed serially in batches of 5. For large document volumes, consider increasing batch size or restructuring to use parallel sub-workflows.
- **OpenRouter free tier** — free models may have rate limits, queuing delays, or intermittent availability. Production use should consider a paid tier with defined SLA.

---

## Execution Time

_[To be provided by workflow owner — requires execution data from n8n run history]_

Estimated: 2–10 minutes per run depending on number of documents and OpenRouter API latency.

---

## Resource Usage

_[To be provided by workflow owner — requires execution data]_

---

## Troubleshooting

| Symptom | Likely Cause | Investigation Steps |
|---|---|---|
| No documents processed; workflow completes immediately | Drive folder is empty or folder ID is wrong | Open GDrive List Files → check folder ID matches your target folder; verify files exist in that folder |
| All documents categorised as `unknown` | PDF extraction returning unreadable binary noise | Check `raw_preview` field in Sheet1; if it contains garbled characters, the PDF needs proper OCR |
| Parse AI Response outputs `category: 'unknown'` with notes `Parse error` | LLM returned markdown-wrapped JSON or non-JSON text | Check the raw `output` field in AI Categorization executions; consider lowering temperature |
| Gmail: Send Reminder fails | `$env.CLIENT_EMAIL` not set | Add `CLIENT_EMAIL` environment variable in n8n Settings → Environment Variables |
| Loop never reaches Aggregate All Results | Loop-back connection missing or broken | Verify Google Sheets: Upsert Document Row output connects to Loop Over Items **input 0** (not input 1) |
| `$json.briefing_generated` is undefined in Gmail: Send Final Report | Field name mismatch — Log Run writes `briefing_generated` as briefing text but Final Report references it | Check that `Code: Extract Briefing Text` → `Convert to Binary` → `Drive: Save` → `Log Run` chain completed successfully; also verify `$json.briefing_generated` vs `$json.briefing_text` field names |
| Checklist shows all items as missing even though docs were collected | IF condition checks `all_documents[1].missing_flag` not `has_missing` | See Known Limitations — the IF condition may need to be updated to `$json.has_missing` |
| Briefing text saved to Drive but email preview is empty | `$json.briefing_generated` field mismatch in Gmail: Send Final Report | The email body references `$json.briefing_generated` but the correct field from Log Run context is `briefing_generated` (the mapped briefing text); verify field propagation |
| OpenRouter call fails with 429 | Rate limit on free tier | Add a Wait node (5–10 seconds) before the AI Categorization node inside the loop |

---

## Known Limitations

1. **IF condition uses wrong field.** `IF: Missing Documents?` evaluates `$('Checklist Engine (Code Node)').item.json.all_documents[1].missing_flag` — the `missing_flag` on the **second document in the raw array** (index 1), not the summary `has_missing` boolean. This means the condition only correctly reflects document-level flags, and will behave unexpectedly if fewer than 2 documents are collected or if index 1 happens to be a document that was not flagged. The condition should reference `$('Checklist Engine (Code Node)').item.json.has_missing` instead.

2. **PDF extraction is not functional.** `Normalize documents data` reads raw PDF bytes as a UTF-8 string — for binary PDFs this produces mostly garbage. Real text extraction requires a PDF parsing library or an external OCR service. Until this is addressed, PDF classification relies entirely on the filename.

3. **Briefing and tax documents share the same Drive folder.** `Google Drive: Save Briefing as Text File` uploads the briefing to the same folder ID that `GDrive List Files` scans. On subsequent runs, the briefing `.txt` file will be included in the Drive search results and fed into the AI categorization pipeline, likely producing incorrect classifications.

4. **Log Run column mismatch.** The Log Run Sheets schema includes `total_docs`, `total_income`, `total_expenses` columns but they are marked `removed: true` and are not populated — these fields exist in the schema but will always be empty. The `briefing_generated` column is mapped to the full briefing text (potentially thousands of characters in a single cell), rather than a boolean flag as the name implies.

5. **Hardcoded accountant email.** `Gmail: Send Final Report` sends to `storysnippents@gmail.com` — this is a literal value in the JSON, not an environment variable. Deploying this workflow for another client requires editing the node directly.

6. **No retry on LLM calls.** If OpenRouter returns an error, the current batch's documents will not be categorized and an empty or error row will be appended to the Sheets tracker.

7. **No deduplication.** Each Monday run scans all files in the Drive folder and all matching emails since 6 months ago. Previously processed documents will be re-categorized and new duplicate rows appended to Sheet1 on every run.

8. **Checklist thresholds are hardcoded.** The five required categories and their minimums (12 bank statements, 4 payroll, 1 invoice, 1 receipt, 1 tax doc) are constants in the Checklist Engine Code node. Different clients with different requirements require code changes.

---

## Assumptions

1. The n8n server timezone matches the intended weekly Monday 09:00 schedule timezone for the client — no explicit timezone override is configured.
2. `$env.CLIENT_EMAIL` is set as an environment variable on the n8n instance before activation.
3. The Google Drive folder `1dyFx4IlKAUcdXngaKhL23lbuboaTOKNM` is the intended source of tax documents (cached display name is "n8n" — this may be a test/dev folder).
4. `storysnippents@gmail.com` is a test email address used during development; a production accountant email will replace it.
5. OpenRouter's `openrouter/free` model routes to a capable instruction-following model at runtime — free model availability can change without notice.
6. The Google Sheets spreadsheet ID `1OaVVOh-KFPUjvRiYyucY2dqYoaTCD-TVTKpiSNuL7sw` is already created with the correct tabs (Sheet1, Sheet2, Log Run) and column headers matching the defined schema.
7. The workflow is currently **inactive** — it has never executed against a live instance. The node configuration reflects design intent; some field reference expressions (especially those relying on upstream node names with `$('Node Name')`) may need adjustment based on actual runtime data shapes.

---

## Testing & Validation

_[To be provided by workflow owner — no test results available at documentation time]_

Recommended test approach:
- Manually trigger (add a Manual Trigger node temporarily) with 3–5 sample documents of varied types in the Drive folder
- Verify Sheet1 rows contain expected `category` and `extracted_amount` values
- Test with intentionally missing documents to confirm the Gmail reminder fires
- Confirm briefing `.txt` file appears in Drive and email is received

---

## Version History / Change Log

| Version | Date | Author | Changes |
|---|---|---|---|
| v1.0 | _[To be provided]_ | _[To be provided]_ | Initial workflow build |

---

## References

- OpenRouter model catalogue: https://openrouter.ai/models
- n8n AI Agent node docs: https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent/
- Google Drive API v3 reference: https://developers.google.com/drive/api/v3/reference
- Gmail API reference: https://developers.google.com/gmail/api/reference/rest
- n8n SplitInBatches (Loop Over Items) docs: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.splitinbatches/

---

## Maintenance Notes

_[To be provided by workflow owner]_

Suggested items:
- Review and update the OpenRouter model IDs quarterly — free models are added/removed without notice
- Add a separate output Google Drive folder for briefing files to prevent re-ingestion
- Replace hardcoded `storysnippents@gmail.com` with `$env.ACCOUNTANT_EMAIL` before production use
- Fix the IF condition to reference `has_missing` instead of `all_documents[1].missing_flag`
- Implement deduplication logic (e.g. check Sheet1 for existing `doc_id` before appending)
- Consider integrating a PDF text extraction service (e.g. Adobe PDF Extract API or Google Document AI) to replace the current raw-byte PDF preview
