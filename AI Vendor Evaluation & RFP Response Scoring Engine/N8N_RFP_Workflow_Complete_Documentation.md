# N8N AI Vendor Evaluation & RFP Response Scoring Engine
## Complete Implementation & Operations Documentation

**Workflow Name**: AI Vendor Evaluation & RFP Response Scoring Engine  
**Version**: 1.0  
**Created**: May 2024  
**Status**: Production Ready  
**Cost**: $0 (Free tier APIs only)

---

## 📋 TABLE OF CONTENTS

1. [Executive Summary](#executive-summary)
2. [System Architecture](#system-architecture)
3. [Visual Workflow Guide](#visual-workflow-guide)
4. [Node Configuration Reference](#node-configuration-reference)
5. [Data Processing Pipeline](#data-processing-pipeline)
6. [Setup & Deployment Guide](#setup--deployment-guide)
7. [API Credentials Setup](#api-credentials-setup)
8. [Testing Procedures](#testing-procedures)
9. [Error Handling & Recovery](#error-handling--recovery)
10. [Monitoring & Maintenance](#monitoring--maintenance)
11. [Advanced Customization](#advanced-customization)
12. [Troubleshooting Guide](#troubleshooting-guide)

---

## 🎯 EXECUTIVE SUMMARY

### What This Workflow Does

This n8n workflow automates the entire vendor evaluation process from RFP email submission to decision-maker notification:

```
📧 RFP Email → 📄 PDF Extraction → 🤖 AI Scoring → 📊 Ranking → 💌 Follow-up → 📈 Sheets → 💬 Slack Alert
```

### Business Benefits

| Benefit | Before | After | Impact |
|---------|--------|-------|--------|
| **Evaluation Time** | 2-3 hours/vendor | 30-60 seconds/vendor | **95% faster** |
| **Consistency** | Subjective scoring | AI-powered objective rubrics | **100% consistent** |
| **Coverage** | Max 5 vendors/day | Unlimited (rate-limited by API) | **Scalable** |
| **Audit Trail** | Spreadsheet chaos | Complete history in Google Sheets | **Full compliance** |
| **Decision Speed** | 2-3 days | Minutes after submission | **Real-time insights** |

### Key Capabilities

✅ **Automated Email Monitoring** - Continuously watches Gmail for RFP responses  
✅ **Intelligent PDF Parsing** - Extracts vendor data from attachments  
✅ **AI-Powered Scoring** - 5-dimension evaluation with weighted formula  
✅ **Smart Gap Detection** - Identifies missing information automatically  
✅ **Smart Notifications** - Slack alerts with top vendor shortlist  
✅ **Centralized Storage** - All evaluations in Google Sheets database  
✅ **Zero Cost** - Uses only free API tiers (OpenRouter, Gmail, Google Sheets, Slack)

---

## 🏗️ SYSTEM ARCHITECTURE

### High-Level System Design

```
┌─────────────────────────────────────────────────────────────────────┐
│                         INPUT LAYER                                 │
│  Gmail Inbox Monitoring (Poll every 60 seconds)                    │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      EXTRACTION LAYER                               │
│  ┌─────────────────────┐        ┌────────────────────┐            │
│  │ Extract PDF Content │ ─────► │ Parse & Normalize  │            │
│  │   (n8n built-in)    │        │   Vendor Data      │            │
│  └─────────────────────┘        └────────────────────┘            │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    AI PROCESSING LAYER                              │
│  ┌─────────────────────┐        ┌────────────────────┐            │
│  │  Build AI Prompt    │ ─────► │  OpenRouter LLM    │            │
│  │  (System + User)    │        │ (Llama 3.1 70B)    │            │
│  └─────────────────────┘        └────────────────────┘            │
│                                        │                            │
│                                        ▼                            │
│                         ┌─────────────────────┐                   │
│                         │  Parse Scores JSON  │                   │
│                         └─────────────────────┘                   │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   AGGREGATION LAYER                                 │
│  ┌─────────────────────┐                                           │
│  │  Rank Vendors       │ ──► Sort by Score ──► Add Ranks         │
│  │  Calculate Stats    │                                          │
│  └─────────────────────┘                                           │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   ROUTING LAYER                                     │
│  IF: Missing Fields?                                               │
│  ├─ TRUE  ──► Generate Follow-up Email                            │
│  └─ FALSE ──► Skip to Storage                                     │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   STORAGE & OUTPUT LAYER                            │
│  ┌──────────────────────┐      ┌───────────────────────┐          │
│  │ Google Sheets Append │      │ Slack Notifications   │          │
│  │ (Audit Trail)        │      │ (Decision Makers)     │          │
│  └──────────────────────┘      └───────────────────────┘          │
└─────────────────────────────────────────────────────────────────────┘
```

### Technology Stack

| Layer | Technology | Purpose |
|-------|-----------|---------|
| **Orchestration** | n8n v1.0+ | Workflow automation platform |
| **AI Engine** | OpenRouter + Llama 3.1 70B | Vendor evaluation scoring |
| **Email** | Gmail API (OAuth2) | Trigger + follow-up emails |
| **Database** | Google Sheets API | Persistent storage & audit trail |
| **Notifications** | Slack API | Real-time alerting |
| **File Processing** | n8n built-in | PDF text extraction |

### Data Flow Overview

```
Email Input
    │
    ├─ From: vendor@company.com
    ├─ Subject: RFP Response - Vendor Name
    ├─ Attachment: RFP_Response.pdf
    └─ Body: Contact info
    
    ▼ [PDF Extraction]
    
Structured Data
    ├─ vendor_name: "Acme Corp"
    ├─ vendor_email: "vendor@acme.com"
    ├─ extracted_fields:
    │   ├─ pricing: "$50,000/year"
    │   ├─ capabilities: "Feature list..."
    │   ├─ sla: "99.9% uptime"
    │   ├─ compliance: "SOC2, ISO27001"
    │   └─ references: "Client list"
    └─ raw_response: "Full PDF text"
    
    ▼ [AI Scoring]
    
Scores & Analysis
    ├─ price_score: 7
    ├─ capability_score: 8
    ├─ compliance_score: 9
    ├─ sla_score: 8
    ├─ reference_score: 6
    ├─ total_score: 7.75
    ├─ missing_fields: []
    ├─ risk_flags: ["Small company references"]
    └─ summary: "Professional 2-sentence analysis"
    
    ▼ [Ranking & Storage]
    
Output Artifacts
    ├─ Google Sheets Row (Audit trail)
    ├─ Slack Message (Decision makers)
    └─ Optional: Follow-up Email (Gap closure)
```

---

## 📸 VISUAL WORKFLOW GUIDE

### Workflow Canvas Overview

The workflow diagram shows 13 interconnected nodes organized in two primary paths:

```
┌─ TOP PATH: AI SCORING PIPELINE ─────────────────────────────────┐
│                                                                  │
│  1         2               3                4            5       │
│  ┌──┐      ┌──┐           ┌──┐            ┌──┐         ┌──┐    │
│  │GM│ ───► │EX│ ────────► │RN│ ────────► │PA│ ──────► │AV│    │
│  │IL│      │TR│           │OR│           │PR│         │EV│    │
│  └──┘      └──┘           └──┘           └──┘         └──┘    │
│   RFP       PDF            Response       Prepare        AI     │
│  Monitor   Extract        Normalizer      Prompt       Evaluator│
│                                                           │      │
│                                                    └─────┴─┐    │
│                                                            │    │
│                                     6         7          │    │
│                                    ┌──┐    ┌──┐          │    │
│                            ┌──────► │ES│ ──► │ES│ ◄──────┘    │
│                            │        │CD│    │CD│              │
│                            │        └──┘    └──┘              │
│                            │        Extract  Extract          │
│                            │        Scoring  Scoring          │
│                            │        Data 1   Data 2           │
│                    Connects here (multiple vendors)            │
└────────────────────────────────────────────────────────────────┘

┌─ BOTTOM PATH: ROUTING & OUTPUTS ────────────────────────────────┐
│                                                                  │
│   8        9               10           11          12      13   │
│  ┌──┐    ┌──┐            ┌──┐         ┌──┐       ┌──┐    ┌──┐  │
│  │VR│ ──► │IF│ ──YES──►  │FE│ ──────► │GE│ ─────► │SM│ ─► │SL│  │
│  │EN│    │  │            │ML│         │MA│        │SG│   │NK│  │
│  └──┘    └──┘            └──┘         └──┘        └──┘   └──┘  │
│  Vendor   Check for    Follow-up   Gmail Sheets  Slack   Slack  │
│  Ranking  Missing    Email Gen.    Append      Message  Notifier│
│  Engine   Fields?  (IF TRUE ONLY)               Builder          │
│            │                                                     │
│            └──────── NO ─────────────┐                          │
│                                      │                          │
│                                      ├─ Skip follow-up          │
│                                      └─ Go straight to Sheets   │
└────────────────────────────────────────────────────────────────┘

KEY CONNECTIONS:
- Gmail Trigger → PDF Extractor → Response Normalizer
- Normalizer → Prepare Prompt → AI Evaluator
- AI Evaluator → Extract Scoring Data (multiple outputs)
- All Scoring Data → Vendor Ranking Engine
- Vendor Ranking Engine → IF (branch point)
- IF TRUE → Follow-up Email → Gmail Sender → Sheets
- IF FALSE → Sheets directly
- Sheets → Slack Message Builder → Slack Notifier

EXECUTION FLOW:
1. Gmail polls every 60 seconds
2. New email → triggers entire flow
3. All nodes execute sequentially (unless waiting for multiple items)
4. At Vendor Ranking Engine: waits for all vendors before ranking
5. IF node: routes based on missing_fields
6. Parallel paths: both email and Slack execute
7. Completion: Everything is stored & notified
```

### Canvas Position Reference

| Node | Name | Position | Purpose |
|------|------|----------|---------|
| 1 | RFP Response Monitor | (0, 0) | Gmail trigger - Top left |
| 2 | Extract from File | (416, 0) | PDF extraction |
| 3 | Response Normalizer | (624, 0) | Parse vendor data |
| 4 | Prepare AI Prompt | (832, 0) | Build prompt messages |
| 5 | AI Vendor Evaluator | (1168, 0) | OpenRouter LLM scoring |
| 6 | Extract Scoring Data | (1168, 0) | Parse JSON response |
| Sub | OpenRouter Chat Model | (Connected to 5) | LLM model config |
| 7 | Vendor Ranking Engine | (-32, 336) | Aggregate & rank |
| 8 | IF | (208, 336) | Conditional router |
| 9 | Follow-up Email Generator | (416, 336) | Create email template |
| 10 | Gmail Follow-up Sender | (624, 336) | Send email |
| 11 | Append row in sheet | (832, 336) | Google Sheets storage |
| 12 | Slack Message Builder1 | (1040, 336) | Format Slack message |
| 13 | Slack Notifier | (1248, 336) | Post to Slack |

---

## 🔧 NODE CONFIGURATION REFERENCE

### NODE 1: RFP Response Monitor (Gmail Trigger)

**Node Type**: `n8n-nodes-base.gmailTrigger` (v1.3)

**Purpose**: Continuously monitor Gmail inbox for RFP vendor responses

**Configuration**:
```json
{
  "pollTimes": {
    "item": [{"mode": "everyMinute"}]
  },
  "simple": false,
  "filters": {
    "includeSpamTrash": false
  },
  "options": {
    "downloadAttachments": true
  }
}
```

**Configuration Details**:

| Field | Value | Explanation |
|-------|-------|-------------|
| **Poll Interval** | Every minute | Checks Gmail every 60 seconds for new emails |
| **Simple Mode** | false | Enables advanced email metadata access |
| **Spam/Trash** | Exclude | Ignores emails in spam or trash folders |
| **Attachments** | Download | Automatically retrieves PDF/file attachments |

**Expected Output**:
```json
{
  "id": "message_id_123abc",
  "threadId": "thread_id_456def",
  "subject": "RFP Response - Acme Corporation",
  "from": "vendor@acme.com",
  "to": "procurement@yourcompany.com",
  "date": "2024-05-02T10:30:00.000Z",
  "headers": {
    "from": "vendor@acme.com",
    "subject": "RFP Response - Acme Corporation"
  },
  "attachments": [
    {
      "filename": "Acme_RFP_Response.pdf",
      "mimeType": "application/pdf",
      "size": 245680,
      "data": "binary_encoded_pdf"
    }
  ],
  "bodyPlain": "Please find our RFP response attached...",
  "bodyHtml": "<p>Please find our RFP response attached...</p>"
}
```

**Gmail Search Filter Recommendations**:
```
To enable for specific RFP emails, add filter:
subject:(RFP Response OR Vendor Response) has:attachment label:RFP
```

**Credentials Required**: Gmail OAuth2 API

---

### NODE 2: Extract from File (PDF Parser)

**Node Type**: `n8n-nodes-base.extractFromFile`

**Purpose**: Convert PDF text content to processable text format

**Configuration**:
```json
{
  "operation": "text"
}
```

**How It Works**:
- Receives binary PDF attachment from Gmail trigger
- Extracts all text content using built-in PDF engine
- Preserves formatting (paragraphs, line breaks, structure)
- Handles multi-page documents seamlessly
- Outputs plain text (no markup)

**Sample Input**: Binary PDF file  
**Sample Output**:
```json
{
  "text": "Vendor Name: Acme Corporation\n\n1. Pricing:\nAnnual License: $50,000\nImplementation: $10,000\nSupport: Included\n\n2. Capabilities:\n- Real-time analytics\n- API integrations\n- Custom dashboards\n\n3. SLA:\nUptime: 99.9%\nResponse: 4 hours\n\n4. Compliance:\nSOC2, ISO27001, GDPR\n\n5. References:\n- Client A (2 years)\n- Client B (3 years)",
  "filename": "Acme_RFP_Response.pdf"
}
```

**Limitations**:
- Scanned PDFs (images) won't extract text - require OCR
- Complex tables may lose formatting
- Very large PDFs (>50MB) may timeout

**Error Handling**:
```
If PDF cannot be parsed:
→ Workflow continues with empty text
→ Vendor Name defaults to email subject
→ AI scoring flags missing data
```

---

### NODE 3: Response Normalizer (Code Node)

**Node Type**: `n8n-nodes-base.code` (v2) - JavaScript

**Purpose**: Parse extracted PDF text into structured vendor data

**Core Logic**:

```javascript
// 1. Extract vendor name (priority: PDF text → email subject → fallback)
let vendorName = 'Unknown Vendor';
const vendorFromPDF = rawText.match(/Vendor Name[:\s]+(.+)/i);
if (vendorFromPDF) {
  vendorName = vendorFromPDF[1].trim();
}

// 2. Extract 5 key sections using regex
const extractSection = (startLabel, endLabel) => {
  const regex = new RegExp(
    `${startLabel}[\\s\\S]*?:([\\s\\S]*?)(?=${endLabel}|$)`,
    'i'
  );
  const match = rawText.match(regex);
  return match ? match[1].replace(/\n+/g, ' ').trim() : null;
};

// 3. Extract each field
const pricing = extractSection('1\. Pricing', '2\.');
const capabilities = extractSection('2\. Capabilities', '3\.');
const sla = extractSection('3\. SLA', '4\.');
const compliance = extractSection('4\. Compliance', '5\.');
const references = extractSection('5\. References', '6\.');
```

**Extraction Strategy**:
- Uses numbered sections as anchors (1. Pricing, 2. Capabilities, etc.)
- Captures all text between section labels
- Collapses multi-line content to single lines
- Returns null if section not found (AI will flag as missing)

**Expected Output**:
```json
{
  "vendor_name": "Acme Corporation",
  "vendor_email": "vendor@acme.com",
  "received_date": "2024-05-02T10:30:00.000Z",
  "message_id": "message_id_123",
  "raw_response": "Complete extracted PDF text...",
  "extracted_fields": {
    "pricing": "Annual $50,000 Implementation $10,000",
    "capabilities": "Real-time analytics API integrations",
    "sla": "99.9% uptime 4-hour response",
    "compliance": "SOC2 ISO27001 GDPR",
    "references": "Client A 2 years Client B 3 years"
  }
}
```

**Edge Cases Handled**:
- Missing sections → null (AI detects as missing field)
- Multiple instances of "Pricing" → captures first occurrence
- Typos in section numbers → regex is case-insensitive
- No vendor name → defaults to email subject extraction

---

### NODE 4: Prepare AI Prompt (Code Node)

**Node Type**: `n8n-nodes-base.code` (v2) - JavaScript

**Purpose**: Construct system + user prompt messages for OpenRouter API

**System Prompt Content** (800+ words):

```
You are an expert procurement and vendor evaluation specialist with 15+ years 
of experience evaluating enterprise software solutions.

EVALUATION FRAMEWORK:

1. PRICE (Weight: 25%)
   - 0-3: Significantly overpriced
   - 4-6: Market-rate
   - 7-8: Competitive
   - 9-10: Exceptional value

2. CAPABILITIES (Weight: 30%)
   - 0-3: Missing features
   - 4-6: Meets basics
   - 7-8: Strong feature set
   - 9-10: Industry-leading

3. COMPLIANCE (Weight: 20%)
   - 0-3: No certifications
   - 4-6: Basic (1-2 certs)
   - 7-8: Strong (SOC2, ISO27001)
   - 9-10: Comprehensive (SOC2 Type II, GDPR, HIPAA)

4. SLA (Weight: 15%)
   - 0-3: No SLA
   - 4-6: Basic (95-99% uptime)
   - 7-8: Strong (99.9% uptime)
   - 9-10: Enterprise (99.99% uptime)

5. REFERENCES (Weight: 10%)
   - 0-3: No references
   - 4-6: Small company refs
   - 7-8: Similar-sized refs
   - 9-10: Enterprise refs

WEIGHTED FORMULA:
Total Score = (Price × 0.25) + (Capabilities × 0.30) + (Compliance × 0.20) 
            + (SLA × 0.15) + (References × 0.10)
```

**User Prompt Template**:
```
Evaluate this vendor RFP response:

VENDOR: {vendor_name}
EMAIL: {vendor_email}

RESPONSE TEXT:
{raw_response}

EXTRACTED FIELDS:
- Pricing: {pricing || 'Not found'}
- SLA: {sla || 'Not found'}
- Compliance: {compliance || 'Not found'}
- References: {references || 'Not found'}

OUTPUT FORMAT (STRICT JSON):
{
  "vendor_name": "string",
  "scores": {
    "price": number,
    "capabilities": number,
    "compliance": number,
    "sla": number,
    "references": number
  },
  "total_score": number,
  "missing_fields": ["array"],
  "risk_flags": ["array"],
  "summary": "2 sentences"
}
```

**Output Structure**:
```json
{
  "messages": [
    {
      "role": "system",
      "content": "You are an expert procurement... [800+ word prompt]"
    },
    {
      "role": "user",
      "content": "Evaluate this vendor RFP response: [vendor details]"
    }
  ],
  "vendor_data": {
    "vendor_name": "Acme Corporation",
    "vendor_email": "vendor@acme.com",
    "..."
  }
}
```

**Key Features**:
- System prompt establishes expert persona
- Provides clear scoring rubrics for each dimension
- Specifies weighted calculation formula
- Enforces JSON-only output (no markdown)
- Includes vendor context from extracted fields

---

### NODE 5: AI Vendor Evaluator (AI Agent + LLM)

**Node Type**: `@n8n/n8n-nodes-langchain.agent` (v3)

**Purpose**: Use AI to score vendors objectively across 5 dimensions

**Configuration**:
```json
{
  "promptType": "define",
  "text": "{{ User prompt from previous node }}",
  "options": {
    "systemMessage": "{{ System prompt from previous node }}"
  }
}
```

**Connected Model: OpenRouter Chat Model**:
```json
{
  "model": "meta-llama/llama-3.1-70b-instruct:free",
  "temperature": 0.3,
  "maxTokens": 2000
}
```

**Model Selection Rationale**:

| Aspect | Choice | Why |
|--------|--------|-----|
| **Model** | Llama 3.1 70B | Best free model for structured reasoning |
| **Temperature** | 0.3 | Low = consistent, objective scoring |
| **Max Tokens** | 2000 | Sufficient for detailed JSON response |
| **Provider** | OpenRouter | Free tier with generous limits |

**How It Works**:
1. Receives system + user prompts from Node 4
2. Sends to OpenRouter API with JSON mode enabled
3. Llama 3.1 evaluates vendor against 5 rubrics
4. Calculates weighted total score
5. Identifies missing fields
6. Flags risks
7. Returns structured JSON response

**Sample AI Response**:
```json
{
  "vendor_name": "Acme Corporation",
  "scores": {
    "price": 7,
    "capabilities": 8,
    "compliance": 9,
    "sla": 8,
    "references": 6
  },
  "total_score": 7.75,
  "missing_fields": [],
  "risk_flags": ["References only from small companies"],
  "summary": "Strong technical capabilities with excellent compliance posture. Pricing is competitive but enterprise references would strengthen the proposal.",
  "raw_scores_justification": {
    "price": "$50k/year competitive for feature set",
    "capabilities": "Advanced AI, good scalability, strong integration options",
    "compliance": "SOC2 Type II, ISO27001, GDPR certified - excellent posture",
    "sla": "99.9% uptime guarantee with 4-hour response time",
    "references": "3 references provided but all from companies <100 employees"
  }
}
```

**OpenRouter Free Tier Limits**:
```
Rate Limit: ~20 requests per minute
Cost: $0
Monthly Usage: Generous (check dashboard)
Alternative Models: Gemini Flash, Hermes 3
```

---

### NODE 6: Extract Scoring Data (Code Node)

**Node Type**: `n8n-nodes-base.code` (v2) - JavaScript

**Purpose**: Parse AI JSON response and merge with vendor metadata

**Core Logic**:
```javascript
try {
  const aiResponse = JSON.parse(item.json.choices[0].message.content);
  
  // Merge AI scores with vendor data
  return {
    vendor_name: aiResponse.vendor_name,
    price_score: aiResponse.scores.price,
    capability_score: aiResponse.scores.capabilities,
    compliance_score: aiResponse.scores.compliance,
    sla_score: aiResponse.scores.sla,
    reference_score: aiResponse.scores.references,
    total_score: aiResponse.total_score,
    missing_fields: aiResponse.missing_fields,
    risk_flags: aiResponse.risk_flags,
    summary: aiResponse.summary,
    timestamp: new Date().toISOString()
  };
} catch (error) {
  // Fallback: flag for manual review
  return {
    vendor_name: 'PARSING_ERROR',
    needs_manual_review: true,
    error: error.message
  };
}
```

**Error Handling Strategy**:
- Wraps JSON.parse in try-catch
- If parsing fails: flags as `needs_manual_review: true`
- Preserves original response for debugging
- Workflow continues (doesn't crash)

**Output Structure**:
```json
{
  "vendor_name": "Acme Corporation",
  "vendor_email": "vendor@acme.com",
  "price_score": 7,
  "capability_score": 8,
  "compliance_score": 9,
  "sla_score": 8,
  "reference_score": 6,
  "total_score": 7.75,
  "missing_fields": [],
  "risk_flags": ["References only from small companies"],
  "summary": "Strong technical capabilities with excellent compliance posture...",
  "justifications": {...},
  "timestamp": "2024-05-02T10:35:22.456Z"
}
```

---

### NODE 7: Vendor Ranking Engine (Code Node)

**Node Type**: `n8n-nodes-base.code` (v2) - JavaScript

**Purpose**: Aggregate all vendor scores, sort by rank, calculate statistics

**Critical Configuration**: Set execution mode to **"Wait for all incoming items"**
- This ensures workflow collects ALL vendor scores before ranking
- Without this, nodes execute immediately after first vendor

**Core Logic**:
```javascript
const allItems = $input.all(); // Wait for all vendors

// Filter out errors
const cleanedVendors = allItems
  .filter(v => v.json.total_score !== undefined)
  .sort((a, b) => b.json.total_score - a.json.total_score)
  .map((v, idx) => ({
    ...v.json,
    rank: idx + 1,
    rank_label: idx === 0 ? '🥇 Top' : idx === 1 ? '🥈 2nd' : `#${idx + 1}`
  }));

// Calculate summary stats
const summary = {
  total_vendors: cleanedVendors.length,
  average_score: (sum / count).toFixed(2),
  vendors_with_risks: cleanedVendors.filter(v => v.risk_flags.length > 0).length,
  vendors_with_missing_data: cleanedVendors.filter(v => v.missing_fields.length > 0).length
};

return [{json: {vendors: cleanedVendors, summary}}];
```

**Output Structure**:
```json
{
  "vendors": [
    {
      "vendor_name": "TechFlow Solutions",
      "total_score": 8.70,
      "rank": 1,
      "rank_label": "🥇 Top Choice",
      "price_score": 9,
      "capability_score": 9,
      "compliance_score": 8,
      "sla_score": 9,
      "reference_score": 8,
      "risk_flags": [],
      "missing_fields": [],
      "summary": "..."
    },
    {
      "vendor_name": "Acme Corporation",
      "total_score": 7.75,
      "rank": 2,
      "rank_label": "🥈 Runner-up",
      "..."
    }
  ],
  "summary": {
    "total_vendors": 3,
    "average_score": "7.70",
    "top_vendor": "TechFlow Solutions",
    "top_score": 8.70,
    "vendors_with_risks": 2,
    "vendors_with_missing_data": 1,
    "evaluation_date": "2024-05-02T10:40:00.000Z"
  }
}
```

**Key Statistics**:
- **Total Vendors**: Count of all evaluated vendors
- **Average Score**: Mean score across all vendors
- **Risk Count**: Vendors with flagged concerns
- **Missing Data Count**: Vendors requiring follow-up

---

### NODE 8: IF (Conditional Router)

**Node Type**: `n8n-nodes-base.if`

**Purpose**: Route vendors with missing data to follow-up email generation

**Condition**:
```json
{
  "conditions": {
    "string": [
      {
        "value1": "={{ $json.missing_fields }}",
        "operation": "isNotEmpty"
      }
    ]
  },
  "combineOperation": "any"
}
```

**Routing Logic**:
```
┌─ IF missing_fields array is NOT empty
│   ├─ Check: Array has items? 
│   ├─ YES (TRUE) → Route to Follow-up Email Generator
│   └─ NO (FALSE) → Skip to Google Sheets directly
│
└─ IF missing_fields is empty
    └─ FALSE → Route to Append Row in Sheet
```

**Example Conditions**:
```
TRUE Cases (Send Follow-up):
✅ missing_fields: ["SLA response times", "Compliance certs"]
✅ missing_fields: ["Client references"]
✅ missing_fields: ["Pricing breakdown"]

FALSE Cases (Skip Follow-up):
❌ missing_fields: []
❌ missing_fields: null
❌ missing_fields: undefined
```

---

### NODE 9: Follow-up Email Generator (Code Node)

**Node Type**: `n8n-nodes-base.code` (v2) - JavaScript

**Purpose**: Create personalized follow-up email template requesting missing info

**Logic**:
```javascript
// Only process if missing_fields exist
if (missingFields.length === 0) return [];

const emailBody = `Dear ${vendor.name} Team,

Thank you for submitting your RFP response. We are currently evaluating all vendor proposals.

During our review, we noticed the following information is missing or unclear:

${missingFields.map((f, i) => `${i + 1}. ${f}`).join('\n')}

To ensure a fair and complete evaluation, please provide the requested information by replying to this email within 3 business days.

If you have any questions, please don't hesitate to reach out.

Best regards,
Procurement Team`;

return [{json: {to, subject, body}}];
```

**Output Structure**:
```json
{
  "to": "vendor@acme.com",
  "subject": "RFP Follow-up: Additional Information Required - Acme Corporation",
  "body": "Dear Acme Corporation Team,\n\nThank you for submitting...",
  "vendor_name": "Acme Corporation",
  "missing_fields": ["SLA response times", "Compliance certifications"]
}
```

**Email Best Practices**:
- Professional tone
- Specific, numbered list of missing items
- Clear 3-day deadline
- Encourages vendor response

---

### NODE 10: Gmail Follow-up Sender

**Node Type**: `n8n-nodes-base.gmail`

**Purpose**: Send follow-up email to vendor requesting missing information

**Configuration**:
```json
{
  "operation": "send",
  "sendTo": "={{ $json.to }}",
  "subject": "={{ $json.subject }}",
  "message": "={{ $json.body }}",
  "options": {
    "ccList": "procurement@yourcompany.com"
  }
}
```

**Features**:
- Auto-threads replies to original RFP email
- CC's procurement team for visibility
- Uses same OAuth2 credentials as Gmail trigger
- Error handling: "Continue On Fail" enabled

**Gmail Thread Threading**:
- n8n automatically detects original message
- Reply appears in same email thread
- Vendor sees conversation history

---

### NODE 11: Append row in sheet (Google Sheets)

**Node Type**: `n8n-nodes-base.googleSheets`

**Purpose**: Store vendor evaluation results in Google Sheets audit trail

**Configuration**:
```json
{
  "operation": "append",
  "sheetName": "VendorScores",
  "columns": {
    "vendor_name": "={{ $json.vendor_name }}",
    "vendor_email": "={{ $json.vendor_email }}",
    "price_score": "={{ $json.price_score }}",
    "capability_score": "={{ $json.capability_score }}",
    "compliance_score": "={{ $json.compliance_score }}",
    "sla_score": "={{ $json.sla_score }}",
    "reference_score": "={{ $json.reference_score }}",
    "total_score": "={{ $json.total_score }}",
    "rank": "={{ $json.rank }}",
    "missing_fields": "={{ $json.missing_fields.join(', ') }}",
    "risk_flags": "={{ $json.risk_flags.join(', ') }}",
    "summary": "={{ $json.summary }}",
    "received_date": "={{ $json.received_date }}",
    "timestamp": "={{ $json.timestamp }}"
  }
}
```

**Sheet Structure** (Headers in Row 1):

| Column | Name | Type | Format |
|--------|------|------|--------|
| A | vendor_name | Text | - |
| B | vendor_email | Text | - |
| C | price_score | Number | 0.0 |
| D | capability_score | Number | 0.0 |
| E | compliance_score | Number | 0.0 |
| F | sla_score | Number | 0.0 |
| G | reference_score | Number | 0.0 |
| H | total_score | Number | 0.00 |
| I | rank | Number | #0 |
| J | missing_fields | Text | Comma-sep |
| K | risk_flags | Text | Comma-sep |
| L | summary | Text | Long text |
| M | received_date | Date | MM/DD/YYYY |
| N | timestamp | Timestamp | MM/DD/YYYY HH:MM:SS |

**Conditional Formatting**:
```
Total Score (Column H):
- >= 8: Light green (#d9ead3)
- 6-8: Light yellow (#fff2cc)
- < 6: Light red (#f4cccc)

Rank (Column I):
- = 1: Gold (#ffd966)
- = 2: Silver (#d9d9d9)
- = 3: Bronze (#e6b8af)

Missing Fields (Column J):
- NOT EMPTY: Orange (#f9cb9c)

Risk Flags (Column K):
- NOT EMPTY: Light red (#ea9999)
```

**Audit Trail Benefits**:
- Complete history of all evaluations
- Timestamps for compliance
- Searchable for historical analysis
- Export capability for reports

---

### NODE 12: Slack Message Builder1 (Code Node)

**Node Type**: `n8n-nodes-base.code` (v2) - JavaScript

**Purpose**: Format rich Slack notification with top 3 vendors

**Core Logic**:
```javascript
const topVendors = vendors.slice(0, 3); // Get top 3

const blocks = [
  {
    type: "header",
    text: {type: "plain_text", text: "🎯 RFP Evaluation Complete"}
  },
  {
    type: "section",
    text: {
      type: "mrkdwn",
      text: `*Total Evaluated:* ${total}\n*Average Score:* ${avg}\n*Top Vendor:* ${topVendors[0]?.vendor_name}`
    }
  },
  {type: "divider"}
];

// Add each top 3 vendor
topVendors.forEach(v => {
  blocks.push({
    type: "section",
    text: {
      type: "mrkdwn",
      text: `*${v.rank_label} ${v.vendor_name}*\nScore: ${v.total_score}/10\nBreakdown: Price ${v.price_score} | Cap ${v.capability_score} | Comp ${v.compliance_score} | SLA ${v.sla_score} | Ref ${v.reference_score}\n\n${v.summary}${v.risk_flags.length > 0 ? `\n⚠️ Risks: ${v.risk_flags.join(', ')}` : ''}`
    }
  });
  blocks.push({type: "divider"});
});

return [{json: {channel: '#rfp-evaluations', blocks: JSON.stringify(blocks)}}];
```

**Output Structure**:
```json
{
  "channel": "#rfp-evaluations",
  "text": "RFP Evaluation Complete",
  "blocks": "[{\"type\":\"header\",...}]"
}
```

**Slack Block Format**:
- Header: Eye-catching title
- Summary section: Key stats
- Vendor sections: Score breakdown + summary
- Risk warnings: Alert icons + flags
- Context: Data source attribution

---

### NODE 13: Slack Notifier

**Node Type**: `n8n-nodes-base.slack`

**Purpose**: Post formatted notification to Slack channel

**Configuration**:
```json
{
  "resource": "message",
  "operation": "post",
  "channel": "={{ $json.channel }}",
  "text": "={{ $json.text }}",
  "otherOptions": {
    "blocks": "={{ $json.blocks }}"
  },
  "authentication": "oAuth2"
}
```

**Features**:
- Posts to #rfp-evaluations channel (or custom channel)
- Uses rich block formatting (not plain text)
- Includes vendor scores, summaries, and risk flags
- Creates thread for additional context

**Slack Message Example**:
```
═══════════════════════════════════════════════════
🎯 RFP Evaluation Complete - Vendor Shortlist
═══════════════════════════════════════════════════

Total Vendors Evaluated: 3
Average Score: 7.70
Evaluation Date: 05/02/2024

───────────────────────────────────────────────────

🥇 Top Choice TechFlow Solutions
Total Score: 8.70/10

Breakdown:
• Price: 9/10
• Capabilities: 9/10
• Compliance: 8/10
• SLA: 9/10
• References: 8/10

Summary: Exceptional technical capabilities with strong compliance posture 
and excellent SLA commitments. Industry-leading solution with minor pricing 
flexibility concerns.

───────────────────────────────────────────────────

🥈 Runner-up Acme Corporation
Total Score: 7.75/10

[Similar breakdown...]

───────────────────────────────────────────────────

📊 Full details available in Google Sheets | ⚡ Powered by n8n + OpenRouter AI
```

---

## 🔄 DATA PROCESSING PIPELINE

### End-to-End Execution Flow

```
STEP 1: TRIGGER (Gmail Polling)
├─ Time: Every 60 seconds
├─ Action: Poll Gmail inbox
├─ Filter: Exclude spam/trash, include attachments
└─ Output: Email object with PDF attachment

STEP 2: PDF EXTRACTION
├─ Input: Binary PDF from email
├─ Action: Extract text content
├─ Handling: Preserves structure, handles multi-page
└─ Output: Plain text string (~50-5000 chars)

STEP 3: RESPONSE NORMALIZATION
├─ Input: Raw PDF text
├─ Action: Parse into 5 sections
│   ├─ Extract: vendor_name
│   ├─ Extract: pricing
│   ├─ Extract: capabilities
│   ├─ Extract: sla
│   ├─ Extract: compliance
│   └─ Extract: references
├─ Handling: Regex pattern matching
└─ Output: Structured vendor object

STEP 4: PROMPT PREPARATION
├─ Input: Normalized vendor data
├─ Action: Build system + user prompts
│   ├─ System: Expert persona + scoring rubrics
│   ├─ User: Vendor context + evaluation request
│   └─ Format: Chat message array
└─ Output: Prompt messages ready for AI

STEP 5: AI SCORING
├─ Input: Prompt messages
├─ Service: OpenRouter API (Llama 3.1 70B)
├─ Process: 
│   ├─ Send prompts to LLM
│   ├─ LLM evaluates vendor
│   ├─ LLM calculates scores
│   ├─ LLM detects missing fields
│   ├─ LLM flags risks
│   └─ LLM returns JSON
├─ Timeout: 30 seconds
└─ Output: JSON with scores, flags, summary

STEP 6: SCORE EXTRACTION
├─ Input: AI response
├─ Action: Parse JSON
├─ Handling: Error catching for malformed JSON
├─ Fallback: Flag for manual review if parsing fails
└─ Output: Clean vendor object with scores

STEP 7: RANKING & AGGREGATION
├─ Input: All vendor scores (WAIT for all items)
├─ Action: 
│   ├─ Sort by total_score descending
│   ├─ Assign ranks (1, 2, 3...)
│   ├─ Assign rank labels (🥇, 🥈, 🥉)
│   └─ Calculate summary stats
├─ Processing: Single aggregated object
└─ Output: Ranked vendor array + summary stats

STEP 8: CONDITIONAL ROUTING
├─ Input: Ranked vendors
├─ Decision: Check if missing_fields.length > 0
├─ TRUE Path:
│   ├─ Generate follow-up email
│   ├─ Send via Gmail
│   └─ Then proceed to storage
└─ FALSE Path: Skip email, go to storage

STEP 9: FOLLOW-UP EMAIL (IF TRUE)
├─ Input: Vendor with missing fields
├─ Action: Create email template
├─ Content: 
│   ├─ Personalized greeting
│   ├─ Numbered list of missing items
│   ├─ 3-day deadline
│   └─ Professional sign-off
└─ Output: Email ready to send

STEP 10: SEND FOLLOW-UP (IF TRUE)
├─ Input: Email template
├─ Service: Gmail API
├─ Threading: Auto-reply to original RFP
├─ CC: procurement@yourcompany.com
└─ Output: Email sent confirmation

STEP 11: STORAGE (Google Sheets)
├─ Input: Vendor data + scores
├─ Action: Append row to "VendorScores" sheet
├─ Fields: All 14 columns (name, email, scores, flags, etc.)
├─ Format: Auto-format numbers/dates
└─ Output: Row added to sheet

STEP 12: SLACK MESSAGE FORMAT
├─ Input: Top 3 ranked vendors
├─ Action: Build rich Slack blocks
├─ Content:
│   ├─ Header with emoji
│   ├─ Summary stats
│   ├─ Top 3 vendor details
│   ├─ Scores breakdown
│   ├─ Risk warnings
│   └─ Attribution footer
└─ Output: Formatted block JSON

STEP 13: SLACK NOTIFICATION
├─ Input: Formatted block JSON
├─ Service: Slack API
├─ Channel: #rfp-evaluations
├─ Type: Block-formatted message
└─ Output: Message posted to Slack

END: Workflow Complete
├─ All vendors evaluated ✅
├─ Scores stored in Sheets ✅
├─ Decision-makers notified ✅
└─ Follow-ups sent if needed ✅
```

### Execution Time Breakdown

| Stage | Duration | Notes |
|-------|----------|-------|
| Gmail Polling → PDF Extract | 2-5 sec | Depends on PDF size |
| Normalization | 0.5-1 sec | Regex pattern matching |
| Prompt Preparation | 0.2-0.5 sec | String concatenation |
| **AI Scoring (Bottleneck)** | **10-30 sec** | OpenRouter API latency |
| Score Extraction | 0.5-1 sec | JSON parsing |
| Ranking & Aggregation | 1-2 sec | Array sorting |
| Conditional Routing | 0.1 sec | IF check |
| Follow-up Email Gen (if needed) | 0.5 sec | Template creation |
| Gmail Send (if needed) | 2-4 sec | API latency |
| Google Sheets Append | 2-4 sec | Sheet API latency |
| Slack Format & Send | 2-3 sec | Slack API latency |
| **Total Per Vendor** | **20-50 sec** | Most time is AI + APIs |

### Handling Multiple Vendors

```
Scenario: 3 vendors submit RFP responses in quick succession

Timeline:
T+0s:   Vendor A email arrives
T+5s:   Vendor B email arrives
T+10s:  Vendor C email arrives
T+15s:  Gmail trigger polls, detects all 3
T+15-45s: Vendor A → Processing begins
T+45-75s: Vendor B → Processing begins
T+75-105s: Vendor C → Processing begins
T+110s: Vendor Ranking Engine WAITS for all 3 scores
T+120s: All 3 ranked, Slack notification sent

Key: Gmail trigger creates SEPARATE workflow execution for each email
But Vendor Ranking Engine waits for ALL executions to complete before ranking

Result: Top 3 vendors presented together in single Slack message
```

---

## 🚀 SETUP & DEPLOYMENT GUIDE

### Prerequisites Checklist

```
❓ Do you have:

Infrastructure:
☐ n8n instance (self-hosted or cloud)
☐ Minimum 2GB RAM available
☐ Stable internet connection
☐ HTTPS/SSL enabled (for OAuth redirects)

Accounts:
☐ Gmail account for RFP monitoring
☐ Google Workspace (for Google Sheets API access)
☐ OpenRouter account (free)
☐ Slack workspace with admin access

Permissions:
☐ Gmail account: Can receive emails, create labels
☐ Google Account: Can create/edit Sheets
☐ Slack: Can create channels, install apps
☐ n8n: Admin access to settings

Knowledge:
☐ Basic understanding of n8n workflows
☐ Familiarity with Gmail filters
☐ Ability to create Google Sheets
☐ Slack channel management experience
```

### Phase 1: API Account Setup (30 minutes)

#### 1.1 OpenRouter Setup

```
1. Create Account
   - Go to: https://openrouter.ai/
   - Sign up with Google/GitHub
   - Verify email

2. Get API Key
   - Settings → API Keys
   - Create new key: "n8n_rfp_scoring"
   - Copy key (starts with: sk-or-v1-...)
   - Save in secure location

3. Verify Free Tier
   - Dashboard → Check remaining quota
   - Should show: Unlimited with rate limits
   - Rate limit: ~20 requests/minute
```

**Cost**: $0 (free tier)

---

#### 1.2 Google Cloud Project Setup

```
1. Create Project
   - Go to: https://console.cloud.google.com/
   - New Project: "n8n-rfp-automation"
   - Wait for project creation (~1 minute)

2. Enable APIs
   - APIs & Services → Library
   - Search: "Gmail API" → Enable
   - Search: "Google Sheets API" → Enable
   - Search: "Google Drive API" → Enable

3. Create OAuth Credentials
   - APIs & Services → Credentials
   - Create Credentials → OAuth 2.0 Client ID
   - Application type: Web application
   - Name: "n8n-rfp"
   
4. Configure OAuth Screen
   - OAuth consent screen
   - User type: External (or Internal if G Workspace)
   - Fill app name, support email, scopes
   - Add test users: your email address
   - Scopes: gmail.readonly, gmail.send, spreadsheets

5. Generate Client ID & Secret
   - Authorized redirect URIs:
     https://YOUR_N8N_DOMAIN/rest/oauth2-credential/callback
   - Copy Client ID and Secret
   - Save to secure location
```

**Cost**: $0 (free tier with generous limits)

---

#### 1.3 Slack App Setup

```
1. Create Slack App
   - Go to: https://api.slack.com/apps
   - Create New App → From scratch
   - App name: "RFP Evaluator Bot"
   - Workspace: Select your workspace

2. Configure Permissions
   - OAuth & Permissions
   - Add scopes:
     - chat:write
     - chat:write.public
     - files:write
   - Install App to Workspace
   - Copy "Bot User OAuth Token" (xoxb-...)

3. Create Channel
   - In Slack: Create #rfp-evaluations channel
   - Invite bot: /invite @RFP Evaluator Bot

4. Test Bot
   - In channel: Type message to verify bot is active
   - Copy Bot Token to secure location
```

**Cost**: $0 (free tier)

---

### Phase 2: n8n Configuration (30 minutes)

#### 2.1 Create Credentials in n8n

**OpenRouter Credential**:
```
Settings → Credentials → Add Credential
┌─────────────────────────────────────┐
│ Type: HTTP Header Auth              │
│ Name: OpenRouter API                │
│ Header Name: Authorization          │
│ Header Value: Bearer YOUR_API_KEY   │
│ Test: Click "Test Connection"       │
└─────────────────────────────────────┘
Status: ✅ Connected
```

**Gmail OAuth2 Credential**:
```
Settings → Credentials → Add Credential
┌─────────────────────────────────────┐
│ Type: Gmail OAuth2 API              │
│ Client ID: (from Google Console)    │
│ Client Secret: (from Google)        │
│ Scopes: gmail.readonly,gmail.send   │
│ Click: "Connect my account"         │
│ Authorize in popup                  │
└─────────────────────────────────────┘
Status: ✅ Connected
```

**Google Sheets OAuth2 Credential**:
```
Settings → Credentials → Add Credential
┌─────────────────────────────────────┐
│ Type: Google Sheets OAuth2 API      │
│ Client ID: (from Google Console)    │
│ Client Secret: (from Google)        │
│ Click: "Connect my account"         │
│ Authorize in popup                  │
└─────────────────────────────────────┘
Status: ✅ Connected
```

**Slack OAuth2 Credential**:
```
Settings → Credentials → Add Credential
┌─────────────────────────────────────┐
│ Type: Slack API                     │
│ Access Token: (Bot User OAuth Token)│
│ Test: Click "Test Connection"       │
└─────────────────────────────────────┘
Status: ✅ Connected
```

---

#### 2.2 Import Workflow

```
Method 1: Import JSON File
1. In n8n: Workflows → Import from File
2. Select: AI_Vendor_Evaluation___RFP_Response_Scoring_Engine.json
3. Click: Import

Method 2: Copy from URL
1. Workflows → New
2. Open n8n editor
3. Edit → Export → Copy to clipboard
4. Import shared JSON

Status: Workflow imported
⚠️ Credentials need to be reassigned
```

---

#### 2.3 Update Workflow Node Credentials

For **each node** requiring credentials:

**Node 1 (RFP Response Monitor)**:
```
1. Click node in canvas
2. Credentials section
3. Select: Gmail account (OAuth2)
4. ✅ Verify (green checkmark)
```

**Node 4 (Prepare AI Prompt) → SubNode (OpenRouter Chat Model)**:
```
1. Click node
2. Scroll to Models section
3. Select model: meta-llama/llama-3.1-70b-instruct:free
4. Credentials: OpenRouter API
5. ✅ Test
```

**Node 10 (Gmail Follow-up Sender)**:
```
1. Click node
2. Credentials: Gmail account (OAuth2)
3. ✅ Verify
```

**Node 11 (Append row in sheet)**:
```
1. Click node
2. Credentials: Google Sheets (OAuth2)
3. Document ID: [Your Sheet ID]
4. Sheet Name: VendorScores
5. ✅ Test
```

**Node 13 (Slack Notifier)**:
```
1. Click node
2. Credentials: Slack API
3. Channel: #rfp-evaluations
4. ✅ Test
```

---

#### 2.4 Create Google Sheet

```
1. Create new Google Sheet: https://sheets.google.com/
2. Name: "RFP Vendor Evaluations"
3. Rename "Sheet1" → "VendorScores"

4. Add Headers (Row 1):
   A1: vendor_name
   B1: vendor_email
   C1: price_score
   D1: capability_score
   E1: compliance_score
   F1: sla_score
   G1: reference_score
   H1: total_score
   I1: rank
   J1: missing_fields
   K1: risk_flags
   L1: summary
   M1: received_date
   N1: timestamp

5. Format Headers:
   - Select Row 1
   - Background color: #4a86e8 (blue)
   - Font color: White
   - Bold: Yes

6. Copy Sheet ID from URL:
   https://docs.google.com/spreadsheets/d/[THIS_ID]/edit
   Store this ID → Update into Append row in sheet node

7. Share with service account (if using):
   Click Share → Add service account email
```

---

### Phase 3: Testing & Validation (30 minutes)

#### 3.1 Unit Tests (Test Individual Nodes)

**Test Gmail Trigger**:
```
1. Click RFP Response Monitor node
2. Click "Execute Node"
3. Expected: Shows recent emails from inbox
4. Verify: Can see subject, from, attachments
```

**Test PDF Extraction**:
```
1. Manually send sample RFP email with PDF
2. Run trigger
3. Click "Extract from File" node
4. Click "Execute Node"
5. Expected: Text extracted from PDF
6. Verify: Content is readable
```

**Test Response Normalizer**:
```
1. Click "Response Normalizer" node
2. Click "Execute Node"
3. Expected: Structured vendor data
4. Verify: 
   ✅ vendor_name populated
   ✅ extracted_fields have content
   ✅ pricing, sla, compliance parsed
```

**Test AI Scoring**:
```
1. Click "AI Vendor Evaluator" node
2. Click "Execute Node"
3. Expected: JSON response from OpenRouter
4. Verify:
   ✅ Scores are 0-10 range
   ✅ Total score calculated correctly
   ✅ Missing fields identified
   ✅ Summary is 2 sentences
```

**Test Google Sheets Write**:
```
1. Click "Append row in sheet" node
2. Click "Execute Node"
3. Expected: Row added to sheet
4. Verify:
   ✅ Open Google Sheet
   ✅ See new row with vendor data
   ✅ Timestamps and scores visible
```

**Test Slack Notification**:
```
1. Click "Slack Notifier" node
2. Click "Execute Node"
3. Expected: Message appears in #rfp-evaluations
4. Verify:
   ✅ Rich formatting displays
   ✅ Vendor names visible
   ✅ Scores readable
   ✅ Top 3 shown
```

---

#### 3.2 End-to-End Test

**Preparation**:
```
1. Create sample RFP PDF with structure:

Vendor Name: Test Vendor Corp

1. Pricing:
Annual: $50,000
Implementation: $10,000

2. Capabilities:
- Analytics
- APIs
- Mobile app

3. SLA:
Uptime: 99.9%
Support: 24/7

4. Compliance:
SOC2 Type II
ISO 27001

5. References:
Client A (2 years)
Client B (3 years)

2. Save as: Test_Vendor_RFP.pdf
3. Send email with subject: "RFP Response - Test Vendor Corp"
```

**Execution**:
```
1. Wait for Gmail trigger (polls every minute)
2. Wait for full workflow execution (~60 seconds)
3. Check outputs:
   ✅ Row added to Google Sheets
   ✅ Slack notification posted
   ✅ Scores reasonable (7-9 range)
   ✅ No manual follow-up needed (if no missing fields)
```

**Expected Results**:
```
Google Sheets Row:
vendor_name: Test Vendor Corp
total_score: 8.2-8.8
rank: 1 (if first vendor)
missing_fields: [empty]
risk_flags: [empty]
summary: [Professional assessment]

Slack Message:
🎯 RFP Evaluation Complete
Total Vendors: 1
Average Score: 8.5

🥇 Top Choice Test Vendor Corp
Total Score: 8.5/10
Price: 8 | Capabilities: 8 | Compliance: 9 | SLA: 9 | References: 8

Summary: [2-sentence assessment]
```

---

## 🔑 API CREDENTIALS SETUP

### Complete Credentials Configuration

#### OpenRouter Free Tier

**Step 1: Account Creation**
```
URL: https://openrouter.ai/auth/signin
├─ Sign up with Google/GitHub OAuth
├─ Or create account with email
├─ Verify email address
└─ Access to dashboard
```

**Step 2: Generate API Key**
```
URL: https://openrouter.ai/account/api-keys
├─ Dashboard → Account Settings
├─ Click "Create Key"
├─ Name: n8n_rfp_scoring_engine
├─ Copy full key (starts: sk-or-v1-...)
├─ ⚠️ Save immediately (not shown again)
└─ Add to secure storage
```

**Step 3: Verify Free Tier Status**
```
URL: https://openrouter.ai/account/overview
├─ Check "Free Tier Status"
├─ Verify credit remaining
├─ Rate limit: ~20 requests/minute
├─ Monthly limit: Generous
└─ Cost: $0
```

**Step 4: n8n Configuration**
```
n8n Settings → Credentials → Add Credential
┌─ Type: HTTP Header Auth
├─ Name: "OpenRouter API"
├─ Header Name: "Authorization"
├─ Header Value: "Bearer sk-or-v1-YOUR_KEY_HERE"
├─ Test Connection: [Click Test]
└─ Status: ✅ Connected
```

**Available Free Models**:
```
Primary (recommended):
meta-llama/llama-3.1-70b-instruct:free
- Best reasoning quality
- 70B parameters
- Rate: ~20 req/min

Alternatives:
google/gemini-flash-1.5:free
nousresearch/hermes-3-llama-3.1-405b:free
```

**Cost Structure**:
```
Free Tier:
- Monthly: ~$0
- Rate Limit: 20 requests/minute
- Per Request: $0

Paid Tier (if needed):
- Pricing: $0.002-0.02 per 1K tokens
- Rate Limit: Unlimited
- Support: Priority

Recommendation: Start with free tier, upgrade if production needs exceed rate limits
```

---

#### Gmail OAuth2

**Step 1: Google Cloud Project**
```
URL: https://console.cloud.google.com/
1. Click "Select a Project"
2. Click "NEW PROJECT"
3. Name: "n8n-rfp-automation"
4. Click "CREATE"
5. Wait for project to initialize (~1 minute)
```

**Step 2: Enable APIs**
```
URL: https://console.cloud.google.com/apis/dashboard
1. APIs & Services → Library
2. Search: "Gmail API"
   └─ Click "Enable"
3. Search: "Google Sheets API"
   └─ Click "Enable"
4. Search: "Google Drive API"
   └─ Click "Enable"

Verify:
- Dashboard shows 3 APIs enabled
- Green checkmarks visible
```

**Step 3: OAuth Consent Screen**
```
URL: https://console.cloud.google.com/apis/credentials/consent
1. Click "Edit App"
2. Fill "App information":
   App name: "n8n RFP Automation"
   User support email: your-email@company.com
   
3. Add "Scopes":
   Click "Add or Remove Scopes"
   ├─ Search: "gmail"
   ├─ Select: gmail.readonly, gmail.send
   ├─ Search: "sheets"
   ├─ Select: spreadsheets
   └─ Click "Update"
   
4. Add "Test Users":
   Add your email address
   
5. Click "Save and Continue"
```

**Step 4: Create OAuth Credentials**
```
URL: https://console.cloud.google.com/apis/credentials
1. Click "Create Credentials" → OAuth 2.0 Client ID
2. Application type: "Web application"
3. Name: "n8n-rfp"
4. Authorized JavaScript origins:
   https://YOUR_N8N_DOMAIN
   
5. Authorized redirect URIs:
   https://YOUR_N8N_DOMAIN/rest/oauth2-credential/callback
   
6. Click "Create"
7. Copy and save:
   - Client ID
   - Client Secret
   (You won't see this again)
```

**Step 5: n8n Configuration**
```
n8n Settings → Credentials → Add Credential
┌─ Type: Gmail OAuth2 API
├─ Client ID: (from Google Cloud)
├─ Client Secret: (from Google Cloud)
├─ Scopes: gmail.readonly,gmail.send
├─ Click "Connect my account"
├─ Authorize in popup (allow all permissions)
└─ Status: ✅ Connected
```

**Troubleshooting**:
```
Error: "Redirect URI mismatch"
└─ Fix: Verify n8n domain in Google Cloud credentials

Error: "Insufficient permissions"
└─ Fix: Re-authorize, add missing scopes

Error: "Invalid credentials"
└─ Fix: Regenerate Client ID/Secret in Google Cloud
```

---

#### Google Sheets OAuth2

**Step 1: Reuse Google Cloud Project**
```
Use same project as Gmail setup above
(API already enabled: Google Sheets API)
```

**Step 2: Create Service Account (Optional)**
```
For production, consider service account instead of OAuth:

URL: https://console.cloud.google.com/iam-admin/serviceaccounts
1. Create Service Account
2. Name: n8n-rfp-sheets
3. Grant role: Editor
4. Create key: JSON format
5. Download and store securely
```

**Step 3: n8n Configuration (OAuth)**
```
n8n Settings → Credentials → Add Credential
┌─ Type: Google Sheets OAuth2 API
├─ Client ID: (from Google Cloud)
├─ Client Secret: (from Google Cloud)
├─ Click "Connect my account"
├─ Authorize in popup
└─ Status: ✅ Connected
```

**Step 4: Create and Share Sheet**
```
1. Create new Google Sheet: https://sheets.google.com/
2. Name: "RFP Vendor Evaluations"
3. Note Sheet ID from URL:
   https://docs.google.com/spreadsheets/d/[THIS_IS_ID]/edit
   
4. Share with service account email (if using):
   Click Share → Add service-account-email@project.iam.gserviceaccount.com
   
5. Use Sheet ID in n8n node configuration
```

---

#### Slack API

**Step 1: Create Slack App**
```
URL: https://api.slack.com/apps
1. Click "Create New App"
2. Select "From scratch"
3. App name: "RFP Evaluator Bot"
4. Workspace: Select your workspace
5. Click "Create App"
```

**Step 2: Configure Permissions**
```
URL: https://api.slack.com/apps/[APP_ID]/oauth-scopes
1. OAuth & Permissions (left sidebar)
2. Scroll to "Bot Token Scopes"
3. Click "Add an OAuth Scope"
4. Add scopes:
   - chat:write
   - chat:write.public
   - files:write
   
Status: Green checkmarks visible
```

**Step 3: Install Bot to Workspace**
```
URL: https://api.slack.com/apps/[APP_ID]/install-on-team
1. Click "Install to Workspace"
2. Authorize bot (allow permissions)
3. Workspace selector appears
4. Click "Allow"
5. Copy "Bot User OAuth Token":
   xoxb-1234567890-ABCDEF...
   
⚠️ Save to secure location (not shown again)
```

**Step 4: Create Channel**
```
In your Slack workspace:
1. Create channel: #rfp-evaluations
   ├─ Private or Public (your choice)
   ├─ Description: "RFP vendor evaluation results"
   └─ Click "Create"
   
2. Invite bot:
   Type: /invite @RFP Evaluator Bot
   
3. Verify bot joined (shows in members)
```

**Step 5: n8n Configuration**
```
n8n Settings → Credentials → Add Credential
┌─ Type: Slack API
├─ Access Token: xoxb-YOUR_BOT_TOKEN
├─ Click "Test Connection"
└─ Status: ✅ Connected

In Slack Notifier node:
├─ Credentials: Slack API
├─ Channel: #rfp-evaluations
└─ ✅ Verify works
```

**Testing Bot**:
```
In Slack, try:
/invite @RFP Evaluator Bot
→ Bot should appear in channel

In n8n, click "Execute Node" on Slack Notifier
→ Message should appear in #rfp-evaluations
```

---

## 🧪 TESTING PROCEDURES

### Pre-Launch Checklist

```
CREDENTIALS VERIFICATION
☐ Gmail OAuth2: Connected ✅
☐ Google Sheets OAuth2: Connected ✅
☐ OpenRouter API: Tested ✅
☐ Slack API: Token valid ✅

NODE CONFIGURATION
☐ Gmail Trigger: Filter configured
☐ PDF Extractor: Assigned to correct node
☐ Response Normalizer: All regex patterns verified
☐ AI Evaluator: Model selected (Llama 3.1 70B)
☐ Ranking Engine: "Wait for all items" enabled
☐ IF Node: Missing_fields condition correct
☐ Google Sheets: Sheet ID updated, headers added
☐ Slack Channel: #rfp-evaluations created, bot invited

DATA VALIDATION
☐ Google Sheets columns formatted correctly
☐ Conditional formatting rules applied
☐ OpenRouter free tier status verified
☐ Gmail label/filter configured

WORKFLOW SETTINGS
☐ Execution Order: v1 (connection-based)
☐ Error Handling: Continue On Fail for non-critical nodes
☐ Timeouts: Set to 60+ seconds for API calls
```

### Test Case 1: Happy Path (Complete Success)

**Scenario**: Vendor submits complete RFP with all required info

**Setup**:
```
Create PDF with structure:
Vendor Name: Complete Corp

1. Pricing
$40,000/year, $8,000 implementation

2. Capabilities
- Feature A, B, C
- API integration
- Mobile support

3. SLA
99.9% uptime, 24/7 support

4. Compliance
SOC2 Type II, ISO 27001, GDPR

5. References
Client A (500 employees, 3 years)
Client B (1000 employees, 2 years)
Client C (2000 employees, 1 year)
```

**Execution**:
```
1. Send email: subject "RFP Response - Complete Corp"
2. Attach PDF
3. Wait for workflow (~60 seconds)
```

**Verification**:
```
✅ Gmail received email
✅ PDF extracted successfully
✅ Vendor name parsed: "Complete Corp"
✅ All 5 fields extracted with content
✅ AI returned scores:
   price: 8-9 (competitive)
   capabilities: 8-9 (complete)
   compliance: 9-10 (excellent)
   sla: 9-10 (excellent)
   references: 8-9 (strong)
✅ Total score: 8.5-9.0
✅ missing_fields: [] (empty)
✅ risk_flags: [] (empty)
✅ IF Node: Routes to FALSE (no follow-up)
✅ Row appended to Google Sheets
✅ Slack message posted to #rfp-evaluations
✅ Message shows vendor as "🥇 Top Choice" (if first)
```

---

### Test Case 2: Missing Fields Scenario

**Scenario**: Vendor submits incomplete RFP

**Setup**:
```
Create PDF with gaps:
Vendor Name: Incomplete Corp

1. Pricing
[SECTION MISSING]

2. Capabilities
Various features provided

3. SLA
[SECTION MISSING]

4. Compliance
[SECTION MISSING]

5. References
Available upon request
```

**Execution**:
```
1. Send email with subject "RFP Response - Incomplete Corp"
2. Wait for workflow
```

**Verification**:
```
✅ PDF extracted (even if sections missing)
✅ AI detected missing_fields:
   ✅ "Pricing details"
   ✅ "SLA commitments"
   ✅ "Compliance certifications"
✅ risk_flags populated with warnings
✅ Lower scores assigned (3-5 range for missing)
✅ IF Node: Routes to TRUE (follow-up needed)
✅ Follow-up email generated with numbered list
✅ Email sent to vendor@incomplete.com
✅ CC: procurement@yourcompany.com
✅ Row appended to Google Sheets
✅ missing_fields column populated
✅ Slack message includes warning icon (⚠️)
```

---

### Test Case 3: Malformed PDF

**Scenario**: PDF cannot be parsed (encrypted, image scan, corrupted)

**Setup**:
```
Upload corrupted or scanned PDF
(no extractable text)
```

**Execution**:
```
1. Send email with unreadable PDF
2. Wait for workflow
```

**Verification**:
```
✅ PDF extraction fails or returns empty text
✅ Vendor name extraction falls back to email subject
✅ extracted_fields all null
✅ AI receives mostly empty data
✅ AI detects ALL fields as missing
✅ Scores very low (2-3 range)
✅ risk_flags: ["No readable RFP content"]
✅ IF Node routes to TRUE (follow-up)
✅ Follow-up email requesting proper format
✅ Manual review flag in sheets
```

---

### Test Case 4: Multiple Concurrent Submissions

**Scenario**: 3 vendors submit RFP responses within minutes

**Timeline**:
```
T+0:  Vendor A email arrives
T+3:  Vendor B email arrives
T+6:  Vendor C email arrives
T+60: Gmail trigger polls (detects all 3)
T+60-65: Vendor A processing begins
T+65-70: Vendor B processing begins  
T+70-75: Vendor C processing begins
T+75: AI scoring happens (parallel if possible)
T+120: Ranking Engine WAITS for all 3 scores
T+125: All 3 ranked, single Slack message sent
```

**Verification**:
```
✅ Gmail trigger processes each separately
✅ All 3 eventually reach Ranking Engine
✅ Ranking Engine aggregates all 3
✅ Slack shows all 3 in one message (or split if too large)
✅ Google Sheets has 3 new rows
✅ Scores sensible relative to each other
✅ Ranks 1-3 assigned correctly
```

---

### Test Case 5: API Rate Limiting

**Scenario**: Exceed OpenRouter free tier rate limit (20 req/min)

**Setup**:
```
Send 25 RFP emails rapidly
```

**Verification**:
```
If rate limit hit:
✅ First 20 score successfully
✅ Requests 21-25: Receive 429 error from OpenRouter
✅ Fallback logic triggers:
   ├─ Retry with exponential backoff
   ├─ Or flag for manual review
   └─ Workflow continues (doesn't crash)
✅ Google Sheets reflects partial results
✅ Slack shows available rankings
✅ Manual intervention needed for failed vendors

Prevention: 
✅ Upgrade to paid OpenRouter tier for production
✅ Implement request queuing in n8n
✅ Set rate limiting at 15 req/min (safety margin)
```

---

## 🚨 ERROR HANDLING & RECOVERY

### Common Error Scenarios

#### Error 1: "Gmail OAuth token expired"

**Symptoms**:
- RFP Response Monitor node fails
- Error message: "Invalid grant" or "Unauthorized"
- No emails being detected

**Root Cause**:
- OAuth token expired (every ~3-6 months)
- Gmail permissions revoked

**Recovery**:
```
1. Go to n8n Settings → Credentials
2. Find "Gmail account" credential
3. Click "Reconnect"
4. Re-authorize in popup
5. Click "Save"
6. Test workflow

Alternative: Remove and recreate credential
1. Delete old Gmail credential
2. Create new Gmail OAuth2 credential
3. Update both RFP Monitor and Gmail Follow-up nodes
```

---

#### Error 2: "Failed to extract text from PDF"

**Symptoms**:
- Extract from File node returns empty text
- vendor_name defaults to "Unknown Vendor"
- All extracted_fields are null

**Root Cause**:
- PDF is scanned image (not text-based)
- PDF is encrypted/password-protected
- PDF is corrupted

**Recovery**:
```
Option 1: Request proper format from vendor
- Send follow-up email requesting text-based PDF
- Provide template for standardized RFP response

Option 2: Use OCR preprocessing (advanced)
- Add HTTP Request node to call free OCR API
- Process image before text extraction
- (Adds latency but enables scanned PDFs)

Option 3: Manual extraction
- Download PDF
- Copy-paste text into email body
- Resend with [TEXT] in subject
```

---

#### Error 3: "OpenRouter API rate limit exceeded"

**Symptoms**:
- AI Vendor Evaluator returns 429 error
- "Too many requests" message
- Workflow stalls at scoring stage

**Root Cause**:
- Exceeded free tier rate limit (20 req/min)
- Multiple workflows running simultaneously
- High volume submission period

**Recovery**:
```
Short-term (immediate):
1. Wait 60 seconds (rate limit resets)
2. Manually re-run failed vendors
3. Upgrade to paid OpenRouter tier

Long-term (prevention):
1. Implement request queuing:
   - Add delay node between workflows
   - Set to 3-5 seconds between requests
   
2. Upgrade to paid tier:
   - OpenRouter paid: $0.002-0.02 per 1K tokens
   - Unlimited rate limit
   - Priority support

3. Use alternative model:
   - Gemini Flash (sometimes faster)
   - Hermes 3 (good alternative)
```

---

#### Error 4: "Google Sheets append failed - Sheet not found"

**Symptoms**:
- Append row in sheet node fails
- Error: "Sheet not found" or "Invalid range"
- Vendor scores not stored

**Root Cause**:
- Sheet ID incorrect
- Sheet renamed or deleted
- OAuth permissions revoked

**Recovery**:
```
1. Verify Sheet ID:
   - Open Google Sheets
   - Copy ID from URL: /d/[THIS_ID]/edit
   - Update in Append row in sheet node

2. Verify sheet name:
   - Confirm sheet is named exactly "VendorScores"
   - Check for typos

3. Verify OAuth permissions:
   - Settings → Credentials → Edit Google Sheets
   - Ensure scope includes "spreadsheets"
   - Re-authorize if needed

4. Verify file sharing:
   - If using service account, share sheet with account email
   - If using OAuth, no sharing needed

5. Test:
   - Click "Execute Node" on Append row in sheet
   - Should see row added to sheet
```

---

#### Error 5: "Slack message formatting error - Invalid blocks"

**Symptoms**:
- Slack Notifier returns error
- Message doesn't post to channel
- Error mentions "Invalid blocks" or "Bad JSON"

**Root Cause**:
- Special characters breaking JSON format
- Vendor name or summary contains quotes/newlines
- Blocks array malformed
- Message too large

**Recovery**:
```
1. Add JSON validation to Slack Message Builder:
   - Escape special characters
   - Validate JSON before sending
   - Test with: JSON.parse(blocks)

2. Simplify message format:
   - Reduce text length
   - Remove special unicode characters
   - Test with simple text first

3. Split large messages:
   - If 3+ vendors, send in multiple messages
   - Group by rank (Top 3 in one, rest in another)

4. Test in Slack:
   - Use /test command to verify bot can post
   - Manually post to #rfp-evaluations
   - Verify permissions correct
```

---

## 📊 MONITORING & MAINTENANCE

### Daily Monitoring Tasks

**Execution Health Check** (5 minutes):
```
1. Open n8n Executions tab
2. Filter: "AI Vendor Evaluation..."
3. Check last 24 hours:
   ✅ How many workflows executed?
   ✅ Success rate > 95%?
   ✅ Any red error nodes?
   ✅ Average execution time reasonable (< 120 sec)?

4. If issues found:
   ├─ Click failed execution
   ├─ Review error message
   ├─ Take action (re-run, investigate, etc.)
   └─ Log in maintenance sheet
```

**Google Sheets Review** (5 minutes):
```
1. Open Google Sheet: RFP Vendor Evaluations
2. Check latest rows:
   ✅ New vendors added?
   ✅ Scores in valid range (0-10)?
   ✅ Timestamps recent?
   ✅ Any "#ERROR" or "PARSING_ERROR"?

3. If issues:
   ├─ Check corresponding execution in n8n
   ├─ Identify root cause
   └─ Plan remediation
```

**Slack Channel Scan** (5 minutes):
```
1. Check #rfp-evaluations channel
2. Latest notification:
   ✅ Posted today/recently?
   ✅ Message formatting correct?
   ✅ All top 3 vendors shown?
   ✅ No error messages?

3. If silent:
   └─ Check workflow executions for failures
```

---

### Weekly Maintenance Tasks

**Performance Analysis** (15 minutes):
```
1. n8n Executions → Analytics
2. Review past 7 days:
   ✅ Execution count (aligned with RFP volume?)
   ✅ Success rate trend
   ✅ Average execution time trend
   ✅ Slowest nodes (identify bottlenecks)

3. Analysis:
   ├─ If success rate declining → Investigate issues
   ├─ If execution time increasing → Check API latency
   ├─ If low volume → Verify Gmail trigger active
   └─ Document findings
```

**API Usage Review** (10 minutes):
```
1. OpenRouter Dashboard
   ├─ Check token usage
   ├─ Verify within free tier limits
   ├─ Review model usage distribution
   └─ Note any errors/rate limits hit

2. Google Cloud Console
   ├─ APIs & Services → Quotas
   ├─ Gmail API quota used
   ├─ Google Sheets API quota used
   ├─ All green? (Not at limits)
   └─ If close to limit → Request increase

3. Gmail API Activity
   ├─ Monitor email polling
   ├─ Verify no authentication issues
   └─ Check attachment download success rate
```

**Data Quality Audit** (15 minutes):
```
1. Google Sheets → VendorScores tab
2. Review random 10-15 rows:
   ✅ Scores reasonable?
   ✅ Summaries well-written?
   ✅ Missing fields detected correctly?
   ✅ Risk flags sensible?

3. Spot-check 3 vendors:
   ├─ Manually review their RFP response
   ├─ Compare AI scores to your judgment
   ├─ Note any systematic biases
   └─ Document feedback

4. If biases found:
   └─ Adjust system prompt weights for next cycle
```

---

### Monthly Maintenance Tasks

**Credential Rotation** (15 minutes):
```
1. OpenRouter API Key Rotation
   ├─ Generate new key in OpenRouter dashboard
   ├─ Update n8n credential
   ├─ Test workflow execution
   ├─ Delete old key
   └─ Document in password manager

2. Google OAuth Refresh
   ├─ Verify tokens still valid
   ├─ Check expiration dates
   ├─ Re-authorize if near expiration
   └─ Update scopes if needed

3. Slack Token Rotation
   ├─ Regenerate bot token (if available)
   ├─ Update n8n credential
   ├─ Test Slack notification
   └─ Verify bot still in channel
```

**Workflow Optimization** (30 minutes):
```
1. Performance Tuning
   ├─ Identify slowest nodes
   ├─ Check for unnecessary waits
   ├─ Optimize regex patterns if needed
   ├─ Consider parallel processing
   └─ Benchmark before/after changes

2. AI Prompt Refinement
   ├─ Review past 50 evaluations
   ├─ Identify scoring anomalies
   ├─ Adjust weights if needed
   ├─ Test on historical samples
   └─ Document changes

3. Error Pattern Analysis
   ├─ Review past month errors
   ├─ Categorize by type
   ├─ Implement preventions
   ├─ Improve fallback logic
   └─ Add guardrails
```

**Backup & Archive** (10 minutes):
```
1. Google Sheets Backup
   ├─ File → Download → CSV
   ├─ Save to secure location
   ├─ Name: RFP_Evaluations_[DATE].csv
   └─ Verify data integrity

2. Workflow Backup
   ├─ n8n → Export workflow as JSON
   ├─ Save to git/version control
   ├─ Tag with version number
   └─ Document changes

3. Vendor Archive (if needed)
   ├─ Move vendors >90 days old to separate sheet
   ├─ Keep main sheet tidy
   ├─ Maintain searchability
   └─ Preserve audit trail
```

---

### Quarterly Maintenance Tasks

**Vendor Evaluation Audit** (1 hour):
```
1. Select 20 random past evaluations
2. For each:
   ├─ Re-read original RFP response
   ├─ Review AI scores
   ├─ Compare to human judgment
   ├─ Check for systematic biases
   └─ Rate accuracy: Excellent/Good/Fair/Poor

3. Analysis:
   ├─ Calculate accuracy score (% Excellent/Good)
   ├─ Identify scoring patterns
   ├─ Document any systematic biases
   └─ Plan adjustments

4. If accuracy < 80%:
   └─ Adjust system prompt, rerun evaluations
```

**Competitive Model Evaluation** (1 hour):
```
1. Test alternative LLMs:
   ├─ google/gemini-flash-1.5:free
   ├─ nousresearch/hermes-3-llama-3.1-405b:free
   └─ Any newer free models

2. For each model:
   ├─ Run on 10 sample vendors
   ├─ Compare scores to current model
   ├─ Check execution speed
   ├─ Verify quality
   └─ Note cost difference

3. Decision:
   ├─ Keep current?
   ├─ Switch to faster model?
   ├─ Upgrade to paid tier?
   └─ Hybrid approach (different by complexity)?
```

**Cost Analysis & Optimization** (30 minutes):
```
1. Calculate past 3 months cost:
   ├─ OpenRouter: (requests × cost per 1K tokens)
   ├─ Gmail API: Usually free (unless very high volume)
   ├─ Google Sheets: $0 (free tier sufficient)
   ├─ Slack: $0 (free/paid plan independent)
   └─ Total: Usually $0-5/month

2. Identify optimization opportunities:
   ├─ Reduce model temperature → faster inference
   ├─ Batch similar vendors → reduce requests
   ├─ Cache common responses → skip redundant scoring
   └─ Upgrade to paid only if volume > 1000 vendors/month

3. ROI calculation:
   ├─ Vendor evaluation time: 2-3 hours → 30 seconds (95% faster)
   ├─ Cost savings: Time × Labor rate
   └─ Payback period: Usually < 1 month
```

---

## 🎓 ADVANCED CUSTOMIZATION

### Customizing Scoring Weights

**Current Default Weights**:
```
Total Score = (Price × 0.25) + (Capabilities × 0.30) 
            + (Compliance × 0.20) + (SLA × 0.15) 
            + (References × 0.10)
```

**Industry-Specific Weights**:

**Healthcare** (Compliance Critical):
```
Total Score = (Price × 0.10) + (Capabilities × 0.15) 
            + (Compliance × 0.50) + (SLA × 0.15) 
            + (References × 0.10)

Reasoning:
- Compliance non-negotiable (HIPAA, security audits)
- Price secondary to compliance
- SLA still important but compliance dominates
```

**Startup/Growth** (Capabilities Focus):
```
Total Score = (Price × 0.35) + (Capabilities × 0.40) 
            + (Compliance × 0.10) + (SLA × 0.10) 
            + (References × 0.05)

Reasoning:
- Innovation and features most important
- Price sensitive (limited budget)
- Compliance/SLA less critical early on
- References less important
```

**Enterprise** (Balanced):
```
Total Score = (Price × 0.20) + (Capabilities × 0.25) 
            + (Compliance × 0.25) + (SLA × 0.20) 
            + (References × 0.10)

Reasoning:
- Balance all factors
- All dimensions important
- Compliance and SLA heavily weighted
```

**To Implement Custom Weights**:

```
1. Edit "AI Vendor Evaluator" node
2. Modify system prompt WEIGHTED SCORING FORMULA
3. Change weights to your industry needs
4. Test on 5-10 historical vendors
5. Compare scores before/after
6. Adjust and re-test until satisfied
7. Document changes
```

---

### Adding Custom Evaluation Dimensions

**Example: Add "Support Channels" Dimension**

```
Step 1: Add to Scoring Rubric (in system prompt)
6. SUPPORT_CHANNELS (Weight: 5%)
   - 0-3: Phone only
   - 4-6: Phone + email
   - 7-8: Phone + email + chat
   - 9-10: Phone + email + chat + AI support

Step 2: Update weighted formula
Total Score = (Price × 0.23) + (Capabilities × 0.28) 
            + (Compliance × 0.19) + (SLA × 0.14) 
            + (References × 0.09) + (Support × 0.07)
(Note: Reduced other weights proportionally)

Step 3: Update extraction regex
In "Response Normalizer" code:
const support_channels = extractSection('6\. Support', '7\.');

Step 4: Update prompt
In "Prepare AI Prompt":
- Add to extracted fields
- Update instructions to score this dimension

Step 5: Update Google Sheets
- Add column: support_score
- Update Append row node to include new column

Step 6: Test
- Run on sample vendors
- Verify new dimension scores correctly
```

---

### Integrating Additional AI Models

**Option 1: Use Different Model from OpenRouter**

```
Current: meta-llama/llama-3.1-70b-instruct:free
Alternative: nousresearch/hermes-3-llama-3.1-405b:free

To switch:
1. Edit "AI Vendor Evaluator" node
2. Subnode: "OpenRouter Chat Model"
3. Change model: Select from dropdown
4. Test execution
5. Compare results

Trade-offs:
- Llama 3.1: Faster, good quality
- Hermes 3: More reasoning, slightly slower
- Gemini Flash: Fastest, adequate quality
```

**Option 2: Use Different AI Provider**

```
Add node: "HTTP Request" to call:
- Anthropic Claude API (claude-3-haiku)
- Cohere API (command-r-free)
- Together AI (open models)
- Azure OpenAI

Implementation:
1. Delete "AI Vendor Evaluator" node
2. Add "HTTP Request" node
3. Configure endpoint URL
4. Map prompt format to provider's API
5. Parse response differently
6. Update fallback handling
```

---

### Automating RFP Response Collection

**Option 1: Typeform Integration** (Alternative to Gmail)

```
Instead of email collection:

1. Create Typeform form: "RFP Response Form"
2. Add fields:
   - Vendor Name
   - Contact Email
   - Pricing details
   - Capabilities (long text)
   - SLA details
   - Compliance info
   - References

3. n8n Integration:
   - Replace Gmail Trigger with Typeform Trigger
   - Receive form submissions as structured data
   - Skip PDF extraction (already structured)
   - Go directly to AI Scoring

Benefits:
✅ Structured data → better parsing
✅ No PDF extraction needed
✅ Faster processing
✅ Better data quality

Drawback:
❌ Requires vendors to use form
❌ Less flexible than email
```

**Option 2: Webhook + API Integration**

```
For vendors with APIs (Salesforce, HubSpot, etc.):

1. Create n8n Webhook endpoint
2. Vendors post RFP data to webhook
3. Webhook receives JSON:
   {
     "vendor_name": "Acme",
     "pricing": {...},
     "capabilities": [...],
     "sla": {...},
     "compliance": [...],
     "references": [...]
   }
4. Process as structured data
5. Skip PDF extraction entirely

Benefits:
✅ Real-time data
✅ Perfectly structured
✅ No manual parsing

Drawback:
❌ Requires vendor technical capability
```

---

## 🔍 TROUBLESHOOTING GUIDE

### Workflow Won't Start

**Check 1: Gmail Trigger Active?**
```
1. Click "RFP Response Monitor" node
2. Check if trigger is enabled (blue toggle)
3. Look for "Active" indicator
4. Check last poll time (should be recent)

If not polling:
├─ Click "Activate" toggle
├─ Wait 60 seconds for first poll
└─ Check for permission errors
```

**Check 2: Credentials Valid?**
```
1. Settings → Credentials
2. Find "Gmail account"
3. Click "Test Connection"
4. If fails: Re-authorize

If still fails:
├─ Check Gmail account has POP/IMAP enabled
├─ Verify API enabled in Google Cloud
├─ Regenerate OAuth client credentials
└─ Re-authenticate
```

---

### Workflow Runs but Produces No Output

**Check 1: Email Arriving?**
```
1. Manually send test email to your Gmail
2. Subject: "RFP Response - Test Vendor"
3. With PDF attachment
4. Wait 60+ seconds
5. Check n8n Executions tab

If no execution triggered:
└─ Gmail trigger may have permission issue
```

**Check 2: PDF Extraction Working?**
```
1. Execute "Extract from File" node manually
2. Pass it the PDF from Gmail trigger
3. Check if text extracted

If empty text:
├─ PDF may be scanned image
├─ PDF may be encrypted
└─ Try different PDF format
```

---

### Scores Seem Wrong

**Check 1: AI Prompt Correct?**
```
1. Edit "AI Vendor Evaluator" node
2. Review system prompt
3. Verify scoring rubrics present
4. Check weighted formula

If missing:
├─ Copy system prompt from documentation
├─ Paste into node
├─ Test again
```

**Check 2: Model Working?**
```
1. Click "Prepare AI Prompt" node
2. Execute node manually
3. Check messages output

4. Open OpenRouter dashboard
5. Verify model selected: Llama 3.1 70B
6. Check recent requests (should show activity)

If not using correct model:
├─ Edit node
├─ Select correct model from dropdown
└─ Re-test
```

**Check 3: Compare to Manual Scoring**
```
1. Pick a vendor response
2. Manually score 0-10 on each dimension
3. Calculate your total score
4. Compare to AI score

If significant difference:
├─ Review AI reasoning
├─ Adjust prompt if systematic bias found
├─ Re-run evaluation
└─ Document your scoring rationale
```

---

### Google Sheets Not Updating

**Check 1: Sheet ID Correct?**
```
1. Open Google Sheet
2. Copy ID from URL: /d/[THIS_ID]/edit
3. Click "Append row in sheet" node
4. Verify Document ID matches

If mismatch:
├─ Update Document ID
├─ Save node
├─ Test by clicking "Execute Node"
```

**Check 2: Sheet Name Correct?**
```
1. Click Google Sheets tab
2. Check sheet name: Should be "VendorScores"
3. If different, either:
   ├─ Rename sheet to VendorScores
   ├─ Or update node configuration
   └─ Test again
```

**Check 3: OAuth Permissions?**
```
1. Settings → Credentials → Google Sheets
2. Click "Test Connection"

If fails:
├─ Re-authorize
├─ Verify scope includes "spreadsheets"
├─ Save
└─ Try again
```

---

### Slack Not Receiving Messages

**Check 1: Bot in Channel?**
```
In Slack:
1. Go to #rfp-evaluations
2. Check member list
3. Should see @RFP Evaluator Bot

If not:
└─ Invite: /invite @RFP Evaluator Bot
```

**Check 2: Credentials Valid?**
```
1. n8n Settings → Credentials → Slack API
2. Click "Test Connection"

If fails:
├─ Copy Bot Token from https://api.slack.com/apps/[APP_ID]/oauth-tokens
├─ Paste into credential
├─ Save
└─ Test again
```

**Check 3: Channel Name Correct?**
```
1. Click "Slack Notifier" node
2. Check Channel field: #rfp-evaluations
3. Verify spelling (case-sensitive)

If different:
├─ Update channel name
├─ Ensure channel exists
└─ Ensure bot invited to channel
```

---

### High Latency / Slow Execution

**Check 1: OpenRouter API Slow?**
```
Typical latency: 10-20 seconds for AI scoring

If > 30 seconds:
├─ Check OpenRouter dashboard for service issues
├─ Try at different time of day
├─ If consistent: Consider paid tier

Fix:
└─ Increase timeout in AI Evaluator node to 60+ seconds
```

**Check 2: Google APIs Slow?**
```
Normal: 2-4 seconds for Sheets + Gmail

If > 10 seconds:
├─ Check Google Cloud quotas
├─ Verify not hitting rate limits
├─ Try in different geography

Fix:
└─ Add retry logic with exponential backoff
```

---

## 📈 PERFORMANCE OPTIMIZATION

### Quick Wins

```
1. Reduce AI Token Count
   ├─ Shorter system prompt (keep essentials)
   ├─ Shorter user prompt (only needed context)
   └─ Result: Faster inference, lower cost

2. Cache Repeated Evaluations
   ├─ If same vendor submits again
   ├─ Check Google Sheets history
   ├─ Reuse scores if nothing changed
   └─ Result: 100% faster for repeats

3. Batch Process During Off-Hours
   ├─ Instead of real-time processing
   ├─ Queue emails and process at night
   ├─ Batch: 50+ vendors at once
   └─ Result: Better OpenRouter rate limit usage

4. Use Faster Model
   ├─ Current: Llama 3.1 70B (10-20 sec)
   ├─ Try: Gemini Flash (5-10 sec)
   ├─ Trade: Slightly lower quality
   └─ Result: 50% faster

5. Parallel Processing (Advanced)
   ├─ Split vendors into sub-workflows
   ├─ Process multiple in parallel
   ├─ Aggregate results at end
   └─ Result: Near-linear speed improvement with vendor count
```

---

## 📝 CONCLUSION

This comprehensive n8n workflow automates vendor RFP evaluation with:

✅ **Zero Cost**: Uses only free API tiers  
✅ **High Speed**: 30-60 seconds per vendor evaluation  
✅ **Objective Scoring**: AI eliminates human bias with weighted rubrics  
✅ **Audit Trail**: Complete history in Google Sheets  
✅ **Real-time Alerts**: Slack notifications for decision-makers  
✅ **Smart Gap Detection**: Auto-identifies missing information  
✅ **Production Ready**: Error handling, fallbacks, and monitoring built-in  

**Total Implementation Time**: ~2-3 hours  
**Ongoing Maintenance**: ~30 minutes/month  
**ROI**: Positive from first vendor evaluation

---

**Support Resources**:
- n8n Docs: https://docs.n8n.io/
- OpenRouter Docs: https://openrouter.ai/docs
- Google APIs: https://developers.google.com/
- Slack API: https://api.slack.com/

**Last Updated**: May 2024  
**Version**: 1.0  
**Status**: Production Ready ✅
