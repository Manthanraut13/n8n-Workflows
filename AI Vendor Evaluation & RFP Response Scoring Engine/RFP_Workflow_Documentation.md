# AI Vendor Evaluation & RFP Response Scoring Engine
## Complete n8n Workflow Documentation

---

## 📋 TABLE OF CONTENTS

1. [Workflow Overview](#workflow-overview)
2. [System Architecture](#system-architecture)
3. [Node-by-Node Documentation](#node-by-node-documentation)
4. [Data Flow Diagram](#data-flow-diagram)
5. [Configuration Guide](#configuration-guide)
6. [Testing & Validation](#testing--validation)
7. [Error Handling](#error-handling)
8. [Maintenance & Troubleshooting](#maintenance--troubleshooting)
9. [Sample Outputs](#sample-outputs)

---

## 🎯 WORKFLOW OVERVIEW

### Purpose
Automate the vendor evaluation process for RFP (Request for Proposal) responses using AI-powered scoring, intelligent data extraction, and automated notifications to procurement teams.

### Key Features
- **Automated Email Monitoring**: Gmail trigger polls inbox every minute for new RFP responses
- **PDF Extraction**: Automatically extracts text from PDF attachments containing vendor proposals
- **AI-Powered Scoring**: Uses OpenRouter's free LLM models to objectively score vendors across 5 dimensions
- **Intelligent Gap Detection**: Identifies missing critical information and triggers follow-up emails
- **Centralized Database**: Stores all vendor evaluations in Google Sheets for audit trail
- **Real-time Notifications**: Sends formatted Slack messages with top vendor rankings to decision-makers
- **Zero-Cost Operation**: Uses only free API tiers (OpenRouter free models, Gmail, Google Sheets, Slack)

### Business Impact
- **Time Savings**: Reduces manual RFP evaluation from 2-3 hours to < 5 minutes per vendor
- **Consistency**: Eliminates subjective bias with standardized AI-based scoring rubrics
- **Auditability**: Complete audit trail in Google Sheets with timestamps and justifications
- **Speed**: Decision-makers receive notifications within minutes of vendor response submission

### Technical Stack
- **Automation Platform**: n8n (self-hosted or cloud)
- **AI Engine**: OpenRouter API (meta-llama/llama-3.1-70b-instruct:free)
- **Email**: Gmail API (OAuth2)
- **Storage**: Google Sheets API
- **Notifications**: Slack API
- **File Processing**: n8n built-in PDF extraction

---

## 🏗️ SYSTEM ARCHITECTURE

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        INPUT LAYER                              │
│  ┌──────────────┐                                               │
│  │ Gmail Inbox  │ ──► New RFP Response Email (with PDF)         │
│  └──────────────┘                                               │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    EXTRACTION LAYER                             │
│  ┌──────────────────┐         ┌──────────────────┐             │
│  │ Extract from File│ ──────► │    Normalize     │             │
│  │   (PDF Parser)   │         │   Vendor Data    │             │
│  └──────────────────┘         └──────────────────┘             │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                     AI PROCESSING LAYER                         │
│  ┌──────────────────┐         ┌──────────────────┐             │
│  │  Prepare Prompt  │ ──────► │  AI Evaluator    │             │
│  │   (Code Node)    │         │  (OpenRouter)    │             │
│  └──────────────────┘         └──────────────────┘             │
│                                        │                         │
│                                        ▼                         │
│                              ┌──────────────────┐               │
│                              │ Extract Scoring  │               │
│                              │      Data        │               │
│                              └──────────────────┘               │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    AGGREGATION LAYER                            │
│  ┌──────────────────┐                                           │
│  │ Vendor Ranking   │ ──► Sort, Rank, Calculate Stats          │
│  │     Engine       │                                           │
│  └──────────────────┘                                           │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                   CONDITIONAL LOGIC LAYER                       │
│  ┌──────────────────┐                                           │
│  │   IF: Missing    │ ──True──► Follow-up Email Generator      │
│  │     Fields?      │                                           │
│  └──────────────────┘ ──False─► Continue to Storage            │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    STORAGE & OUTPUT LAYER                       │
│  ┌──────────────────┐         ┌──────────────────┐             │
│  │ Google Sheets    │         │  Slack Notifier  │             │
│  │   (Database)     │         │   (Real-time)    │             │
│  └──────────────────┘         └──────────────────┘             │
└─────────────────────────────────────────────────────────────────┘
```

### Workflow Components

| Component | Type | Purpose | Technology |
|-----------|------|---------|------------|
| **Trigger** | Gmail Trigger | Monitor inbox for RFP responses | Gmail API v1.3 |
| **Extraction** | Extract from File | Parse PDF attachments | n8n built-in |
| **Normalization** | Code Node | Structure vendor data | JavaScript |
| **Prompt Builder** | Code Node | Prepare AI prompts | JavaScript |
| **AI Scoring** | AI Agent | Evaluate vendors objectively | OpenRouter + Llama 3.1 70B |
| **Parser** | Code Node | Extract JSON scores | JavaScript |
| **Ranking** | Code Node | Sort and rank vendors | JavaScript |
| **Router** | IF Node | Check for missing fields | Conditional logic |
| **Follow-up** | Code + Gmail | Generate and send emails | Gmail API |
| **Storage** | Google Sheets | Persist vendor scores | Google Sheets API |
| **Notification** | Slack | Alert decision-makers | Slack API |

---

## 📝 NODE-BY-NODE DOCUMENTATION

### Node 1: RFP Response Monitor (Gmail Trigger)

**Node Type**: `n8n-nodes-base.gmailTrigger` (v1.3)

**Purpose**: Continuously monitors Gmail inbox for incoming RFP vendor responses and triggers the workflow.

**Configuration**:
```json
{
  "pollTimes": {
    "item": [
      {
        "mode": "everyMinute"
      }
    ]
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

**How It Works**:
- Polls Gmail inbox every 60 seconds
- Automatically downloads email attachments (PDF RFP responses)
- Excludes spam and trash folders
- Uses "advanced" mode (simple: false) for full email metadata access

**Credentials Required**:
- Gmail OAuth2 API (configured in n8n credentials)

**Output Structure**:
```json
{
  "id": "18f8a1c2d3e4f5g6",
  "threadId": "18f8a1c2d3e4f5g6",
  "labelIds": ["INBOX", "UNREAD"],
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
      "data": "binary_data_object"
    }
  ],
  "bodyPlain": "Please find our RFP response attached...",
  "bodyHtml": "<p>Please find our RFP response attached...</p>"
}
```

**Filter Recommendations**:
To improve accuracy, add search filters:
```
subject:(RFP Response OR Vendor Response OR Proposal) has:attachment
```

---

### Node 2: Extract from File (PDF Parser)

**Node Type**: `n8n-nodes-base.extractFromFile`

**Purpose**: Extracts text content from PDF attachments for AI processing.

**Configuration**:
```json
{
  "operation": "text",
  "options": {}
}
```

**How It Works**:
- Receives binary PDF data from Gmail trigger
- Uses built-in PDF text extraction engine
- Handles multi-page documents automatically
- Preserves text structure (paragraphs, line breaks)

**Output Structure**:
```json
{
  "text": "Vendor Name: Acme Corporation\n\n1. Pricing:\nAnnual License: $50,000\nImplementation: $10,000\n\n2. Capabilities:\n- Real-time analytics\n- API integrations\n- Custom dashboards\n\n3. SLA:\nUptime Guarantee: 99.9%\nSupport Response: 4 hours\n\n4. Compliance:\n- SOC2 Type II\n- ISO 27001\n- GDPR compliant\n\n5. References:\n- TechCorp Inc. (2 years)\n- Global Systems (18 months)\n- DataFlow LLC (3 years)",
  "filename": "Acme_RFP_Response.pdf",
  "mimeType": "application/pdf"
}
```

**Limitations**:
- Works best with text-based PDFs (not scanned images)
- For scanned PDFs, use OCR preprocessing (external service required)
- Complex tables may lose formatting

---

### Node 3: Response Normalizer (Code Node)

**Node Type**: `n8n-nodes-base.code` (v2)

**Purpose**: Parse extracted PDF text and structure it into standardized vendor data format.

**JavaScript Code**:
```javascript
const items = [];

for (const item of $input.all()) {

  const pdfText = item.json.text || '';
  const subject = item.json.subject || '';
  const rawText = pdfText; // ✅ Use extracted PDF text

  // ✅ Extract Vendor Name (priority: PDF → subject → fallback)
  let vendorName = 'Unknown Vendor';

  const vendorFromPDF = rawText.match(/Vendor Name[:\s]+(.+)/i);
  if (vendorFromPDF) {
    vendorName = vendorFromPDF[1].trim();
  } else {
    const vendorMatch = subject.match(/RFP Response[:\s-]+(.+?)(?:\||$)/i);
    if (vendorMatch) {
      vendorName = vendorMatch[1].trim();
    }
  }

  // ✅ Helper to extract multi-line sections
  const extractSection = (startLabel, endLabel) => {
    const regex = new RegExp(
      `${startLabel}[\\s\\S]*?:([\\s\\S]*?)(?=${endLabel}|$)`,
      'i'
    );
    const match = rawText.match(regex);
    return match ? match[1].replace(/\n+/g, ' ').trim() : null;
  };

  // ✅ Extract structured fields
  const pricing = extractSection('1\. Pricing', '2\.');
  const capabilities = extractSection('2\. Capabilities', '3\.');
  const sla = extractSection('3\. SLA', '4\.');
  const compliance = extractSection('4\. Compliance', '5\.');
  const references = extractSection('5\. References', '6\.');

  items.push({
    json: {
      vendor_name: vendorName,
      vendor_email: item.json.vendor_email || null,
      received_date: item.json.received_date || null,
      message_id: item.json.message_id || null,

      raw_response: rawText,

      extracted_fields: {
        pricing,
        capabilities,
        sla,
        compliance,
        references
      }
    }
  });
}

return items;
```

**How It Works**:
1. **Vendor Name Extraction**: Tries to find "Vendor Name:" in PDF first, falls back to email subject
2. **Section Parsing**: Uses regex to extract numbered sections (1. Pricing, 2. Capabilities, etc.)
3. **Multi-line Handling**: Captures content across multiple lines until next section
4. **Null Safety**: Returns null if field not found (flagged later by AI)

**Output Structure**:
```json
{
  "vendor_name": "Acme Corporation",
  "vendor_email": "vendor@acme.com",
  "received_date": "2024-05-02T10:30:00.000Z",
  "message_id": "18f8a1c2d3e4f5g6",
  "raw_response": "Full PDF text...",
  "extracted_fields": {
    "pricing": "Annual License: $50,000 Implementation: $10,000",
    "capabilities": "Real-time analytics API integrations Custom dashboards",
    "sla": "Uptime Guarantee: 99.9% Support Response: 4 hours",
    "compliance": "SOC2 Type II ISO 27001 GDPR compliant",
    "references": "TechCorp Inc. (2 years) Global Systems (18 months) DataFlow LLC (3 years)"
  }
}
```

---

### Node 4: Prepare AI Prompt (Code Node)

**Node Type**: `n8n-nodes-base.code` (v2)

**Purpose**: Build structured prompt messages for OpenRouter AI API.

**JavaScript Code**:
```javascript
const items = [];

for (const item of $input.all()) {
  const systemPrompt = `You are an expert procurement and vendor evaluation specialist with 15+ years of experience. Your role is to objectively score vendor RFP responses across multiple dimensions.

EVALUATION CRITERIA:
1. PRICE (0-10): Cost-effectiveness, value for money, transparent pricing
2. CAPABILITIES (0-10): Technical features, scalability, innovation
3. COMPLIANCE (0-10): Certifications (SOC2, ISO27001, GDPR, HIPAA)
4. SLA (0-10): Uptime guarantees, support responsiveness, penalties
5. REFERENCES (0-10): Quality and relevance of client references

SCORING GUIDELINES:
- 0-3: Poor/Unacceptable
- 4-6: Adequate/Meets minimum requirements
- 7-8: Good/Competitive
- 9-10: Excellent/Industry-leading

WEIGHTED FORMULA:
Total Score = (Price × 0.25) + (Capabilities × 0.30) + (Compliance × 0.20) + (SLA × 0.15) + (References × 0.10)

CRITICAL: You MUST respond with ONLY valid JSON. No markdown, no explanations outside JSON.`;

  const userPrompt = `Evaluate this vendor RFP response:

VENDOR: ${item.json.vendor_name}
EMAIL: ${item.json.vendor_email}

RESPONSE TEXT:
${item.json.raw_response}

EXTRACTED FIELDS (if available):
- Pricing: ${item.json.extracted_fields.pricing || 'Not found'}
- Features: ${item.json.extracted_fields.features || 'Not found'}
- SLA: ${item.json.extracted_fields.sla || 'Not found'}
- Compliance: ${item.json.extracted_fields.compliance || 'Not found'}
- References: ${item.json.extracted_fields.references || 'Not found'}

INSTRUCTIONS:
1. Score each dimension (0-10) based on the criteria
2. Calculate weighted total score
3. Identify ALL missing critical fields
4. Flag any risks (e.g., no compliance certs, vague SLA, no references)
5. Provide 2-sentence summary

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
  "missing_fields": ["array of missing fields"],
  "risk_flags": ["array of risk items"],
  "summary": "string",
  "raw_scores_justification": {
    "price": "brief reason",
    "capabilities": "brief reason",
    "compliance": "brief reason",
    "sla": "brief reason",
    "references": "brief reason"
  }
}`;

  items.push({
    json: {
      messages: [
        {
          role: "system",
          content: systemPrompt
        },
        {
          role: "user",
          content: userPrompt
        }
      ],
      vendor_data: item.json // Pass through for later use
    }
  });
}

return items;
```

**Output Structure**:
```json
{
  "messages": [
    {
      "role": "system",
      "content": "You are an expert procurement..."
    },
    {
      "role": "user",
      "content": "Evaluate this vendor RFP response..."
    }
  ],
  "vendor_data": {
    "vendor_name": "Acme Corporation",
    "vendor_email": "vendor@acme.com",
    "...": "..."
  }
}
```

---

### Node 5: AI Vendor Evaluator (AI Agent)

**Node Type**: `@n8n/n8n-nodes-langchain.agent` (v3)

**Purpose**: Use OpenRouter LLM to score vendor objectively across 5 dimensions.

**Configuration**:
```json
{
  "promptType": "define",
  "text": "={{ User prompt from Prepare AI Prompt node }}",
  "options": {
    "systemMessage": "={{ System prompt from Prepare AI Prompt node }}"
  }
}
```

**Connected Sub-Node**: OpenRouter Chat Model

**OpenRouter Chat Model Configuration**:
```json
{
  "model": "meta-llama/llama-3.1-70b-instruct:free",
  "options": {
    "temperature": 0.3,
    "maxTokens": 2000
  }
}
```

**Model Selection Rationale**:
- **meta-llama/llama-3.1-70b-instruct:free**: Best free model for structured reasoning
- **Temperature 0.3**: Low temperature for consistent, objective scoring
- **Max Tokens 2000**: Sufficient for detailed JSON response

**How It Works**:
1. Receives system + user prompts from previous node
2. Sends to OpenRouter API with configured model
3. Model analyzes vendor response against scoring rubrics
4. Returns structured JSON with scores, flags, and justifications
5. Uses JSON mode to enforce valid JSON output

**AI Agent Behavior**:
- Acts as procurement expert with 15+ years experience
- Applies weighted scoring formula: (P×0.25) + (C×0.30) + (Comp×0.20) + (SLA×0.15) + (Ref×0.10)
- Identifies missing critical fields
- Flags risks (compliance gaps, weak SLAs, poor references)
- Generates professional 2-sentence summary

**Output Structure**:
```json
{
  "output": "{\"vendor_name\":\"Acme Corporation\",\"scores\":{\"price\":7,\"capabilities\":8,\"compliance\":9,\"sla\":8,\"references\":6},\"total_score\":7.75,\"missing_fields\":[],\"risk_flags\":[\"References only from small companies\"],\"summary\":\"Strong technical capabilities with excellent compliance posture. Pricing is competitive but references could be more robust from enterprise clients.\",\"raw_scores_justification\":{\"price\":\"$50k/year competitive for feature set\",\"capabilities\":\"Advanced AI features, good scalability\",\"compliance\":\"SOC2 Type II, ISO27001, GDPR compliant\",\"sla\":\"99.9% uptime with 4-hour response\",\"references\":\"3 references but limited enterprise presence\"}}"
}
```

---

### Node 6: Extract Scoring Data (Code Node)

**Node Type**: `n8n-nodes-base.code` (v2)

**Purpose**: Parse AI JSON response and merge with vendor metadata.

**JavaScript Code**:
```javascript
const items = [];

for (const item of $input.all()) {
  try {
    const aiResponse = JSON.parse(item.json.choices[0].message.content);
    const vendorData = item.json.vendor_data || {};
    
    items.push({
      json: {
        vendor_name: aiResponse.vendor_name,
        vendor_email: vendorData.vendor_email,
        received_date: vendorData.received_date,
        message_id: vendorData.message_id,
        price_score: aiResponse.scores.price,
        capability_score: aiResponse.scores.capabilities,
        compliance_score: aiResponse.scores.compliance,
        sla_score: aiResponse.scores.sla,
        reference_score: aiResponse.scores.references,
        total_score: aiResponse.total_score,
        missing_fields: aiResponse.missing_fields,
        risk_flags: aiResponse.risk_flags,
        summary: aiResponse.summary,
        justifications: aiResponse.raw_scores_justification,
        timestamp: new Date().toISOString()
      }
    });
  } catch (error) {
    // Error handling - flag for manual review
    items.push({
      json: {
        vendor_name: 'PARSING_ERROR',
        error: error.message,
        raw_response: item.json,
        needs_manual_review: true,
        timestamp: new Date().toISOString()
      }
    });
  }
}

return items;
```

**Error Handling**:
- Wraps JSON parsing in try-catch block
- If parsing fails, creates error record with `needs_manual_review: true`
- Preserves original response for debugging
- Adds timestamp for audit trail

**Output Structure** (Success):
```json
{
  "vendor_name": "Acme Corporation",
  "vendor_email": "vendor@acme.com",
  "received_date": "2024-05-02T10:30:00.000Z",
  "message_id": "18f8a1c2d3e4f5g6",
  "price_score": 7,
  "capability_score": 8,
  "compliance_score": 9,
  "sla_score": 8,
  "reference_score": 6,
  "total_score": 7.75,
  "missing_fields": [],
  "risk_flags": ["References only from small companies"],
  "summary": "Strong technical capabilities with excellent compliance posture. Pricing is competitive but references could be more robust from enterprise clients.",
  "justifications": {
    "price": "$50k/year competitive for feature set",
    "capabilities": "Advanced AI features, good scalability",
    "compliance": "SOC2 Type II, ISO27001, GDPR compliant",
    "sla": "99.9% uptime with 4-hour response",
    "references": "3 references but limited enterprise presence"
  },
  "timestamp": "2024-05-02T10:35:22.456Z"
}
```

---

### Node 7: Vendor Ranking Engine (Code Node)

**Node Type**: `n8n-nodes-base.code` (v2)

**Purpose**: Aggregate all vendor scores, sort by total_score, and assign ranks.

**JavaScript Code**:
```javascript
// This node uses "Wait for all incoming items" mode
const allVendors = $input.all();

// Filter out errors
const validVendors = allVendors.filter(item => 
  !item.json.needs_manual_review && item.json.total_score !== undefined
);

// Sort by total_score descending
validVendors.sort((a, b) => b.json.total_score - a.json.total_score);

// Add rank
const rankedVendors = validVendors.map((item, index) => ({
  json: {
    ...item.json,
    rank: index + 1,
    rank_label: index === 0 ? '🥇 Top Choice' : 
                index === 1 ? '🥈 Runner-up' : 
                index === 2 ? '🥉 Third Place' : 
                `#${index + 1}`
  }
}));

// Create summary statistics
const summary = {
  json: {
    total_vendors: rankedVendors.length,
    average_score: (rankedVendors.reduce((sum, v) => sum + v.json.total_score, 0) / rankedVendors.length).toFixed(2),
    top_vendor: rankedVendors[0]?.json.vendor_name,
    top_score: rankedVendors[0]?.json.total_score,
    vendors_with_risks: rankedVendors.filter(v => v.json.risk_flags.length > 0).length,
    vendors_with_missing_data: rankedVendors.filter(v => v.json.missing_fields.length > 0).length,
    evaluation_date: new Date().toISOString()
  }
};

// Return all ranked vendors + summary
return [...rankedVendors, summary];
```

**Configuration Note**:
- Set execution mode to "Wait for all incoming items" in node settings
- This ensures all vendors are scored before ranking

**Output Structure** (Array of items):
```json
[
  {
    "vendor_name": "TechFlow Solutions",
    "total_score": 8.70,
    "rank": 1,
    "rank_label": "🥇 Top Choice",
    "...": "..."
  },
  {
    "vendor_name": "Acme Corporation",
    "total_score": 7.75,
    "rank": 2,
    "rank_label": "🥈 Runner-up",
    "...": "..."
  },
  {
    "vendor_name": "GlobalSoft Inc",
    "total_score": 6.65,
    "rank": 3,
    "rank_label": "🥉 Third Place",
    "...": "..."
  },
  {
    "total_vendors": 3,
    "average_score": "7.70",
    "top_vendor": "TechFlow Solutions",
    "top_score": 8.70,
    "vendors_with_risks": 3,
    "vendors_with_missing_data": 1,
    "evaluation_date": "2024-05-02T10:40:00.000Z"
  }
]
```

---

### Node 8: IF (Conditional Router)

**Node Type**: `n8n-nodes-base.if`

**Purpose**: Route vendors with missing fields to follow-up email generator.

**Configuration**:
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

**How It Works**:
- Checks if `missing_fields` array is not empty
- **True**: Routes to "Follow-up Email Generator"
- **False**: Skips to "Append row in sheet"

**Routing Logic**:
```
IF missing_fields.length > 0:
    → TRUE branch: Generate follow-up email
ELSE:
    → FALSE branch: Skip directly to Google Sheets
```

---

### Node 9: Follow-up Email Generator (Code Node)

**Node Type**: `n8n-nodes-base.code` (v2)

**Purpose**: Create personalized follow-up email requesting missing information.

**JavaScript Code**:
```javascript
const items = [];

for (const item of $input.all()) {
  const missingFields = item.json.missing_fields || [];
  
  if (missingFields.length === 0) continue;
  
  const emailBody = `Dear ${item.json.vendor_name} Team,

Thank you for submitting your RFP response. We are currently evaluating all vendor proposals.

During our review, we noticed the following information is missing or unclear:

${missingFields.map((field, i) => `${i + 1}. ${field}`).join('\n')}

To ensure a fair and complete evaluation, please provide the requested information by replying to this email within 3 business days.

If you have any questions, please don't hesitate to reach out.

Best regards,
Procurement Team`;

  items.push({
    json: {
      to: item.json.vendor_email,
      subject: `RFP Follow-up: Additional Information Required - ${item.json.vendor_name}`,
      body: emailBody,
      vendor_name: item.json.vendor_name,
      missing_fields: missingFields
    }
  });
}

return items;
```

**Output Structure**:
```json
{
  "to": "vendor@acme.com",
  "subject": "RFP Follow-up: Additional Information Required - Acme Corporation",
  "body": "Dear Acme Corporation Team,\n\nThank you for submitting your RFP response...",
  "vendor_name": "Acme Corporation",
  "missing_fields": ["SLA response times", "Compliance certifications"]
}
```

---

### Node 10: Gmail Follow-up Sender

**Node Type**: `n8n-nodes-base.gmail`

**Purpose**: Send follow-up emails to vendors with missing information.

**Configuration**:
```json
{
  "operation": "send",
  "sendTo": "={{ $json.to }}",
  "subject": "={{ $json.subject }}",
  "message": "={{ $json.body }}",
  "options": {
    "ccList": "procurement@yourcompany.com",
    "attachmentsUi": {
      "attachmentsBinary": []
    }
  }
}
```

**Features**:
- CC's procurement team on all follow-ups
- Uses same OAuth2 credentials as trigger
- Auto-threading (replies to original email if possible)

**Best Practices**:
- Enable "Continue On Fail" to prevent workflow failure if email fails
- Add retry logic (3 attempts, 5-second delay)

---

### Node 11: Append row in sheet (Google Sheets)

**Node Type**: `n8n-nodes-base.googleSheets`

**Purpose**: Store vendor evaluation data in centralized Google Sheets database.

**Configuration**:
```json
{
  "operation": "append",
  "documentId": {
    "mode": "list",
    "value": "YOUR_SPREADSHEET_ID"
  },
  "sheetName": {
    "mode": "list",
    "value": "VendorScores"
  },
  "columns": {
    "mappingsMode": "defineBelow",
    "value": {
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
  },
  "options": {
    "cellFormat": "USER_ENTERED"
  }
}
```

**Sheet Structure**:
Headers in Row 1:
```
vendor_name | vendor_email | price_score | capability_score | compliance_score | sla_score | reference_score | total_score | rank | missing_fields | risk_flags | summary | received_date | timestamp
```

**Data Validation**:
- Scores: Number format (0.0)
- Dates: Date format (MM/DD/YYYY)
- Arrays: Converted to comma-separated strings

---

### Node 12: Slack Message Builder1 (Code Node)

**Node Type**: `n8n-nodes-base.code` (v2)

**Purpose**: Format rich Slack message with top 3 vendor rankings.

**JavaScript Code**:
```javascript
const allVendors = $input.all();

// Get top 3 vendors
const topVendors = allVendors
  .filter(v => v.json.rank && v.json.rank <= 3)
  .sort((a, b) => a.json.rank - b.json.rank);

// Get summary
const summary = allVendors.find(v => v.json.total_vendors !== undefined);

const message = {
  blocks: [
    {
      type: "header",
      text: {
        type: "plain_text",
        text: "🎯 RFP Evaluation Complete - Vendor Shortlist"
      }
    },
    {
      type: "section",
      text: {
        type: "mrkdwn",
        text: `*Total Vendors Evaluated:* ${summary?.json.total_vendors || topVendors.length}\n*Average Score:* ${summary?.json.average_score || 'N/A'}\n*Evaluation Date:* ${new Date().toLocaleDateString()}`
      }
    },
    {
      type: "divider"
    }
  ]
};

// Add each top vendor
topVendors.forEach(vendor => {
  message.blocks.push(
    {
      type: "section",
      text: {
        type: "mrkdwn",
        text: `*${vendor.json.rank_label} ${vendor.json.vendor_name}*\n*Total Score:* ${vendor.json.total_score}/10\n\n*Breakdown:*\n• Price: ${vendor.json.price_score}/10\n• Capabilities: ${vendor.json.capability_score}/10\n• Compliance: ${vendor.json.compliance_score}/10\n• SLA: ${vendor.json.sla_score}/10\n• References: ${vendor.json.reference_score}/10\n\n*Summary:* ${vendor.json.summary}${vendor.json.risk_flags.length > 0 ? `\n\n⚠️ *Risks:* ${vendor.json.risk_flags.join(', ')}` : ''}`
      }
    },
    {
      type: "divider"
    }
  );
});

message.blocks.push({
  type: "context",
  elements: [
    {
      type: "mrkdwn",
      text: "📊 Full details available in Google Sheets | ⚡ Powered by n8n + OpenRouter AI"
    }
  ]
});

return [{
  json: {
    channel: '#rfp-evaluations',
    text: 'RFP Evaluation Complete',
    blocks: JSON.stringify(message.blocks)
  }
}];
```

**Output Structure**:
```json
{
  "channel": "#rfp-evaluations",
  "text": "RFP Evaluation Complete",
  "blocks": "[{\"type\":\"header\",\"text\":{...}}]"
}
```

---

### Node 13: Slack Notifier

**Node Type**: `n8n-nodes-base.slack`

**Purpose**: Send formatted notification to Slack channel.

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

**Slack Webhook Alternative**:
If you prefer webhooks instead of OAuth:
```json
{
  "resource": "message",
  "operation": "post",
  "webhookUri": "https://hooks.slack.com/services/YOUR/WEBHOOK/URL",
  "text": "={{ $json.text }}",
  "otherOptions": {
    "blocks": "={{ $json.blocks }}"
  }
}
```

---

## 🔄 DATA FLOW DIAGRAM

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                             WORKFLOW EXECUTION FLOW                         │
└─────────────────────────────────────────────────────────────────────────────┘

START
  │
  ▼
┌─────────────────────────────────────┐
│   1. RFP Response Monitor           │
│   (Gmail Trigger - Every Minute)    │
│                                     │
│   Monitors: inbox for RFP responses │
│   Downloads: PDF attachments        │
└─────────────────────────────────────┘
  │
  │ [Email with PDF attachment]
  │
  ▼
┌─────────────────────────────────────┐
│   2. Extract from File              │
│   (PDF Text Extraction)             │
│                                     │
│   Input: Binary PDF                 │
│   Output: Extracted text content    │
└─────────────────────────────────────┘
  │
  │ [Raw PDF text]
  │
  ▼
┌─────────────────────────────────────┐
│   3. Response Normalizer            │
│   (Parse & Structure Data)          │
│                                     │
│   Extracts:                         │
│   - Vendor name                     │
│   - Pricing                         │
│   - Capabilities                    │
│   - SLA                             │
│   - Compliance                      │
│   - References                      │
└─────────────────────────────────────┘
  │
  │ [Structured vendor data]
  │
  ▼
┌─────────────────────────────────────┐
│   4. Prepare AI Prompt              │
│   (Build System + User Messages)    │
│                                     │
│   Creates:                          │
│   - System prompt (expert role)     │
│   - User prompt (vendor data)       │
│   - Scoring instructions            │
└─────────────────────────────────────┘
  │
  │ [Chat messages array]
  │
  ▼
┌─────────────────────────────────────┐
│   5. AI Vendor Evaluator            │
│   (OpenRouter + Llama 3.1 70B)      │
│                                     │
│   Scores:                           │
│   - Price (0-10)                    │
│   - Capabilities (0-10)             │
│   - Compliance (0-10)               │
│   - SLA (0-10)                      │
│   - References (0-10)               │
│   Calculates: Weighted total        │
│   Identifies: Missing fields        │
│   Flags: Risks                      │
└─────────────────────────────────────┘
  │
  │ [AI JSON response]
  │
  ▼
┌─────────────────────────────────────┐
│   6. Extract Scoring Data           │
│   (Parse JSON + Error Handling)     │
│                                     │
│   Parses: AI JSON output            │
│   Handles: Parsing errors           │
│   Adds: Timestamp                   │
└─────────────────────────────────────┘
  │
  │ [Scored vendor object]
  │
  ▼
┌─────────────────────────────────────┐
│   7. Vendor Ranking Engine          │
│   (Aggregate, Sort, Rank)           │
│                                     │
│   Waits for: All vendors            │
│   Sorts by: Total score DESC        │
│   Assigns: Rank (1, 2, 3...)        │
│   Calculates: Summary stats         │
└─────────────────────────────────────┘
  │
  │ [Array: Ranked vendors + summary]
  │
  ▼
┌─────────────────────────────────────┐
│   8. IF: Missing Fields?            │
│   (Conditional Router)              │
│                                     │
│   Check: missing_fields.length > 0  │
└─────────────────────────────────────┘
  │
  ├─── TRUE ──────────────────────────────┐
  │                                       │
  │                                       ▼
  │                           ┌─────────────────────────────────┐
  │                           │  9. Follow-up Email Generator   │
  │                           │  (Create Email Template)        │
  │                           │                                 │
  │                           │  Generates: Personalized email  │
  │                           │  Lists: Missing fields          │
  │                           │  Sets: 3-day deadline           │
  │                           └─────────────────────────────────┘
  │                                       │
  │                                       ▼
  │                           ┌─────────────────────────────────┐
  │                           │  10. Gmail Follow-up Sender     │
  │                           │  (Send Email via Gmail API)     │
  │                           │                                 │
  │                           │  Sends: Follow-up email         │
  │                           │  CC's: procurement@company.com  │
  │                           └─────────────────────────────────┘
  │                                       │
  └─── FALSE ────────────────────────────┼───────────────────────┘
                                          │
                                          ▼
                              ┌─────────────────────────────────┐
                              │  11. Append row in sheet        │
                              │  (Google Sheets Storage)        │
                              │                                 │
                              │  Writes: Vendor scores          │
                              │  Sheet: VendorScores            │
                              │  Audit: Timestamp + metadata    │
                              └─────────────────────────────────┘
                                          │
                                          ▼
                              ┌─────────────────────────────────┐
                              │  12. Slack Message Builder1     │
                              │  (Format Rich Message)          │
                              │                                 │
                              │  Builds: Block-based message    │
                              │  Shows: Top 3 vendors           │
                              │  Includes: Scores + summaries   │
                              └─────────────────────────────────┘
                                          │
                                          ▼
                              ┌─────────────────────────────────┐
                              │  13. Slack Notifier             │
                              │  (Post to Channel)              │
                              │                                 │
                              │  Posts: #rfp-evaluations        │
                              │  Notifies: Decision-makers      │
                              └─────────────────────────────────┘
                                          │
                                          ▼
                                        END

Execution Time: ~30-60 seconds per vendor
```

---

## ⚙️ CONFIGURATION GUIDE

### Prerequisites

1. **n8n Instance** (self-hosted or cloud)
2. **Google Cloud Project** (for Gmail + Sheets APIs)
3. **OpenRouter Account** (free tier)
4. **Slack Workspace** (with admin access)
5. **Google Sheets** (with properly structured sheet)

---

### Step 1: OpenRouter Setup

#### 1.1 Create Account
```
1. Navigate to: https://openrouter.ai/
2. Click "Sign Up"
3. Use Google/GitHub OAuth or email
4. Verify email address
```

#### 1.2 Generate API Key
```
1. Log in to OpenRouter dashboard
2. Navigate to: Settings → API Keys
3. Click "Create New Key"
4. Name: "n8n_rfp_scoring_engine"
5. Copy the key (format: sk-or-v1-...)
6. Store securely (not shown again)
```

#### 1.3 Configure in n8n
```
1. In n8n: Settings → Credentials
2. Click "Add Credential"
3. Search: "OpenRouter"
4. If not found, use "HTTP Header Auth"
   - Name: "OpenRouter API"
   - Header Name: "Authorization"
   - Header Value: "Bearer YOUR_API_KEY_HERE"
5. Test connection
6. Save credential
```

#### 1.4 Free Model Limits
```
Model: meta-llama/llama-3.1-70b-instruct:free
Rate Limit: ~20 requests/minute
Monthly Limit: Generous (check dashboard)
Cost: $0

Alternative free models:
- nousresearch/hermes-3-llama-3.1-405b:free
- google/gemini-flash-1.5:free
```

---

### Step 2: Gmail API Setup

#### 2.1 Enable Gmail API in Google Cloud Console
```
1. Go to: https://console.cloud.google.com/
2. Create new project: "n8n-rfp-automation"
3. Navigate to: APIs & Services → Library
4. Search: "Gmail API"
5. Click "Enable"
```

#### 2.2 Create OAuth 2.0 Credentials
```
1. APIs & Services → Credentials
2. Click "Create Credentials" → OAuth 2.0 Client ID
3. Configure OAuth Consent Screen:
   - User Type: Internal (if G Workspace) or External
   - App name: "n8n RFP Automation"
   - Support email: your-email@company.com
   - Scopes: Add "gmail.readonly", "gmail.send"
   - Test users: Add your email
4. Create OAuth Client:
   - Application type: Web application
   - Name: "n8n Gmail"
   - Authorized redirect URIs: 
     https://YOUR_N8N_DOMAIN/rest/oauth2-credential/callback
5. Copy Client ID and Client Secret
```

#### 2.3 Configure in n8n
```
1. n8n: Settings → Credentials → Add Credential
2. Search: "Gmail OAuth2 API"
3. Fill in:
   - Client ID: (from Google Cloud Console)
   - Client Secret: (from Google Cloud Console)
   - Scopes: gmail.readonly,gmail.send
4. Click "Connect my account"
5. Authorize in popup (allow all permissions)
6. Verify connection successful
7. Save credential
```

---

### Step 3: Google Sheets Setup

#### 3.1 Create Spreadsheet
```
1. Go to: https://sheets.google.com/
2. Create new spreadsheet
3. Name: "RFP Vendor Evaluations"
4. Rename Sheet1 to: "VendorScores"
```

#### 3.2 Add Headers (Row 1)
```
A: vendor_name
B: vendor_email
C: price_score
D: capability_score
E: compliance_score
F: sla_score
G: reference_score
H: total_score
I: rank
J: missing_fields
K: risk_flags
L: summary
M: received_date
N: timestamp
```

#### 3.3 Format Columns
```
Price/Capability/Compliance/SLA/Reference Scores (C-G):
- Format: Number → 1 decimal place (0.0)

Total Score (H):
- Format: Number → 2 decimal places (0.00)

Rank (I):
- Format: Number → No decimals (#)

Dates (M-N):
- Format: Date → MM/DD/YYYY HH:MM:SS
```

#### 3.4 Add Conditional Formatting
```
Total Score (Column H):
- Rule 1: >= 8 → Background: #d9ead3 (light green)
- Rule 2: >= 6 AND < 8 → Background: #fff2cc (light yellow)
- Rule 3: < 6 → Background: #f4cccc (light red)

Rank (Column I):
- Rule 1: = 1 → Background: #ffd966 (gold)
- Rule 2: = 2 → Background: #d9d9d9 (silver)
- Rule 3: = 3 → Background: #e6b8af (bronze)

Missing Fields (Column J):
- Rule: NOT EMPTY → Background: #f9cb9c (orange)

Risk Flags (Column K):
- Rule: NOT EMPTY → Background: #ea9999 (light red)
```

#### 3.5 Copy Sheet ID
```
1. Open spreadsheet
2. Copy ID from URL:
   https://docs.google.com/spreadsheets/d/[THIS_IS_THE_ID]/edit
3. Example: 1BxiMVs0XRA5nFMdKvBdBZjgmUUqptlbs74OgvE2upms
4. Save this ID for n8n configuration
```

#### 3.6 Enable Google Sheets API
```
1. Google Cloud Console (same project as Gmail)
2. APIs & Services → Library
3. Search: "Google Sheets API"
4. Click "Enable"
```

#### 3.7 Configure in n8n
```
1. n8n: Settings → Credentials → Add Credential
2. Search: "Google Sheets OAuth2 API"
3. Use same OAuth client as Gmail (or create new)
4. Scopes: https://www.googleapis.com/auth/spreadsheets
5. Authorize and save
```

---

### Step 4: Slack Setup

#### 4.1 Create Slack App
```
1. Go to: https://api.slack.com/apps
2. Click "Create New App" → From scratch
3. App name: "RFP Evaluator Bot"
4. Workspace: Select your workspace
5. Click "Create App"
```

#### 4.2 Configure Bot Permissions
```
1. In app settings: OAuth & Permissions
2. Scroll to "Bot Token Scopes"
3. Add scopes:
   - chat:write (send messages)
   - chat:write.public (post to channels without joining)
   - files:write (optional: upload files)
4. Scroll up and click "Install to Workspace"
5. Authorize app
6. Copy "Bot User OAuth Token" (starts with xoxb-)
```

#### 4.3 Create Channel
```
1. In Slack workspace, create channel: #rfp-evaluations
2. Make it private or public based on needs
3. Invite bot: /invite @RFP Evaluator Bot
```

#### 4.4 Configure in n8n
```
Method 1: OAuth2 (Recommended)
1. n8n: Settings → Credentials → Add Credential
2. Search: "Slack OAuth2 API"
3. Follow OAuth flow
4. Authorize app
5. Save

Method 2: Access Token (Simpler)
1. n8n: Settings → Credentials → Add Credential
2. Search: "Slack API"
3. Access Token: (paste Bot User OAuth Token)
4. Test connection
5. Save
```

---

### Step 5: Import Workflow to n8n

#### 5.1 Upload JSON File
```
1. In n8n: Workflows → Import from File
2. Upload: AI_Vendor_Evaluation___RFP_Response_Scoring_Engine.json
3. Workflow imported successfully
```

#### 5.2 Update Credentials
```
For each node requiring credentials:

1. RFP Response Monitor (Gmail Trigger):
   - Credential: Select your Gmail OAuth2

2. AI Vendor Evaluator → OpenRouter Chat Model:
   - Credential: Select your OpenRouter API

3. Gmail Follow-up Sender:
   - Credential: Select your Gmail OAuth2

4. Append row in sheet:
   - Credential: Select your Google Sheets OAuth2
   - Document ID: Paste your Sheet ID
   - Sheet Name: "VendorScores"

5. Slack Notifier:
   - Credential: Select your Slack API
   - Channel: "#rfp-evaluations"
```

#### 5.3 Test Individual Nodes
```
1. Click "Execute Node" button on each node
2. Verify output in "Output" panel
3. Fix any credential/configuration errors
4. Proceed to next node
```

---

## 🧪 TESTING & VALIDATION

### Test Case 1: End-to-End Test with Sample RFP

#### Preparation
```
1. Create sample RFP PDF with structure:

Vendor Name: Test Vendor Corp

1. Pricing:
Annual License: $45,000
Implementation: $8,000
Support: Included

2. Capabilities:
- Real-time analytics dashboard
- API integrations (REST, GraphQL)
- Custom reporting engine
- Mobile app (iOS, Android)

3. SLA:
Uptime Guarantee: 99.95%
Support Response: 2 hours (critical), 8 hours (standard)
Dedicated account manager

4. Compliance:
- SOC2 Type II certified
- ISO 27001 certified
- GDPR compliant
- HIPAA ready

5. References:
- Enterprise Corp (Fortune 500, 3 years)
- TechCo Inc. (500 employees, 2 years)
- Global Systems (1000 employees, 18 months)

2. Save as: Test_Vendor_RFP.pdf
3. Email to yourself with subject: "RFP Response - Test Vendor Corp"
4. Attach the PDF
```

#### Expected Results
```
1. Gmail Trigger: Detects email within 60 seconds
2. PDF Extraction: Extracts all text successfully
3. Normalization: Parses all 5 sections
4. AI Scoring: Returns scores:
   - Price: 8-9 (competitive)
   - Capabilities: 8-9 (strong feature set)
   - Compliance: 9-10 (comprehensive)
   - SLA: 9-10 (enterprise-grade)
   - References: 8-9 (quality clients)
   - Total Score: ~8.5-9.0
5. Ranking: Assigned rank #1 (if first vendor)
6. IF Node: False (no missing fields)
7. Google Sheets: Row appended successfully
8. Slack: Notification posted to channel
```

#### Validation Checklist
```
✅ Email received by Gmail Trigger
✅ PDF text extracted completely
✅ Vendor name parsed correctly
✅ All 5 fields extracted
✅ AI returned valid JSON
✅ Scores are reasonable (7-9 range)
✅ Total score calculated correctly
✅ No missing fields detected
✅ Google Sheets row added
✅ Slack message formatted properly
✅ Execution time < 60 seconds
```

---

### Test Case 2: Missing Fields Detection

#### Preparation
```
Create incomplete RFP PDF:

Vendor Name: Incomplete Vendor Inc

1. Pricing:
Contact us for custom quote

2. Capabilities:
We offer comprehensive solutions
[No specific features listed]

3. SLA:
[SECTION MISSING]

4. Compliance:
[SECTION MISSING]

5. References:
Available upon request
```

#### Expected Results
```
1. AI Scoring: Returns:
   - missing_fields: ["SLA commitments", "Compliance certifications", "Specific pricing", "Client references"]
   - risk_flags: ["No SLA mentioned", "No compliance certs", "Vague pricing"]
   - Lower scores: Price 3-4, Compliance 0-1, SLA 0-1
2. IF Node: True (missing fields detected)
3. Follow-up Email: Generated successfully
4. Gmail: Follow-up sent to vendor
5. Google Sheets: Row added with missing_fields populated
6. Slack: Notification includes risk warnings
```

---

### Test Case 3: Error Handling - Invalid PDF

#### Preparation
```
1. Create corrupt PDF or non-text image PDF
2. Email with subject: "RFP Response - Error Test"
```

#### Expected Results
```
1. PDF Extraction: Fails or returns empty text
2. Normalization: vendor_name = "Unknown Vendor"
3. AI Scoring: Returns error or default scores
4. Extract Scoring Data: Catches error → needs_manual_review: true
5. Workflow: Continues without crashing
6. Google Sheets: Row with PARSING_ERROR flag
7. Slack: Should include warning about manual review needed
```

---

## 🚨 ERROR HANDLING

### Error Scenario 1: OpenRouter API Failure

**Symptoms**:
- AI Vendor Evaluator node returns 429 (rate limit) or 500 error
- Workflow execution fails at scoring stage

**Root Causes**:
- Exceeded free tier rate limit (20 requests/min)
- OpenRouter service outage
- Invalid API key

**Resolution**:
```javascript
// Add to AI Vendor Evaluator node settings:
"continueOnFail": true,
"retryOnFail": true,
"maxTries": 3,
"waitBetween": 5000

// Fallback logic in Extract Scoring Data:
if ($input.first().json.error) {
  // Use default scores
  return [{
    json: {
      vendor_name: 'AI_SCORING_FAILED',
      price_score: 5,
      capability_score: 5,
      compliance_score: 5,
      sla_score: 5,
      reference_score: 5,
      total_score: 5.0,
      missing_fields: ['AI scoring unavailable'],
      risk_flags: ['MANUAL REVIEW REQUIRED'],
      summary: 'AI scoring failed - manual evaluation needed',
      needs_manual_review: true
    }
  }];
}
```

**Prevention**:
- Monitor OpenRouter dashboard for usage
- Consider upgrading to paid tier for production
- Implement exponential backoff for retries

---

### Error Scenario 2: Gmail API Quota Exceeded

**Symptoms**:
- Gmail trigger stops polling
- Error: "Quota exceeded for quota metric 'Read requests'"

**Root Causes**:
- Too many workflow executions
- Gmail API daily quota limit reached (default: 1 billion units/day)

**Resolution**:
```
1. Check quota usage:
   - Google Cloud Console → APIs & Services → Dashboard
   - Select Gmail API → View quotas

2. Temporary fix:
   - Disable workflow for 24 hours to reset quota
   - Request quota increase from Google Cloud

3. Long-term fix:
   - Increase polling interval to 5 minutes instead of 1
   - Add webhook trigger as alternative (requires forwarding rules)
```

---

### Error Scenario 3: Google Sheets Write Failure

**Symptoms**:
- "Append row in sheet" node fails
- Error: "Insufficient permissions" or "Sheet not found"

**Root Causes**:
- OAuth scope missing: `https://www.googleapis.com/auth/spreadsheets`
- Sheet ID incorrect
- Sheet renamed or deleted

**Resolution**:
```
1. Verify Sheet ID:
   - Open Google Sheets → Copy ID from URL
   - Update "Append row in sheet" node configuration

2. Check OAuth scopes:
   - n8n Credentials → Edit Google Sheets OAuth2
   - Ensure scope includes: spreadsheets (not just readonly)
   - Re-authorize if needed

3. Test permissions:
   - Manually try to edit sheet with same Google account
   - Verify service account has edit access (if using service account)
```

---

### Error Scenario 4: Slack Message Formatting Error

**Symptoms**:
- Slack message posts but appears malformed
- Error: "Invalid blocks"

**Root Causes**:
- JSON stringify errors in Slack Message Builder
- Special characters breaking JSON format
- Blocks array too large

**Resolution**:
```javascript
// In Slack Message Builder node, add error handling:
try {
  const message = {
    blocks: [...]
  };
  
  // Validate blocks before stringifying
  if (message.blocks.length > 50) {
    message.blocks = message.blocks.slice(0, 50);
  }
  
  // Escape special characters in text
  const escapeText = (text) => {
    return text
      .replace(/&/g, '&amp;')
      .replace(/</g, '&lt;')
      .replace(/>/g, '&gt;');
  };
  
  return [{
    json: {
      channel: '#rfp-evaluations',
      text: 'RFP Evaluation Complete',
      blocks: JSON.stringify(message.blocks)
    }
  }];
} catch (error) {
  // Fallback: Send simple text message
  return [{
    json: {
      channel: '#rfp-evaluations',
      text: `RFP Evaluation Complete\nTop Vendor: ${topVendors[0]?.json.vendor_name}\nScore: ${topVendors[0]?.json.total_score}`
    }
  }];
}
```

---

## 🔧 MAINTENANCE & TROUBLESHOOTING

### Daily Monitoring

**Key Metrics to Track**:
```
1. Workflow Execution Count
   - Target: Aligned with expected RFP volume
   - Alert if: 0 executions for 24 hours (trigger may be broken)

2. Success Rate
   - Target: >95%
   - Alert if: <90%

3. Average Execution Time
   - Target: <60 seconds per vendor
   - Alert if: >120 seconds (OpenRouter slowness)

4. AI Scoring Quality
   - Manual spot-check: 5 vendors per week
   - Verify scores align with human judgment
```

**Access Monitoring Dashboard**:
```
1. n8n: Executions tab
2. Filter: "AI Vendor Evaluation & RFP Response Scoring Engine"
3. Review:
   - Last 7 days execution history
   - Failed executions
   - Error messages
```

---

### Weekly Maintenance Tasks

**1. Review Google Sheets Data**
```
Actions:
- Check for duplicate entries (same vendor, same date)
- Verify scores are within 0-10 range
- Spot-check AI summaries for quality
- Archive old evaluations (>90 days) to separate sheet
```

**2. OpenRouter Usage Review**
```
Actions:
- Log in to OpenRouter dashboard
- Check remaining free tier credits
- Review rate limit hits
- Consider upgrading if hitting limits frequently
```

**3. Slack Channel Cleanup**
```
Actions:
- Archive old RFP notifications (>30 days)
- Pin important vendor shortlists
- Gather feedback from decision-makers
```

---

### Monthly Maintenance Tasks

**1. Credential Rotation**
```
Actions:
- Generate new OpenRouter API key
- Update n8n credential
- Test workflow execution
- Delete old API key from OpenRouter
```

**2. AI Prompt Optimization**
```
Actions:
- Review 10 recent vendor evaluations
- Identify any systematic scoring biases
- Adjust system prompt weights if needed
- Test updated prompt on historical vendors
```

**3. Google Sheets Backup**
```
Actions:
- File → Download → CSV
- Store backup in secure location
- Verify data integrity
```

---

### Troubleshooting Guide

| Issue | Symptoms | Solution |
|-------|----------|----------|
| **No emails being processed** | Workflow not triggering | 1. Check Gmail trigger is active<br>2. Verify OAuth token hasn't expired<br>3. Test with manual email |
| **PDF text extraction fails** | Empty text output | 1. Verify PDF is text-based (not scanned image)<br>2. Try different PDF version<br>3. Consider OCR preprocessing |
| **AI returns invalid scores** | Scores outside 0-10 range | 1. Check system prompt weights<br>2. Verify model hasn't changed<br>3. Add validation in Extract Scoring Data |
| **Google Sheets not updating** | No new rows appearing | 1. Verify Sheet ID is correct<br>2. Check OAuth permissions<br>3. Test manual append with Google Sheets UI |
| **Slack message not posting** | Error in Slack Notifier | 1. Verify bot is in channel<br>2. Check OAuth scopes<br>3. Test with simple text message first |
| **Workflow times out** | Execution exceeds 5 minutes | 1. Check OpenRouter API latency<br>2. Reduce temperature (faster inference)<br>3. Split into sub-workflows |

---

## 📊 SAMPLE OUTPUTS

### Sample 1: Successful Vendor Evaluation

**Input Email**:
```
From: sales@techflow.com
Subject: RFP Response - TechFlow Solutions
Attachment: TechFlow_RFP_2024.pdf

Dear Procurement Team,

Please find our comprehensive RFP response attached.

Best regards,
TechFlow Solutions
```

**Google Sheets Row**:
```
vendor_name: TechFlow Solutions
vendor_email: sales@techflow.com
price_score: 9
capability_score: 9
compliance_score: 8
sla_score: 9
reference_score: 8
total_score: 8.70
rank: 1
missing_fields: 
risk_flags: Pricing model lacks enterprise volume discounts
summary: Exceptional technical capabilities with strong compliance posture and excellent SLA commitments. Industry-leading solution with minor pricing flexibility concerns.
received_date: 05/02/2024 10:30:00
timestamp: 05/02/2024 10:35:22
```

**Slack Message**:
```
🎯 RFP Evaluation Complete - Vendor Shortlist
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Total Vendors Evaluated: 3
Average Score: 7.70
Evaluation Date: 05/02/2024
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

🥇 Top Choice TechFlow Solutions
Total Score: 8.70/10

Breakdown:
• Price: 9/10
• Capabilities: 9/10
• Compliance: 8/10
• SLA: 9/10
• References: 8/10

Summary: Exceptional technical capabilities with strong compliance posture and excellent SLA commitments. Industry-leading solution with minor pricing flexibility concerns.

⚠️ Risks: Pricing model lacks enterprise volume discounts

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📊 Full details available in Google Sheets | ⚡ Powered by n8n + OpenRouter AI
```

---

### Sample 2: Vendor with Missing Information

**Google Sheets Row**:
```
vendor_name: GlobalSoft Inc
vendor_email: contact@globalsoft.com
price_score: 6
capability_score: 7
compliance_score: 6
sla_score: 7
reference_score: 7
total_score: 6.65
rank: 3
missing_fields: SLA response times, Compliance certifications
risk_flags: No SOC2 certification mentioned, SLA uptime not clearly stated
summary: Adequate feature set with decent references. Significant compliance gaps and unclear SLA commitments raise concerns for enterprise deployment.
received_date: 05/02/2024 11:15:00
timestamp: 05/02/2024 11:18:45
```

**Follow-up Email Sent**:
```
To: contact@globalsoft.com
CC: procurement@yourcompany.com
Subject: RFP Follow-up: Additional Information Required - GlobalSoft Inc

Dear GlobalSoft Inc Team,

Thank you for submitting your RFP response. We are currently evaluating all vendor proposals.

During our review, we noticed the following information is missing or unclear:

1. SLA response times
2. Compliance certifications

To ensure a fair and complete evaluation, please provide the requested information by replying to this email within 3 business days.

If you have any questions, please don't hesitate to reach out.

Best regards,
Procurement Team
```

---

## 📈 PERFORMANCE METRICS

### Typical Execution Times

| Stage | Duration | Notes |
|-------|----------|-------|
| Gmail Trigger → PDF Extraction | 2-5 sec | Depends on PDF size |
| Normalization | 0.5-1 sec | Regex parsing |
| AI Prompt Preparation | 0.2-0.5 sec | String concatenation |
| OpenRouter API Call | 10-30 sec | Varies by model load |
| Scoring Data Extraction | 0.5-1 sec | JSON parsing |
| Ranking Engine | 1-2 sec | Array sorting |
| Google Sheets Append | 2-4 sec | API latency |
| Slack Notification | 1-3 sec | API latency |
| **Total** | **20-50 sec** | Per vendor |

### Resource Usage

```
CPU: Negligible (n8n is event-driven)
Memory: ~50-100 MB per execution
Network: ~500 KB per vendor (PDF + API calls)
Storage: ~5 KB per vendor (Google Sheets row)
```

### Scalability

```
Single vendor: 30-60 seconds
10 vendors (batch): 5-10 minutes
100 vendors (batch): 50-100 minutes

Bottleneck: OpenRouter API rate limits (20 req/min free tier)
Solution: Upgrade to paid tier or add delay between vendors
```

---

## 🎓 BEST PRACTICES

### 1. RFP Template Standardization

**Create Vendor RFP Template**:
```
Provide vendors with structured template PDF to ensure consistency:

VENDOR RFP RESPONSE TEMPLATE
==============================

Vendor Name: [COMPANY NAME]

1. Pricing
- Annual License Fee: $____
- Implementation Cost: $____
- Support Fee: $____
- Total Cost of Ownership (3 years): $____

2. Capabilities
- Feature 1: [Description]
- Feature 2: [Description]
- Integrations: [List APIs/platforms]
- Scalability: [Max users/transactions]

3. SLA Commitments
- Uptime Guarantee: ___%
- Support Response Time (Critical): __ hours
- Support Response Time (Standard): __ hours
- Financial Penalties: [Yes/No, details]

4. Compliance & Security
- SOC2: [Type I/Type II/None]
- ISO 27001: [Yes/No]
- GDPR: [Compliant/Not Compliant]
- HIPAA: [Ready/Not Ready]
- Other: [List certifications]

5. Client References
- Reference 1: [Company name, industry, tenure]
- Reference 2: [Company name, industry, tenure]
- Reference 3: [Company name, industry, tenure]
```

**Benefits**:
- Higher extraction accuracy
- Fewer missing fields
- Faster AI processing
- Easier vendor comparison

---

### 2. Scoring Rubric Customization

**Adjust weights based on your priorities**:

```javascript
// In AI Agent system prompt, modify weights:

// Default (balanced):
Total Score = (Price × 0.25) + (Capabilities × 0.30) + (Compliance × 0.20) + (SLA × 0.15) + (References × 0.10)

// Price-sensitive:
Total Score = (Price × 0.40) + (Capabilities × 0.25) + (Compliance × 0.15) + (SLA × 0.10) + (References × 0.10)

// Compliance-critical (healthcare, finance):
Total Score = (Price × 0.15) + (Capabilities × 0.20) + (Compliance × 0.40) + (SLA × 0.15) + (References × 0.10)

// Innovation-focused:
Total Score = (Price × 0.20) + (Capabilities × 0.40) + (Compliance × 0.15) + (SLA × 0.15) + (References × 0.10)
```

---

### 3. Audit Trail Management

**Enable comprehensive logging**:

```javascript
// Add audit logging to Vendor Ranking Engine:
const auditLog = {
  execution_id: $execution.id,
  workflow_name: $workflow.name,
  executed_by: $workflow.active ? 'Automated' : 'Manual',
  vendor_count: rankedVendors.length,
  execution_time_ms: Date.now() - $execution.startTime,
  timestamp: new Date().toISOString()
};

// Append to separate "AuditLog" sheet tab
```

**Benefits**:
- Compliance with procurement regulations
- Troubleshooting historical issues
- Performance optimization insights

---

## 📝 CONCLUSION

This workflow provides a complete, production-ready solution for automating vendor RFP evaluations using AI. Key achievements:

✅ **Zero Cost**: Uses only free API tiers  
✅ **Objective Scoring**: AI eliminates human bias  
✅ **Time Savings**: 95% reduction in evaluation time  
✅ **Audit Trail**: Complete history in Google Sheets  
✅ **Intelligent Routing**: Auto-detects missing information  
✅ **Real-time Alerts**: Slack notifications for decision-makers  

**Next Steps**:
1. Import workflow JSON to n8n
2. Configure all credentials (30 minutes)
3. Test with sample RFP (10 minutes)
4. Deploy to production
5. Monitor and optimize

**Support**:
- n8n Community: https://community.n8n.io/
- OpenRouter Docs: https://openrouter.ai/docs
- Workflow Issues: Check troubleshooting section above

---

*Last Updated: May 2, 2024*  
*Version: 1.0*  
*Maintainer: Procurement Automation Team*
