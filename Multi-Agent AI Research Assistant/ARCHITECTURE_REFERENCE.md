# Multi-Agent AI Research Assistant
## Architecture Reference & Data Schemas

---

## SYSTEM ARCHITECTURE

### High-Level Data Flow
```
User Request
    ↓
Webhook Trigger
    ↓
Input Validator
    ├─ Normalize fields
    ├─ Generate run_id
    └─ Validate required fields
    ↓
Research Agent (AI)
    ├─ Temperature: 0.3
    ├─ Max Tokens: 1500
    └─ Model: GLM-4.5-Air:free
    ↓
JSON Parser
    ├─ Remove markdown
    └─ Validate structure
    ↓
Skeptic Agent (AI)
    ├─ Temperature: 0.5
    ├─ Max Tokens: 1500
    └─ Model: GLM-4.5-Air:free
    ↓
Synthesizer Agent (AI)
    ├─ Temperature: 0.4
    ├─ Max Tokens: 2000
    └─ Model: GLM-4.5-Air:free
    ↓
Comparison Engine
    └─ Track evolution vs. history
    ↓
Markdown Formatter
    ├─ Format for humans
    └─ Add metadata
    ↓
┌─────────────────────┐
├─ Email Sender       │
├─ Memory Archiver    │
└─ Webhook Response   │
```

### Execution Order
- **Sequential** (v1 execution mode)
- Agents run one-after-another (not parallel)
- Each agent waits for previous completion
- Reduces token waste, ensures context preservation

### Processing Timeline
```
T+0s    → Webhook received
T+2s    → Input validation complete
T+3s    → Research Agent starts
T+15s   → Research Agent completes (~1500 tokens used)
T+16s   → JSON parsing
T+17s   → Skeptic Agent starts
T+25s   → Skeptic Agent completes (~1500 tokens used)
T+26s   → Synthesizer Agent starts
T+40s   → Synthesizer Agent completes (~2000 tokens used)
T+41s   → Comparison Engine
T+42s   → Markdown Formatter
T+45s   → Email + Sheets write
T+60s   → Webhook response sent to user
Total: 60-90 seconds (quick) or 90-120 seconds (deep)
```

---

## DATA SCHEMAS

### 1. WEBHOOK INPUT SCHEMA

**Endpoint:** `POST /webhook/research-assistant`

**Request Body:**
```json
{
  "type": "object",
  "required": ["research_topic"],
  "additionalProperties": false,
  "properties": {
    "research_topic": {
      "type": "string",
      "description": "The research question or topic to analyze",
      "minLength": 5,
      "maxLength": 500,
      "example": "AI adoption barriers in healthcare"
    },
    "industry": {
      "type": "string",
      "description": "Industry context for research",
      "minLength": 2,
      "maxLength": 100,
      "default": "general",
      "example": "Healthcare"
    },
    "audience": {
      "type": "string",
      "description": "Who is this research for?",
      "minLength": 2,
      "maxLength": 200,
      "default": "decision-makers",
      "example": "Hospital CIOs"
    },
    "depth_level": {
      "type": "string",
      "enum": ["quick", "deep"],
      "default": "quick",
      "description": "quick = 3 sources (45-60s), deep = 5 sources (90-120s)"
    },
    "recipient_email": {
      "type": "string",
      "format": "email",
      "description": "Email address to send final research brief",
      "optional": true
    }
  }
}
```

**Example Request:**
```bash
curl -X POST http://localhost:5678/webhook/research-assistant \
  -H "Content-Type: application/json" \
  -d '{
    "research_topic": "How does remote work affect team collaboration?",
    "industry": "Technology",
    "audience": "Engineering directors",
    "depth_level": "deep",
    "recipient_email": "director@company.com"
  }'
```

---

### 2. VALIDATED INPUT SCHEMA

**Output from:** Input Validator & Memory Loader  
**Used by:** All downstream nodes

```json
{
  "type": "object",
  "properties": {
    "validated_input": {
      "type": "object",
      "properties": {
        "research_topic": {
          "type": "string",
          "description": "Original topic (validated)"
        },
        "industry": {
          "type": "string",
          "description": "Normalized industry (defaults to 'general')"
        },
        "audience": {
          "type": "string",
          "description": "Normalized audience (defaults to 'decision-makers')"
        },
        "depth_level": {
          "type": "string",
          "enum": ["quick", "deep"],
          "description": "Research depth"
        },
        "sources_count": {
          "type": "integer",
          "enum": [3, 5],
          "description": "3 for quick, 5 for deep"
        },
        "timestamp": {
          "type": "string",
          "format": "date-time",
          "description": "ISO 8601 timestamp of run"
        },
        "run_id": {
          "type": "string",
          "pattern": "^[a-z0-9]{9}$",
          "description": "Unique identifier for this run"
        }
      },
      "required": [
        "research_topic",
        "industry",
        "audience",
        "depth_level",
        "sources_count",
        "timestamp",
        "run_id"
      ]
    },
    "memory_context": {
      "type": "object",
      "properties": {
        "should_load_history": {
          "type": "boolean",
          "default": true
        },
        "comparison_depth": {
          "type": "string",
          "enum": ["full", "summary"]
        }
      }
    }
  }
}
```

---

### 3. RESEARCH AGENT OUTPUT SCHEMA

**Output from:** Research Agent (AI Node)  
**Used by:** Parse Output, Synthesizer Agent

```json
{
  "type": "object",
  "required": ["facts", "analysis_confidence"],
  "properties": {
    "facts": {
      "type": "array",
      "minItems": 0,
      "items": {
        "type": "object",
        "required": ["claim", "evidence_strength", "sources_count"],
        "properties": {
          "claim": {
            "type": "string",
            "description": "The factual statement",
            "minLength": 10,
            "maxLength": 300,
            "example": "Hybrid work models show 10-15% productivity gains"
          },
          "evidence_strength": {
            "type": "string",
            "enum": ["high", "medium", "low"],
            "description": "high: 3+ sources, medium: 2 sources, low: 1 source"
          },
          "sources_count": {
            "type": "integer",
            "minimum": 1,
            "maximum": 10,
            "description": "Number of sources supporting claim"
          },
          "source_examples": {
            "type": "array",
            "items": {
              "type": "string"
            },
            "description": "Name/domain of supporting sources"
          }
        }
      },
      "description": "Key factual findings"
    },
    "recurring_themes": {
      "type": "array",
      "minItems": 0,
      "items": {
        "type": "string",
        "minLength": 5,
        "maxLength": 100
      },
      "description": "Themes appearing 2+ times across sources"
    },
    "numerical_data": {
      "type": "array",
      "minItems": 0,
      "items": {
        "type": "object",
        "required": ["metric", "value"],
        "properties": {
          "metric": {
            "type": "string",
            "example": "Average productivity gain"
          },
          "value": {
            "type": "string",
            "example": "12%"
          },
          "source": {
            "type": "string",
            "example": "McKinsey Report 2024"
          }
        }
      },
      "description": "Numerical benchmarks and statistics"
    },
    "contradictions_found": {
      "type": "array",
      "minItems": 0,
      "items": {
        "type": "object",
        "required": ["claim_a", "claim_b"],
        "properties": {
          "claim_a": {
            "type": "string"
          },
          "claim_b": {
            "type": "string"
          },
          "source_a": {
            "type": "string"
          },
          "source_b": {
            "type": "string"
          }
        }
      },
      "description": "Explicit contradictions in public discourse"
    },
    "implicit_assumptions": {
      "type": "array",
      "minItems": 0,
      "items": {
        "type": "string",
        "minLength": 5,
        "maxLength": 200
      },
      "description": "Unstated assumptions in sources"
    },
    "analysis_confidence": {
      "type": "integer",
      "minimum": 0,
      "maximum": 100,
      "description": "Confidence in overall analysis (0-100%)"
    }
  }
}
```

---

### 4. SKEPTIC AGENT OUTPUT SCHEMA

**Output from:** Skeptic Agent (AI Node)  
**Used by:** Synthesizer Agent

```json
{
  "type": "object",
  "required": ["skepticism_confidence"],
  "properties": {
    "weak_claims": {
      "type": "array",
      "minItems": 0,
      "items": {
        "type": "object",
        "required": ["claim", "reason"],
        "properties": {
          "claim": {
            "type": "string",
            "description": "The claim being challenged"
          },
          "reason": {
            "type": "string",
            "description": "Why the claim is weak",
            "example": "Only 1 source, no independent verification"
          },
          "suggested_evidence": {
            "type": "string",
            "description": "What would strengthen this claim"
          }
        }
      }
    },
    "logical_gaps": {
      "type": "array",
      "minItems": 0,
      "items": {
        "type": "object",
        "required": ["gap", "impact"],
        "properties": {
          "gap": {
            "type": "string",
            "description": "Unsupported leap in logic"
          },
          "impact": {
            "type": "string",
            "enum": ["high", "medium", "low"],
            "description": "How much this gap matters"
          }
        }
      }
    },
    "missing_context": {
      "type": "array",
      "items": {
        "type": "string"
      },
      "description": "Information that would change interpretation"
    },
    "alternative_explanations": {
      "type": "array",
      "minItems": 0,
      "items": {
        "type": "object",
        "properties": {
          "claim": {
            "type": "string"
          },
          "alternative": {
            "type": "string"
          },
          "plausibility": {
            "type": "string",
            "enum": ["high", "medium", "low"]
          }
        }
      }
    },
    "methodological_concerns": {
      "type": "array",
      "items": {
        "type": "string"
      },
      "description": "Issues with how evidence was gathered"
    },
    "skepticism_confidence": {
      "type": "integer",
      "minimum": 0,
      "maximum": 100,
      "description": "Confidence in skepticism (how serious are concerns?)"
    }
  }
}
```

---

### 5. SYNTHESIZER AGENT OUTPUT SCHEMA

**Output from:** Synthesizer Agent (AI Node)  
**Used by:** Comparison Engine, Markdown Formatter, Memory Archiver

```json
{
  "type": "object",
  "required": [
    "executive_summary",
    "key_insights",
    "open_questions",
    "recommended_next_actions",
    "synthesis_confidence"
  ],
  "properties": {
    "executive_summary": {
      "type": "string",
      "minLength": 20,
      "maxLength": 200,
      "description": "1-2 sentence core insight (for busy people)"
    },
    "key_insights": {
      "type": "array",
      "minItems": 1,
      "maxItems": 10,
      "items": {
        "type": "object",
        "required": ["insight", "confidence", "implications"],
        "properties": {
          "insight": {
            "type": "string",
            "description": "Key finding"
          },
          "confidence": {
            "type": "integer",
            "minimum": 0,
            "maximum": 100,
            "description": "How confident in this insight (0-100%)"
          },
          "based_on": {
            "type": "array",
            "items": {
              "type": "string"
            },
            "description": "Which research findings support this"
          },
          "implications": {
            "type": "string",
            "description": "Why this matters to the audience"
          }
        }
      }
    },
    "conflicting_views": {
      "type": "array",
      "minItems": 0,
      "items": {
        "type": "object",
        "required": ["view_a", "view_b", "resolution", "recommendation"],
        "properties": {
          "view_a": {
            "type": "string"
          },
          "view_b": {
            "type": "string"
          },
          "resolution": {
            "type": "string",
            "enum": ["resolved", "unresolved", "needs_investigation"],
            "description": "Can we reconcile these views?"
          },
          "recommendation": {
            "type": "string",
            "description": "What to do about this conflict"
          }
        }
      }
    },
    "verified_assumptions": {
      "type": "array",
      "minItems": 0,
      "items": {
        "type": "object",
        "required": ["assumption", "supporting_evidence"],
        "properties": {
          "assumption": {
            "type": "string"
          },
          "supporting_evidence": {
            "type": "array",
            "items": {
              "type": "string"
            }
          }
        }
      }
    },
    "open_questions": {
      "type": "array",
      "minItems": 1,
      "maxItems": 5,
      "items": {
        "type": "object",
        "required": ["question", "why_it_matters", "data_needed"],
        "properties": {
          "question": {
            "type": "string",
            "description": "Unresolved research question"
          },
          "why_it_matters": {
            "type": "string",
            "description": "Impact if answer is wrong"
          },
          "data_needed": {
            "type": "string",
            "description": "What information would answer this"
          }
        }
      }
    },
    "recommended_next_actions": {
      "type": "array",
      "minItems": 1,
      "maxItems": 7,
      "items": {
        "type": "object",
        "required": ["action", "rationale"],
        "properties": {
          "action": {
            "type": "string",
            "description": "Specific action to take"
          },
          "rationale": {
            "type": "string",
            "description": "Why take this action"
          },
          "owner": {
            "type": "string",
            "description": "Who should own this (optional)"
          },
          "timeline": {
            "type": "string",
            "enum": ["immediate", "1 week", "2 weeks", "1 month", "ongoing"],
            "description": "When to execute"
          }
        }
      }
    },
    "synthesis_confidence": {
      "type": "integer",
      "minimum": 0,
      "maximum": 100,
      "description": "Overall confidence in recommendations (0-100%)"
    },
    "confidence_rationale": {
      "type": "string",
      "description": "Why this confidence level?"
    }
  }
}
```

---

### 6. MARKDOWN BRIEF OUTPUT SCHEMA

**Output from:** Markdown Formatter  
**Used by:** Email Sender, Webhook Response

```markdown
# Research Brief: [Research Topic]

**Generated:** [ISO 8601 timestamp]
**Confidence Level:** [Synthesis Confidence]%

---

## ✅ What We Know

[Executive Summary]

### Key Insights

1. **[Insight 1]** ([Confidence]% confidence)
   - Why it matters: [Implications]

2. **[Insight 2]** ([Confidence]% confidence)
   - Why it matters: [Implications]

---

## ⚠️ What We're Unsure About

### Conflicting Views

**Conflict 1:** [View A] vs. [View B]
- Resolution: [Resolved / Unresolved]
- Recommendation: [Action]

### Open Questions

1. **[Question 1]**
   - Why it matters: [Impact if wrong]
   - Data needed: [What information needed]

2. **[Question 2]**
   - Why it matters: [Impact if wrong]
   - Data needed: [What information needed]

---

## 🎯 Why This Matters

[Strategic implications for the audience]

---

## ➡️ What to Do Next

1. **[Action 1]** - [Rationale]
   - Timeline: [When]

2. **[Action 2]** - [Rationale]
   - Timeline: [When]

3. **[Action 3]** - [Rationale]
   - Timeline: [When]

---

## 📊 Research Evolution

- New insights this round: [N]
- Repeated findings: [N]
- Evolution score: [%]

---

## 🔍 Methodology

- **Analysis confidence:** [%]
- **Skepticism applied:** Yes
- **Reasoning:** [Confidence Rationale]

---

**Next research run recommended:** [Timeframe]
```

---

### 7. GOOGLE SHEETS ARCHIVE SCHEMA

**Table:** research_memory  
**Purpose:** Historical tracking for comparison engine

```
Column A: date (ISO timestamp)
Column B: topic (research topic text)
Column C: findings_hash (count of facts extracted)
Column D: key_insights (JSON stringified)
Column E: conflicting_views (count unresolved)
Column F: confidence_score (synthesis_confidence 0-100)
Column G: evolution_score (% new vs repeated)
Column H: next_actions_count (number of recommendations)
Column I: run_id (unique identifier)

Example Row:
A: 2026-02-24T14:30:00Z
B: "Remote work productivity"
C: 7
D: "[{"insight":"Hybrid beats remote","confidence":74}]"
E: 1
F: 71
G: 60
H: 4
I: a7d9k3f2m
```

---

## NODE SPECIFICATIONS

### OpenRouter Models Used

**All Three Agents Use:**
```
Model:           z-ai/glm-4.5-air:free
Provider:        OpenRouter (free tier)
API Endpoint:    https://openrouter.ai/api/v1/chat/completions
Type:            LangChain ChatOpenRouter
```

**Why GLM-4.5-Air:**
- Free tier available
- Strong reasoning capability
- Fast inference (~2-5 seconds per agent)
- Reliably outputs JSON
- Good instruction following

**Alternative Models (if GLM-4.5 unavailable):**
- `mistral/mistral-7b-instruct` (faster, lower quality)
- `meta-llama/llama-2-70b-chat` (slower, higher quality)
- `openai/gpt-3.5-turbo` (if available, paid)

### Token Budget Per Run

```
Research Agent:    1500 tokens max
  Input:           ~200-300 tokens (prompt)
  Output:          ~300-600 tokens (facts + analysis)
  
Skeptic Agent:     1500 tokens max
  Input:           ~300-400 tokens (prompt + research output)
  Output:          ~200-400 tokens (critiques)
  
Synthesizer Agent: 2000 tokens max
  Input:           ~400-500 tokens (prompt + research + skeptic)
  Output:          ~600-1000 tokens (synthesis)

Total per run:     4000-5000 tokens
Cost (OpenRouter): ~$0.01-0.02 USD (if paid tier)
                  Free (if free tier available)
```

---

## API CONNECTIONS

### OpenRouter API
```
Auth Method:     API Key (header: Authorization: Bearer sk-or-...)
Endpoint:        https://openrouter.ai/api/v1/chat/completions
Method:          POST
Timeout:         60 seconds
Retry Logic:     Automatic (3 attempts)
```

### Gmail API
```
Auth Method:     OAuth 2.0
Endpoint:        https://www.googleapis.com/gmail/v1/users/me/messages/send
Method:          POST
Permissions:     send, read
Timeout:         10 seconds
```

### Google Sheets API
```
Auth Method:     OAuth 2.0
Endpoint:        https://sheets.googleapis.com/v4/spreadsheets/{id}/values:append
Method:          POST
Permissions:     write, read
Timeout:         10 seconds
```

---

## ERROR HANDLING STRATEGY

### Node-Level Errors

**Input Validator:**
- Missing research_topic → Error + descriptive message
- Invalid email → Defaults to no email (continues)

**Parse Output:**
- JSON parse fails → Error with context
- Retry up to 3 times

**Agent Nodes:**
- Timeout (60s) → Retry once
- Invalid credentials → Fail with clear message
- Rate limit (OpenRouter) → Retry with exponential backoff

**Email Sender:**
- Gmail auth fails → Log error, continue
- Invalid recipient → Log error, continue

**Memory Archiver:**
- Google Sheets auth fails → Log warning, continue
- Append fails → Retry once

### Workflow-Level Errors
```
Error in any node → Execution stops
Execution failure recorded → Available in n8n history
User notified → Via webhook response (if active)
Debugging → Check full JSON in n8n logs
```

---

## PERFORMANCE CHARACTERISTICS

### Response Time Distribution
```
Quick Research:
  P50:  60 seconds
  P95:  75 seconds
  P99:  90 seconds

Deep Research:
  P50:  90 seconds
  P95:  110 seconds
  P99:  140 seconds
```

### Resource Usage
```
Memory:   ~50-100 MB per execution
CPU:      Low (mostly network I/O waiting)
Network:  3 HTTP requests (to OpenRouter + optional Gmail + Sheets)
Storage:  ~1-5 KB per execution (n8n history + Sheets row)
```

### Concurrency
```
Free Tier (OpenRouter):  ~5-10 concurrent requests safe
Self-Hosted n8n:        Limited by your server
Recommended:            Queue requests (don't send all at once)
```

---

## SECURITY CONSIDERATIONS

### Credential Storage
```
OpenRouter API Key:     Encrypted in n8n
Gmail OAuth:            Encrypted, refreshed automatically
Google Sheets:          Encrypted, scoped to specific sheet
```

### Data Privacy
```
Research Questions:     NOT logged to OpenRouter
User Email:             NOT stored anywhere except Sheets
Outputs:                Stored in n8n history (configurable retention)
Memory:                 Stored in user's Google Sheets (user controlled)
```

### Access Control
```
Webhook:                Public (anyone with URL can POST)
n8n Dashboard:          Protected by n8n authentication
Google Sheets:          Requires explicit sharing
Gmail:                  Uses OAuth 2.0 (user's own account)
```

**Recommendations:**
- Change webhook path from default
- Use API key authentication if available
- Limit Sheet access to specific team members
- Review n8n history retention policy

---

## EXTENSION POINTS

### Easy Extensions
1. **Add Slack notification** → Insert Slack node after Email
2. **Add PDF export** → Insert PDF generation before Email
3. **Add second researcher agent** → Duplicate Research Agent node, different prompt
4. **Add persistent memory** → Connect Google Sheets lookup before Research Agent

### Complex Extensions
1. **Multiple workflow trigger** → Add Schedule trigger alongside Webhook
2. **Debate agent** → Add 4th AI agent that argues opposite of Synthesizer
3. **Source ranking** → Pre-process research with Authority Scorer node
4. **Multi-language** → Add language detection + translation nodes

---

## MONITORING & OBSERVABILITY

### Metrics to Track
```
execution_count:           Total runs
execution_success_rate:    % completing successfully
average_execution_time:    Mean duration
synthesis_confidence_avg:  Average confidence score
open_questions_per_run:    Avg unresolved items
next_actions_per_run:      Avg recommendations

Per Agent:
research_agent_output_tokens:    Actual tokens used
skeptic_agent_challenge_count:   Weak claims found
synthesizer_confidence_range:    Distribution
```

### Logging
```
Level:     INFO, WARN, ERROR
Format:    JSON (structured logs)
Retention: 90 days (configurable)
Export:    Available via n8n API
```

### Alerts
```
If execution_success_rate < 90%  → Investigate
If average_execution_time > 180s → Check for slowdowns
If synthesis_confidence < 40%    → Review topic specificity
If open_questions > 10           → Topic too ambiguous
```

---

## DEPLOYMENT CHECKLIST

- [ ] OpenRouter API key created and tested
- [ ] n8n credential configured with API key
- [ ] Gmail OAuth 2.0 credentials (if using email)
- [ ] Google Sheets created with research_memory table
- [ ] Workflow JSON imported
- [ ] All nodes connected (verify from diagram)
- [ ] Webhook path configured
- [ ] Test webhook executed successfully
- [ ] Email delivery verified
- [ ] Sheets append verified
- [ ] Execution history reviewed
- [ ] Error handling tested
- [ ] Monitoring set up
- [ ] Team trained on usage
- [ ] Runbook created
- [ ] Production activation

---

**Document Version:** 1.0  
**Last Updated:** February 2026  
**Status:** Production Ready
