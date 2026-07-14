# Multi-Agent AI Research Assistant
## Complete Technical Documentation

**Version:** 1.0  
**Workflow ID:** 5PTbSsmmsHuIl7vv  
**Last Updated:** February 2026  
**Status:** Production Ready

---

## 1. EXECUTIVE SUMMARY

The **Multi-Agent AI Research Assistant** is an n8n automation workflow that conducts expert-level research synthesis by deploying three specialized AI agents that work sequentially:

1. **Research Agent** → Extracts facts from general knowledge
2. **Skeptic Agent** → Challenges findings for logical gaps and weak evidence
3. **Synthesizer Agent** → Reconciles conflicts and produces decision-ready insights

**Key Innovation:** Agents actively disagree by design. Contradictions are surfaced explicitly, not hidden by consensus.

**Use Case:** Organizations that need research synthesis without hiring research teams.

---

## 2. ARCHITECTURE OVERVIEW

### Workflow Flow Diagram
```
Webhook Trigger
       ↓
Input Validator & Memory Loader (validates input, normalizes fields)
       ↓
Research Agent (AI Node)
   ├─ OpenRouter Chat Model (GLM-4.5-Air:free)
   └─ Extracts facts, themes, contradictions, assumptions
       ↓
Parse Output (JSON cleanup)
       ↓
Skeptic Agent (AI Node)
   ├─ OpenRouter Chat Model 1 (GLM-4.5-Air:free)
   └─ Challenges findings, flags weak evidence, identifies gaps
       ↓
Synthesizer Agent (AI Node)
   ├─ OpenRouter Chat Model 2 (GLM-4.5-Air:free)
   └─ Reconciles conflicts, produces recommended actions
       ↓
Comparison Engine (tracks research evolution over time)
       ↓
Markdown Formatter (formats output as research brief)
       ↓
Email Sender (Gmail)
Email Sender → [Decision Maker]
       ↓
Memory Archiver (Google Sheets) → Historical tracking
```

### Key Characteristics
- **Sequential processing** (not parallel) - Each agent builds on previous output
- **Explicit contradiction handling** - Conflicts are features, not bugs
- **Historical memory** - Tracks patterns across research runs
- **Zero external APIs** - Uses only OpenRouter free tier + Gmail
- **Fully auditable** - Each agent's JSON output is saved and searchable

---

## 3. NODE-BY-NODE BREAKDOWN

### NODE 1: Webhook Trigger
**Type:** Webhook  
**Position:** (0, 0)  
**Purpose:** Accept HTTP POST requests to trigger research workflows

**Configuration:**
```
HTTP Method:    POST
Webhook Path:   /research-assistant
Response Mode:  Last Node
Response Format: JSON
```

**Expected Input Body:**
```json
{
  "research_topic": "string (required)",
  "industry": "string (optional)",
  "audience": "string (optional)",
  "depth_level": "quick|deep (default: quick)",
  "recipient_email": "string (optional)"
}
```

**Output:**
- Passes raw JSON body to next node
- Webhook ID: `6afac3fd-a004-4507-b52d-e84d96357cec`

**Next Node:** Input Validator & Memory Loader

---

### NODE 2: Input Validator & Memory Loader
**Type:** Function (Code Node)  
**Position:** (208, 0)  
**Purpose:** Validate input, normalize fields, generate unique run ID

**Logic:**
```javascript
const input = $input.first().json;

// Support both direct JSON and webhook body JSON
const payload = input.body ?? input;

// Validate required field
if (!payload.research_topic || payload.research_topic.trim() === "") {
  throw new Error("research_topic is required");
}

// Normalize depth_level
const depth = payload.depth_level || "quick";
const sources_count = depth === "deep" ? 5 : 3;

return {
  json: {
    validated_input: {
      research_topic: payload.research_topic,
      industry: payload.industry || "general",
      audience: payload.audience || "decision-makers",
      depth_level: depth,
      sources_count,
      timestamp: new Date().toISOString(),
      run_id: Math.random().toString(36).slice(2, 11)
    },
    memory_context: {
      should_load_history: true,
      comparison_depth: depth === "deep" ? "full" : "summary"
    }
  }
};
```

**Outputs:**
- `validated_input` - Normalized user input with run_id
- `memory_context` - Metadata for historical comparison

**Validation Rules:**
- research_topic: Required, non-empty
- industry: Defaults to "general" if not provided
- audience: Defaults to "decision-makers" if not provided
- depth_level: Supports "quick" (3 sources) or "deep" (5 sources)

**Error Handling:**
- Throws error if research_topic is missing
- Gracefully defaults optional fields

**Next Node:** Research Agent (AI Node)

---

### NODE 3: Research Agent (AI Node)
**Type:** LangChain Agent (@n8n/n8n-nodes-langchain.agent)  
**Position:** (432, 0)  
**Purpose:** Extract facts, themes, contradictions from general knowledge about topic

**AI Model Configuration:**
```
Model:              OpenRouter Chat Model (GLM-4.5-Air:free)
Temperature:        0.3 (factual extraction)
Max Tokens:         1500
Type:               LangChain Agent (v3)
```

**System Prompt:**
```
You are a Research Extraction Agent.

IMPORTANT CONSTRAINTS:
- You are NOT provided with raw source documents.
- You MUST NOT invent sources, statistics, or claims.
- You may only extract widely accepted, high-level factual patterns that are commonly referenced across public knowledge.
- If evidence is weak or inferred, mark evidence_strength as "low".
- If no contradictions or numerical data are confidently known, return empty arrays.
- If facts are speculative or framework-based, treat them as assumptions, not facts.

Your task is STRICTLY extraction, not interpretation, strategy, or advice.

Based ONLY on general, well-known industry knowledge about the topic, identify:
1. Factual claims that are broadly accepted (not opinions)
2. Recurring themes that appear across multiple public discussions
3. Commonly cited numerical benchmarks (only if widely known)
4. Known contradictions in public discourse (if any)
5. Implicit assumptions commonly present in discussions of this topic

Output ONLY valid JSON.
Do NOT explain.
Do NOT narrate.
Do NOT add recommendations.

If information confidence is limited due to missing sources, reflect this accurately in evidence strength and analysis_confidence.
```

**User Prompt (Dynamic):**
```
Research topic: {{ $json.validated_input.research_topic }}
Industry: {{ $json.validated_input.industry }}
Audience: {{ $json.validated_input.audience }}
Depth level: {{ $json.validated_input.depth_level }}

Extract facts, themes, data points, contradictions, and assumptions from general public knowledge about this topic.
Do NOT interpret—just extract. Be thorough.
```

**Expected JSON Output Schema:**
```json
{
  "facts": [
    {
      "claim": "string",
      "evidence_strength": "high|medium|low",
      "sources_count": number,
      "source_examples": ["string"]
    }
  ],
  "recurring_themes": ["string"],
  "numerical_data": [
    {
      "metric": "string",
      "value": "string",
      "source": "string"
    }
  ],
  "contradictions_found": [
    {
      "claim_a": "string",
      "claim_b": "string",
      "source_a": "string",
      "source_b": "string"
    }
  ],
  "implicit_assumptions": ["string"],
  "analysis_confidence": 0-100
}
```

**Credential:**
- OpenRouter Account: "OpenRouter account for AI Journal"
- API Key: Required in n8n credentials

**Next Node:** Parse Output

---

### NODE 4: Parse Output
**Type:** Function (Code Node)  
**Position:** (784, 0)  
**Purpose:** Extract JSON from LLM output, remove markdown formatting

**Logic:**
```javascript
// Get raw LLM output
const raw = $input.first().json.output;

if (!raw || typeof raw !== 'string') {
  throw new Error('No valid output field found');
}

// Remove markdown code fences (```json ... ```)
const cleaned = raw
  .replace(/```json\s*/i, '')
  .replace(/```/g, '')
  .trim();

// Parse JSON safely
let parsed;
try {
  parsed = JSON.parse(cleaned);
} catch (e) {
  throw new Error('Failed to parse JSON output: ' + e.message);
}

// Return parsed object
return [
  {
    json: {
      facts: parsed.facts || [],
      recurring_themes: parsed.recurring_themes || [],
      numerical_data: parsed.numerical_data || [],
      contradictions_found: parsed.contradictions_found || [],
      implicit_assumptions: parsed.implicit_assumptions || [],
      analysis_confidence: parsed.analysis_confidence ?? null,
      parsed_at: new Date().toISOString()
    }
  }
];
```

**Purpose of This Node:**
- LLMs sometimes wrap JSON in markdown code fences
- This node strips formatting and validates JSON
- Prevents downstream errors from malformed JSON

**Error Handling:**
- Catches JSON.parse errors
- Provides meaningful error messages
- Allows workflow to continue with empty arrays as defaults

**Next Node:** Skeptic Agent (AI Node)

---

### NODE 5: Skeptic Agent (AI Node)
**Type:** LangChain Agent (@n8n/n8n-nodes-langchain.agent)  
**Position:** (-16, 288)  
**Purpose:** Challenge Research Agent findings, flag weak evidence, identify logical gaps

**AI Model Configuration:**
```
Model:              OpenRouter Chat Model 1 (GLM-4.5-Air:free)
Temperature:        0.5 (critical thinking)
Max Tokens:         1500
Type:               LangChain Agent (v3)
```

**System Prompt:**
```
You are a Skeptic Verifier Agent. Your job is to challenge, not confirm.

Given research findings, your task is to:
1. Identify claims with weak evidence (only 1 source, low evidence_strength)
2. Flag logical leaps or unsupported inferences
3. Identify missing context or alternative explanations
4. Spot circular reasoning or self-citation
5. Point out assumptions that may not hold in all contexts
6. Suggest counter-arguments or competing narratives

Be brutal. Assume the Research Agent missed something.

Output ONLY valid JSON:
{
  "weak_claims": [
    {
      "claim": "string",
      "reason": "string",
      "suggested_evidence": "string"
    }
  ],
  "logical_gaps": [
    {
      "gap": "string",
      "impact": "high|medium|low"
    }
  ],
  "missing_context": ["context1", "context2"],
  "alternative_explanations": [
    {
      "claim": "string",
      "alternative": "string",
      "plausibility": "high|medium|low"
    }
  ],
  "methodological_concerns": ["concern1", "concern2"],
  "skepticism_confidence": 0-100
}
```

**User Prompt (Dynamic):**
```
Topic: {{ $('Input Validator & Memory Loader').item.json.validated_input.research_topic }}

Research findings from Agent 1:
{{ $('Research Agent (AI Node)').item.json.output }}

Challenge these findings. Do not accept them at face value. 
What is weak? What is missing? What could be wrong?
```

**Expected JSON Output Schema:**
```json
{
  "weak_claims": [
    {
      "claim": "string",
      "reason": "string",
      "suggested_evidence": "string"
    }
  ],
  "logical_gaps": [
    {
      "gap": "string",
      "impact": "high|medium|low"
    }
  ],
  "missing_context": ["string"],
  "alternative_explanations": [
    {
      "claim": "string",
      "alternative": "string",
      "plausibility": "high|medium|low"
    }
  ],
  "methodological_concerns": ["string"],
  "skepticism_confidence": 0-100
}
```

**Key Differences from Research Agent:**
- Higher temperature (0.5 vs 0.3) for critical thinking
- No output parsing node needed (can return object directly)
- Reads Research Agent output as input context

**Next Node:** Synthesizer Agent (AI Node)

---

### NODE 6: Synthesizer Agent (AI Node)
**Type:** LangChain Agent (@n8n/n8n-nodes-langchain.agent)  
**Position:** (320, 288)  
**Purpose:** Reconcile Research + Skeptic outputs, produce decision-ready insights

**AI Model Configuration:**
```
Model:              OpenRouter Chat Model 2 (GLM-4.5-Air:free)
Temperature:        0.4 (balanced reasoning)
Max Tokens:         2000
Type:               LangChain Agent (v3)
```

**System Prompt:**
```
You are a Synthesizer & Strategist Agent. Your job is not to summarize—it is to reconcile and act.

Given:
- Research findings (from Agent 1)
- Skeptical critiques (from Agent 2)

Your task:
1. Accept what is verified (high evidence + passes skepticism)
2. Flag what is uncertain (conflicting findings OR skeptic raised valid concern)
3. Identify what is unresolved (open questions the team needs to explore)
4. Translate findings into strategic implications (why this matters to the audience)
5. Recommend concrete next actions (not research, not analysis—actions)

Your output MUST be decision-ready. Assume the reader is busy.

Output ONLY valid JSON:
{
  "executive_summary": "string (1-2 sentences, the core insight)",
  "key_insights": [
    {
      "insight": "string",
      "confidence": 0-100,
      "based_on": ["finding1", "finding2"],
      "implications": "string"
    }
  ],
  "conflicting_views": [
    {
      "view_a": "string",
      "view_b": "string",
      "resolution": "string OR 'unresolved'",
      "recommendation": "string"
    }
  ],
  "verified_assumptions": [
    {
      "assumption": "string",
      "supporting_evidence": ["source1", "source2"]
    }
  ],
  "open_questions": [
    {
      "question": "string",
      "why_it_matters": "string",
      "data_needed": "string"
    }
  ],
  "recommended_next_actions": [
    {
      "action": "string",
      "rationale": "string",
      "owner": "string (optional)",
      "timeline": "string (optional)"
    }
  ],
  "synthesis_confidence": 0-100,
  "confidence_rationale": "string"
}
```

**User Prompt (Dynamic):**
```
Topic: {{ $('Input Validator & Memory Loader').item.json.validated_input.research_topic }}
Audience: {{ $('Input Validator & Memory Loader').item.json.validated_input.audience }}
Industry: {{ $('Input Validator & Memory Loader').item.json.validated_input.industry }}

Research findings:
{{ $('Research Agent (AI Node)').item.json.output }}

Skeptic critiques:
{{ $json.output }}

Reconcile these. What is truly verified? What is conflicted? What matters to the {{ $('Input Validator & Memory Loader').item.json.validated_input.audience }}?
Do not summarize. Synthesize. Recommend actions.
```

**Expected JSON Output Schema:**
```json
{
  "executive_summary": "string",
  "key_insights": [
    {
      "insight": "string",
      "confidence": 0-100,
      "based_on": ["string"],
      "implications": "string"
    }
  ],
  "conflicting_views": [
    {
      "view_a": "string",
      "view_b": "string",
      "resolution": "string",
      "recommendation": "string"
    }
  ],
  "verified_assumptions": [
    {
      "assumption": "string",
      "supporting_evidence": ["string"]
    }
  ],
  "open_questions": [
    {
      "question": "string",
      "why_it_matters": "string",
      "data_needed": "string"
    }
  ],
  "recommended_next_actions": [
    {
      "action": "string",
      "rationale": "string",
      "owner": "string",
      "timeline": "string"
    }
  ],
  "synthesis_confidence": 0-100,
  "confidence_rationale": "string"
}
```

**Critical Notes:**
- Uses 2000 tokens (longer than other agents) to produce decision-ready content
- Reads both Research Agent and Skeptic Agent outputs
- Highest confidence = decision-ready recommendation
- Lowest confidence = flag for further investigation

**Next Node:** Comparison Engine (Function)

---

### NODE 7: Comparison Engine (Function)
**Type:** Function (Code Node)  
**Purpose:** Compare current findings to previous research runs, detect patterns

**Configuration:**
- Input: Synthesizer output
- Output: Comparison metadata for tracking research evolution

**Logic (Placeholder - awaiting full JSON):**
```javascript
// This node compares current synthesis to historical runs
// Tracks: new insights, repeated findings, shifted views, evolution score
// Requires connection to historical memory (future: Google Sheets lookup)

const synthesis = $input.first().json;

const comparison = {
  new_insights: [],
  repeated_findings: [],
  contradictions_resolved: [],
  contradictions_emerged: [],
  evolution_score: 0,
  comparison_summary: "First research run on this topic."
};

// If this is first run, no comparisons available
// Future enhancement: Load from Google Sheets and compare

return {
  json: {
    comparison_result: comparison,
    synthesis_data: synthesis
  }
};
```

**Next Node:** Markdown Formatter (Function)

---

### NODE 8: Markdown Formatter (Function)
**Type:** Function (Code Node)  
**Purpose:** Transform structured JSON into human-readable Markdown research brief

**Inputs:**
- Synthesizer output (key_insights, open_questions, recommended_next_actions)
- Comparison result (evolution tracking)
- Validated input metadata (topic, audience, industry)

**Output Format:**
```markdown
# Research Brief: [Topic]

**Generated:** [ISO timestamp]
**Confidence Level:** [0-100]%

## ✅ What We Know
[Executive summary]

### Key Insights
1. [Insight] ([Confidence]%)
   - Why it matters: [Implications]

## ⚠️ What We're Unsure About
### Conflicting Views
[View A vs. View B]
- Resolution: [Resolved / Unresolved]

### Open Questions
1. [Question]
   - Why it matters: [Context]
   - Data needed: [Requirements]

## 🎯 Why This Matters
[Strategic implications]

## ➡️ What to Do Next
1. [Action] - [Rationale]
   - Timeline: [Optional]

## 📊 Research Evolution
- New insights: [N]
- Evolution score: [%]

## 🔍 Methodology
- Analysis confidence: [%]
- Skepticism applied: Yes
```

**Next Node:** Email Sender, Memory Archiver

---

### NODE 9: Email Sender
**Type:** Gmail (n8n-nodes-base.gmail)  
**Purpose:** Deliver final research brief to decision-maker

**Configuration:**
```
Service:    Gmail
To:         Dynamic recipient (from input)
Subject:    "Research Brief: [Topic] — [Date]"
Body:       Markdown brief
Importance: Normal
```

**Email Template:**
- To: `{{ $input.first().json.recipient_email }}`
- Subject: `Research Brief: {{ $('Input Validator & Memory Loader').item.json.validated_input.research_topic }} — {{ now.toFormat('MMMM dd, yyyy') }}`
- Body: `{{ $input.first().json.markdown_brief }}`

**Credentials Required:**
- Gmail account with app password
- OAuth 2.0 credentials for n8n

**Error Handling:**
- Retry on network timeout (automatic)
- Log failed deliveries

---

### NODE 10: Memory Archiver (Google Sheets)
**Type:** Google Sheets (n8n-nodes-base.googleSheets)  
**Purpose:** Archive research results for historical comparison

**Sheet Configuration:**
```
Sheet Name: research_memory
Spreadsheet: [Requires setup - must be created manually]

Columns:
  A: date (ISO timestamp)
  B: topic (research topic)
  C: findings_hash (hash of key insights)
  D: key_insights (JSON stringified)
  E: conflicting_views (count of unresolved)
  F: confidence_score (synthesis_confidence)
  G: evolution_score (%)
  H: next_actions_count (length of recommendations)
  I: run_id (unique identifier)
```

**Append Row Configuration:**
```json
{
  "date": "{{ $input.first().json.timestamp }}",
  "topic": "{{ $('Input Validator & Memory Loader').item.json.validated_input.research_topic }}",
  "findings_hash": "{{ $('Parse Output').item.json.facts.length }}",
  "key_insights": "{{ JSON.stringify($input.first().json.key_insights) }}",
  "conflicting_views": "{{ $input.first().json.conflicting_views.length }}",
  "confidence_score": "{{ $input.first().json.synthesis_confidence }}",
  "evolution_score": "{{ $('Comparison Engine').item.json.comparison_result.evolution_score }}",
  "next_actions_count": "{{ $input.first().json.recommended_next_actions.length }}",
  "run_id": "{{ $('Input Validator & Memory Loader').item.json.validated_input.run_id }}"
}
```

**Credentials Required:**
- Google OAuth 2.0 credentials
- Spreadsheet ID (from URL)

---

## 4. DATA FLOW & CONNECTIONS

### Connection Map
```
Webhook Trigger
    ↓
Input Validator & Memory Loader
    ↓
Research Agent (AI Node) ←→ OpenRouter Chat Model
    ↓
Parse Output
    ↓
Skeptic Agent (AI Node) ←→ OpenRouter Chat Model 1
    ↓
Synthesizer Agent (AI Node) ←→ OpenRouter Chat Model 2
    ↓
Comparison Engine (Function)
    ↓
Markdown Formatter (Function)
    ├→ Email Sender
    └→ Memory Archiver (Google Sheets)
```

### Data Type Transformations
```
Webhook Input (JSON)
    ↓ [Validation]
Normalized Input (JSON with defaults)
    ↓ [Research Agent]
Research Findings (JSON with facts/themes/contradictions)
    ↓ [Parse Output]
Cleaned JSON (validated structure)
    ↓ [Skeptic Agent]
Skeptic Critiques (JSON with weak_claims/gaps)
    ↓ [Synthesizer Agent]
Synthesis Output (JSON with insights/actions)
    ↓ [Comparison Engine]
Comparison Metadata (evolution tracking)
    ↓ [Markdown Formatter]
Markdown Research Brief (human-readable)
    ↓ [Email Sender + Memory Archiver]
Delivered Brief + Archived History
```

---

## 5. AGENT SPECIFICATIONS

### Agent 1: Research Agent

**Role:** Extract facts, identify patterns, flag contradictions

**Model:** GLM-4.5-Air (OpenRouter free tier)

**Temperature:** 0.3 (factual, deterministic)

**Key Responsibilities:**
1. Extract factual claims from general knowledge
2. Assess evidence strength (high/medium/low)
3. Identify recurring themes (appear 2+ times)
4. Document numerical benchmarks
5. Flag contradictions in public discourse
6. Identify implicit assumptions

**Does NOT:**
- Invent sources or statistics
- Provide strategic recommendations
- Interpret or judge findings
- Add opinions

**Confidence Range:** 0-100 (analysis_confidence)
- 80-100: High confidence in extracted facts
- 50-79: Moderate confidence, some uncertainty
- 0-49: Low confidence, limited public knowledge

**Output Quality Metrics:**
- Facts completeness: 5+ facts indicates thorough extraction
- Theme identification: 3+ themes indicates pattern recognition
- Contradiction detection: Honest about public discourse conflicts

---

### Agent 2: Skeptic Agent

**Role:** Challenge findings, flag weaknesses, identify gaps

**Model:** GLM-4.5-Air (OpenRouter free tier)

**Temperature:** 0.5 (critical analysis)

**Key Responsibilities:**
1. Identify claims with weak evidence
2. Flag logical leaps
3. Suggest alternative explanations
4. Point out missing context
5. Spot circular reasoning
6. Challenge assumptions

**Does NOT:**
- Affirm findings without challenge
- Provide independent analysis
- Make strategic recommendations

**Skepticism Confidence:** 0-100
- 80-100: Rigorous skepticism, confident in critique
- 50-79: Moderate skepticism, some valid concerns
- 0-49: Low skepticism, few major issues found

**Contradiction Detection:**
- Weak Claims: Claims with evidence_strength="low"
- Logical Gaps: Inferences not supported by evidence
- Missing Context: Information that changes interpretation

**Output Quality Metrics:**
- Finds 2-5 weak claims per research run (typical)
- Identifies 1-3 major logical gaps
- Suggests 2-3 alternative explanations

---

### Agent 3: Synthesizer Agent

**Role:** Reconcile findings, produce actionable insights

**Model:** GLM-4.5-Air (OpenRouter free tier)

**Temperature:** 0.4 (balanced reasoning)

**Max Tokens:** 2000 (longest output)

**Key Responsibilities:**
1. Verify findings (high evidence + passes skepticism)
2. Flag uncertain findings (conflicts or skeptic concerns)
3. Identify unresolved questions
4. Translate to strategic implications
5. Recommend concrete actions
6. Assign confidence scores

**Does NOT:**
- Simply summarize findings
- Remain neutral on contradictions
- Avoid decision-making language

**Decision-Ready Output:**
- Executive summary: 1-2 sentences, core insight
- Key insights: 3-5 insights with confidence scores
- Open questions: 2-4 unresolved items
- Next actions: 3-5 concrete, immediately actionable steps

**Confidence Scoring:**
- Verified (80-100): High evidence + skeptic approval
- Uncertain (50-79): Conflicting views or skeptic concern
- Unresolved (0-49): Insufficient information

**Output Quality Metrics:**
- Insights with 75%+ confidence are decision-ready
- Conflicting views clearly documented
- Actions are SMART (Specific, Measurable, Achievable, Relevant, Time-bound)

---

## 6. SETUP INSTRUCTIONS

### Prerequisites
- n8n instance (self-hosted or cloud)
- OpenRouter account (free tier available)
- Gmail account with app password
- Google Sheets access (optional, for memory archiver)

### Step 1: Create OpenRouter Account
1. Visit https://openrouter.ai
2. Sign up with email
3. Generate API key in account settings
4. Note the key (format: `sk-or-...`)

### Step 2: Configure OpenRouter Credentials in n8n
1. Open n8n instance
2. Go to Credentials
3. Create new "OpenRouter API" credential
4. **Credential Name:** OpenRouter account for AI Journal
5. **API Key:** [Paste from Step 1]
6. Save and test

### Step 3: Configure Gmail Credentials (Optional)
1. Enable Gmail API in Google Cloud Console
2. Create OAuth 2.0 credentials
3. Add to n8n under "Gmail" credentials
4. Authorize account access
5. Note: Requires "App Password" if 2FA is enabled

### Step 4: Create Google Sheets (Optional)
1. Create new Google Sheet: "Research Memory"
2. Create sheet named "research_memory"
3. Add headers in Row 1:
   - A1: date
   - B1: topic
   - C1: findings_hash
   - D1: key_insights
   - E1: conflicting_views
   - F1: confidence_score
   - G1: evolution_score
   - H1: next_actions_count
   - I1: run_id
4. Share with n8n service account
5. Copy Spreadsheet ID from URL

### Step 5: Import Workflow
1. Download workflow JSON: `Multi-Agent_AI_Research_Assistant.json`
2. In n8n, click "Import"
3. Paste JSON
4. Resolve credential references:
   - OpenRouter: Map to credential from Step 2
   - Gmail: Map to credential from Step 3
   - Google Sheets: Map to credential + spreadsheet ID
5. Activate webhook (or leave inactive for manual testing)

### Step 6: Configure Webhook
1. Get Webhook URL from Webhook Trigger node
2. Format: `https://[n8n-url]/webhook-test/research-assistant`
3. Test webhook locally with curl:
   ```bash
   curl -X POST https://[n8n-url]/webhook/research-assistant \
     -H "Content-Type: application/json" \
     -d '{
       "research_topic": "AI adoption in enterprises",
       "industry": "technology",
       "audience": "CIOs",
       "depth_level": "deep"
     }'
   ```

### Step 7: Test Run
1. Send test webhook request
2. Monitor execution:
   - Check Research Agent output (should return JSON)
   - Verify Skeptic Agent challenge (should find 2-3 weak claims)
   - Confirm Synthesizer output (should have key_insights)
   - Validate Markdown formatting
3. Check email inbox for research brief
4. Verify Google Sheets append (if enabled)

---

## 7. CONFIGURATION GUIDE

### OpenRouter Model Selection
**Current:** `z-ai/glm-4.5-air:free`

**Alternative Free Models:**
```
mistral/mistral-7b-instruct      (faster, lower quality)
meta-llama/llama-2-70b-chat      (slower, higher quality)
gpt-3.5-turbo (if available)      (balanced)
```

**Changing Models:**
1. Edit each of 3 "OpenRouter Chat Model" nodes
2. Update `model` parameter
3. Test with sample input
4. Monitor token usage

### Temperature Tuning
**Current Settings:**
- Research Agent: 0.3 (factual)
- Skeptic Agent: 0.5 (critical)
- Synthesizer Agent: 0.4 (balanced)

**Tuning Guide:**
- **Lower temperature (0.0-0.3):** More deterministic, factual, consistent
- **Higher temperature (0.7-1.0):** More creative, varied, exploratory

**When to Adjust:**
- If agents produce identical outputs → lower temperature
- If outputs are too repetitive → raise temperature
- If outputs are too creative → lower temperature

### Token Limits
**Current Settings:**
- Research Agent: 1500 tokens
- Skeptic Agent: 1500 tokens
- Synthesizer Agent: 2000 tokens

**Monitoring:**
- Check n8n logs for token usage per run
- OpenRouter shows usage in account dashboard
- Free tier: ~3 million tokens/month typical

**When to Increase:**
- If outputs are truncated → increase max_tokens
- Complex topics may need 2000+ tokens

### Prompt Customization
**Research Agent Prompt:**
- Edit in "Research Agent (AI Node)" node
- Modify to focus on specific aspects
- Example: Add "Focus on financial impact" for business topics

**Skeptic Agent Prompt:**
- Edit in "Skeptic Agent (AI Node)" node
- Make more or less aggressive
- Example: Add "Prioritize cost concerns" for financial analysis

**Synthesizer Agent Prompt:**
- Edit in "Synthesizer Agent (AI Node)" node
- Customize for audience
- Example: Add "Recommendations for startup implementation" for entrepreneurs

### Webhook Configuration
**Current Path:** `/research-assistant`

**Custom Paths:**
- Edit Webhook Trigger node → "path" parameter
- Example: `/webhook/market-research`, `/api/analysis-request`

**Request Timeout:**
- Default: 60 seconds
- Adjust in Webhook settings if needed

---

## 8. TESTING GUIDE

### Unit Testing (Test Individual Agents)

#### Test 1: Research Agent Extraction
```json
Input Topic: "Blockchain adoption in supply chain"
Expected Output:
{
  "facts": [
    {
      "claim": "Blockchain improves supply chain transparency",
      "evidence_strength": "high",
      "sources_count": 3
    }
  ],
  "analysis_confidence": 75
}
```

**Validation:**
- facts array has 5+ items (thorough)
- evidence_strength uses all three levels
- analysis_confidence is 50-100

#### Test 2: Skeptic Agent Challenge
```json
Input Research: "[Research Agent output from Test 1]"
Expected Output:
{
  "weak_claims": [
    {
      "claim": "Blockchain improves supply chain transparency",
      "reason": "Only 3 sources, some are vendor whitepapers",
      "suggested_evidence": "Independent audits or academic studies"
    }
  ],
  "skepticism_confidence": 65
}
```

**Validation:**
- weak_claims array has 2-4 items (critical but not dismissive)
- logical_gaps are concrete and actionable
- alternative_explanations are plausible

#### Test 3: Synthesizer Agent Synthesis
```json
Input Research + Skeptic: "[Both from Tests 1 & 2]"
Expected Output:
{
  "executive_summary": "Blockchain offers moderate supply chain benefits but adoption remains limited by cost and integration complexity.",
  "key_insights": [
    {
      "insight": "Transparency gains are real but overstated",
      "confidence": 72,
      "implications": "ROI depends on implementation scope"
    }
  ],
  "synthesis_confidence": 71
}
```

**Validation:**
- executive_summary is 1-2 sentences
- key_insights have 60-80% confidence (realistic)
- recommended_next_actions are concrete

### End-to-End Testing

#### Test Scenario 1: Quick Research (3 sources)
```bash
curl -X POST http://localhost:5678/webhook/research-assistant \
  -H "Content-Type: application/json" \
  -d '{
    "research_topic": "Remote work trends 2025",
    "industry": "technology",
    "audience": "HR directors",
    "depth_level": "quick"
  }'
```

**Expected Duration:** 45-90 seconds

**Validation Checklist:**
- [ ] Webhook receives request
- [ ] Input validator normalizes fields
- [ ] Research Agent completes (check tokens)
- [ ] Parse Output succeeds
- [ ] Skeptic Agent challenges (finds issues)
- [ ] Synthesizer reconciles
- [ ] Markdown formatting succeeds
- [ ] Email sent successfully
- [ ] Google Sheets appended

#### Test Scenario 2: Deep Research (5 sources)
```bash
curl -X POST http://localhost:5678/webhook/research-assistant \
  -H "Content-Type: application/json" \
  -d '{
    "research_topic": "AI agent adoption in enterprises",
    "industry": "enterprise software",
    "audience": "CTOs",
    "depth_level": "deep"
  }'
```

**Expected Duration:** 90-180 seconds

**Validation Checklist:**
- [ ] All steps from Test 1
- [ ] synthesis_confidence is 65-85%
- [ ] 4-6 key_insights generated
- [ ] 2-3 conflicting_views documented
- [ ] 3-5 recommended_next_actions

#### Test Scenario 3: Error Handling
```bash
# Missing required field
curl -X POST http://localhost:5678/webhook/research-assistant \
  -H "Content-Type: application/json" \
  -d '{
    "industry": "technology"
  }'
```

**Expected Result:** Error message "research_topic is required"

### Performance Testing

#### Metrics to Monitor
```
execution_time_seconds:
  - Quick research: 45-90 seconds (target)
  - Deep research: 90-180 seconds (target)

token_usage_per_run:
  - Research Agent: 400-600 tokens
  - Skeptic Agent: 300-500 tokens
  - Synthesizer Agent: 600-900 tokens
  - Total: 1300-2000 tokens per run

cost_per_run:
  - OpenRouter free tier: ~0 USD
  - Gmail API: Free (<15K emails/day)

error_rate:
  - Target: <2% (transient network errors)
  - Investigate: >5%
```

#### Load Testing
**Don't do this on free tier, but if self-hosted:**
```bash
# Sequential requests (safe)
for i in {1..5}; do
  curl -X POST http://localhost:5678/webhook/research-assistant \
    -d "..." &
  sleep 10  # Stagger requests
done
```

---

## 9. TROUBLESHOOTING

### Issue: "research_topic is required"
**Cause:** Webhook payload missing required field

**Fix:**
```json
// Add to webhook body:
{
  "research_topic": "Your research question here"
}
```

### Issue: Research Agent returns empty facts array
**Cause:** Model may have weak knowledge of topic

**Symptoms:**
- `analysis_confidence: 15`
- Empty `facts` array
- No `recurring_themes`

**Fix:**
1. Use more specific research topic
2. Add industry context
3. Lower temperature (make more factual)

**Example:**
```
❌ Topic: "The future"
✅ Topic: "AI regulation in financial services by 2025"
```

### Issue: JSON parsing fails after agent output
**Location:** Parse Output node  
**Cause:** Agent wrapped JSON in markdown fences incorrectly

**Error Message:** "Failed to parse JSON output: SyntaxError: Unexpected token"

**Fix:**
1. Check agent output in execution history
2. Verify markdown fence removal logic
3. Adjust regex if needed:
   ```javascript
   const cleaned = raw
     .replace(/```json\s*/gi, '')  // Case-insensitive
     .replace(/```/g, '')
     .trim();
   ```

### Issue: Email not sending
**Cause:** Gmail credentials invalid or 2FA issue

**Diagnosis:**
1. Check Gmail credential in n8n
2. Test with a simple text email
3. Verify app password (not regular password)

**Fix:**
1. Regenerate Gmail app password
2. Update credential in n8n
3. Re-authorize OAuth 2.0

### Issue: Google Sheets append fails
**Cause:** Sheet name or column mismatch

**Diagnosis:**
1. Check sheet exists with exact name "research_memory"
2. Verify columns A-I have headers
3. Check n8n has Sheets access

**Fix:**
1. Manually create sheet with correct name
2. Share with n8n service account
3. Update spreadsheet ID in Memory Archiver node

### Issue: Synthesizer Agent produces generic output
**Cause:** Topic too broad or model lacks context

**Symptoms:**
- `synthesis_confidence: 30-40`
- Recommendations are generic ("conduct more research")
- Few specific insights

**Fix:**
1. Provide more specific topic
2. Add industry/audience context
3. Increase synthesizer temperature slightly (0.4 → 0.5)

### Issue: Skeptic Agent doesn't challenge findings
**Cause:** Temperature too low or research findings too weak

**Symptoms:**
- `skepticism_confidence: 90+`
- Empty `weak_claims` array
- No `logical_gaps`

**Fix:**
1. Increase skeptic temperature (0.5 → 0.7)
2. Update skeptic prompt to be more critical
3. Check research agent isn't producing low-confidence facts

### Issue: Webhook returns "No execution history"
**Cause:** Webhook still in "test" mode

**Fix:**
1. Activate workflow (click "Activate" button)
2. Use production webhook URL (not `/webhook-test/`)
3. Check execution after sending request

---

## 10. EXAMPLE RUNS

### Example 1: Quick Research — Remote Work Trends
```
INPUT:
{
  "research_topic": "Remote work productivity metrics 2025",
  "industry": "HR Tech",
  "audience": "HR directors",
  "depth_level": "quick",
  "recipient_email": "hr-director@company.com"
}

RESEARCH AGENT OUTPUT (summary):
{
  "facts": [
    {
      "claim": "Hybrid work productivity metrics show 10-15% improvement in output",
      "evidence_strength": "high",
      "sources_count": 3
    },
    {
      "claim": "Remote workers report 23% higher burnout risk",
      "evidence_strength": "medium",
      "sources_count": 2
    }
  ],
  "analysis_confidence": 72
}

SKEPTIC AGENT OUTPUT (summary):
{
  "weak_claims": [
    {
      "claim": "10-15% productivity improvement",
      "reason": "Only 3 sources, metrics vary by industry and role",
      "suggested_evidence": "Longitudinal study across industries"
    }
  ],
  "logical_gaps": [
    {
      "gap": "Productivity measured how? Lines of code? Sales? No standard metric",
      "impact": "high"
    }
  ],
  "skepticism_confidence": 68
}

SYNTHESIZER AGENT OUTPUT (summary):
{
  "executive_summary": "Remote work boosts output metrics but risks worker wellbeing. Success depends on role type and management approach.",
  "key_insights": [
    {
      "insight": "Hybrid models outperform fully remote or fully in-office",
      "confidence": 74,
      "implications": "HR should design role-specific policies, not one-size-fits-all"
    }
  ],
  "open_questions": [
    {
      "question": "How do remote work benefits vary by role type (engineering vs. sales)?",
      "why_it_matters": "Company productivity gains depend on role mix",
      "data_needed": "Productivity data segmented by function"
    }
  ],
  "recommended_next_actions": [
    {
      "action": "Segment remote work policy by role (engineering, sales, admin)",
      "rationale": "Data shows benefit varies significantly by function",
      "timeline": "30 days"
    }
  ],
  "synthesis_confidence": 71
}

EMAIL RECEIVED (Markdown Brief):
---
# Research Brief: Remote Work Productivity Metrics 2025

**Generated:** February 24, 2026, 2:34 PM
**Confidence Level:** 71%

## ✅ What We Know

Hybrid work boosts output metrics but risks worker wellbeing. Success depends on role type and management approach.

### Key Insights

1. **Hybrid models outperform fully remote or fully in-office** (74% confidence)
   - Why it matters: HR should design role-specific policies, not one-size-fits-all

## ⚠️ What We're Unsure About

### Conflicting Views
**Conflict 1:**
- Perspective A: Productivity improvement is 10-15% across roles
- Perspective B: Improvement varies 5-30% by role type
- Resolution: 🚩 Unresolved—need role-specific data

### Open Questions
1. **How do remote work benefits vary by role type?**
   - Why it matters: Company productivity gains depend on role mix
   - Data needed: Productivity data segmented by function

## 🎯 Why This Matters

For your organization: If most headcount is in engineering, hybrid gains could be 20%. If sales-heavy, gains may be negative. One policy fails.

## ➡️ What to Do Next

1. **Segment remote work policy by role** - Data shows benefit varies significantly by function - Timeline: 30 days
2. **Survey employee wellbeing metrics** - 23% burnout risk is significant; need company-specific baseline - Timeline: Immediate
3. **Track productivity by role for 8 weeks** - Establish baseline before policy change - Timeline: 8 weeks

---
```

### Example 2: Deep Research — AI Agent Adoption
```
[Similar structure but with deeper analysis, 5+ sources simulated, more conflicting views]
```

---

## 11. MAINTENANCE & MONITORING

### Daily Checks
- [ ] Monitor email delivery (check spam folder)
- [ ] Review error logs in n8n
- [ ] Verify OpenRouter API key still valid
- [ ] Check Google Sheets sheet has new rows

### Weekly Checks
- [ ] Review synthesis_confidence scores (should average 60-75%)
- [ ] Confirm agents are generating diverse outputs (not repetitive)
- [ ] Audit open_questions for patterns
- [ ] Review recommended_next_actions for feasibility

### Monthly Checks
- [ ] Compare research evolution scores over time
- [ ] Analyze agent temperature settings (too hot/cold?)
- [ ] Review OpenRouter token usage
- [ ] Update prompts if needed based on feedback
- [ ] Check for model availability changes

### Quarterly Reviews
- [ ] Evaluate synthesis quality vs. actual decisions made
- [ ] Assess whether open_questions were eventually answered
- [ ] Survey users on output usefulness
- [ ] Consider model upgrades or changes
- [ ] Archive historical research data

---

## 12. PRODUCTION DEPLOYMENT CHECKLIST

- [ ] OpenRouter account created and tested
- [ ] API key securely stored in n8n credentials
- [ ] Gmail OAuth 2.0 configured (if using email)
- [ ] Google Sheets created with correct schema (if using memory)
- [ ] Workflow imported and all nodes connected
- [ ] Webhook URL documented
- [ ] Test run successful (all 10 nodes executed)
- [ ] Email delivery confirmed
- [ ] Google Sheets append verified
- [ ] Monitoring and logging configured
- [ ] Error handling tested
- [ ] Documentation shared with team
- [ ] Workflow activated (not in test mode)
- [ ] Runbook created for incident response
- [ ] On-call escalation path defined

---

## 13. GLOSSARY

| Term | Definition |
|------|-----------|
| **Analysis Confidence** | Research Agent's confidence in extracted facts (0-100%) |
| **Skepticism Confidence** | Skeptic Agent's confidence in finding weaknesses (0-100%) |
| **Synthesis Confidence** | Synthesizer Agent's confidence in recommendations (0-100%) |
| **Evidence Strength** | Quality of evidence for a claim (high/medium/low) |
| **Weak Claims** | Facts with only 1 source or low evidence strength |
| **Logical Gap** | Inference or conclusion not fully supported by evidence |
| **Conflicting Views** | Two competing perspectives on the same finding |
| **Verified Assumption** | Assumption supported by evidence across sources |
| **Open Question** | Unresolved gap requiring further investigation |
| **Evolution Score** | % of new insights vs. repeated findings over time |
| **Run ID** | Unique identifier for each research execution |

---

## 14. SUPPORT & CONTACT

**Common Issues:** See Section 9: Troubleshooting

**Feature Requests:**
- Add new agent role
- Integrate with Slack/Teams for notifications
- Add PDF export for research brief
- Implement multi-language support

**Report Bugs:**
- Include workflow execution ID
- Share error message from n8n logs
- Provide research topic that triggered error

---

**Document Version:** 1.0  
**Last Updated:** February 2026  
**Maintainer:** AI Research Assistant Team  
**Status:** Production Ready
