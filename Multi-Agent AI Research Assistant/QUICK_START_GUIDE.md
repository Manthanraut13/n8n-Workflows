# Multi-Agent AI Research Assistant
## Quick Start Guide (5-Minute Setup)

---

## TL;DR

This workflow takes a research question and returns a decision-ready brief by running 3 AI agents that think critically:
1. **Research Agent** → Extract facts
2. **Skeptic Agent** → Challenge findings
3. **Synthesizer Agent** → Reconcile & recommend actions

---

## 1. ONE-TIME SETUP (10 minutes)

### Step 1: Get OpenRouter API Key
- Visit https://openrouter.ai/sign-up
- Create account
- Go to Settings → Keys
- Copy API key

### Step 2: Add to n8n
1. Open n8n Credentials
2. Click "+ Create New"
3. Select "OpenRouter API"
4. Paste API key
5. Name it: "OpenRouter account for AI Journal"
6. Save

### Step 3: (Optional) Set Up Gmail
- If you want email delivery: Add Gmail OAuth credentials in n8n
- If not: Just skip, you'll see output in n8n

### Step 4: Import Workflow
1. Download `Multi-Agent_AI_Research_Assistant.json`
2. In n8n: Click "Workflow" → "Import"
3. Paste JSON
4. Match credentials to what you created above
5. Click "Import"

### Step 5: Test It
```bash
curl -X POST http://localhost:5678/webhook/research-assistant \
  -H "Content-Type: application/json" \
  -d '{
    "research_topic": "AI adoption in enterprises 2025",
    "industry": "enterprise",
    "audience": "CTOs"
  }'
```

Wait 60-90 seconds. Check email or n8n execution history for result.

**Done!** You can now use it.

---

## 2. HOW TO USE

### Via Curl (Developers)
```bash
curl -X POST [YOUR_WEBHOOK_URL] \
  -H "Content-Type: application/json" \
  -d '{
    "research_topic": "Your question here",
    "industry": "your industry",
    "audience": "who this is for",
    "depth_level": "quick"
  }'
```

### Via Postman
1. Create POST request to webhook URL
2. Body (JSON):
```json
{
  "research_topic": "Your research question",
  "industry": "technology",
  "audience": "decision-makers",
  "depth_level": "quick"
}
```

### Via Slack Bot (Future Extension)
Soon: `/research ai adoption in healthcare`

---

## 3. EXAMPLE: Research Remote Work Trends

### Input
```json
{
  "research_topic": "Is remote work productivity real or myth?",
  "industry": "Technology",
  "audience": "HR leaders",
  "depth_level": "deep"
}
```

### Output (Markdown Email)
```markdown
# Research Brief: Is remote work productivity real or myth?

**Confidence:** 71%

## ✅ What We Know

Hybrid work improves measured output by 10-20% but increases burnout risk. 
The effect varies dramatically by role and management quality.

### Key Insights

1. **Hybrid > Remote-Only** (74% confidence)
   - Why: People miss collaboration but love flexibility
   
2. **Output gains are real but overstated** (68% confidence)
   - Why: Measured as lines of code, not impact; varies by role

## ⚠️ What We're Unsure About

**Conflicting Views:**
- View A: +15% productivity across all roles
- View B: Varies 0% to +30% by role
- Resolution: **UNRESOLVED** — need role-specific data

**Open Questions:**
1. Do sales teams actually benefit? 
   - Why it matters: Sales reps may need different policies
   - Data needed: Sales performance by location

## 🎯 Why This Matters

**For your company:** If 60% engineering + 40% sales, expected gain is 10-12%. 
If you optimize for engineering, sales productivity drops.

## ➡️ What to Do Next

1. **Segment policy by role** (30 days)
   - Engineering: Remote-friendly
   - Sales: Office-first
   
2. **Measure for 8 weeks** (Ongoing)
   - Track output metrics by location
   
3. **Survey wellbeing** (Immediate)
   - 23% burnout risk needs baseline

---
```

---

## 4. PARAMETERS EXPLAINED

| Parameter | Required | Options | Default | What It Does |
|-----------|----------|---------|---------|--------------|
| `research_topic` | YES | Any string | — | The research question to analyze |
| `industry` | NO | Any string | "general" | Helps agents contextualize topic |
| `audience` | NO | Any string | "decision-makers" | Who the brief is for (shapes recommendations) |
| `depth_level` | NO | "quick" or "deep" | "quick" | Quick = 3 sources, Deep = 5 sources (slower) |
| `recipient_email` | NO | Valid email | — | Where to send Markdown brief |

### Quick vs. Deep

**Quick (45-60 seconds)**
- 3 knowledge sources
- Good for: Fast decisions, simple topics
- Cost: Lowest

**Deep (90-120 seconds)**
- 5 knowledge sources
- Good for: Complex topics, high-stakes decisions
- Cost: Highest

---

## 5. WHAT YOU GET

### Email (If configured)
- Markdown research brief
- Subject: "Research Brief: [Topic] — [Date]"
- Includes: Summary, Insights, Conflicts, Open Questions, Next Actions
- Human-readable format

### Execution History (n8n)
- Full JSON from each agent
- See exact reasoning
- Debug mode for troubleshooting

### Google Sheets (If configured)
- Each research run appended as row
- Track evolution over time
- Historical comparison automatic

---

## 6. WHAT THE AGENTS DO

### Research Agent
**Says:** "Here's what we know about this topic"

**Outputs:**
- Facts (with evidence strength: high/medium/low)
- Recurring themes
- Numerical data
- Known contradictions
- Hidden assumptions

### Skeptic Agent
**Says:** "I disagree. Here's what's weak"

**Challenges:**
- Claims with only 1 source
- Logical leaps
- Missing context
- Alternative explanations

### Synthesizer Agent
**Says:** "Both of you have points. Here's what to do"

**Produces:**
- Executive summary
- Decision-ready insights
- Open questions
- Concrete next actions
- Confidence scores

---

## 7. OUTPUT CONFIDENCE SCORES

**80-100%** = Decision-ready. Act on this.
**60-79%** = Probably true. Validate before major decisions.
**40-59%** = Uncertain. Need more data.
**0-39%** = Unknown. Don't rely on this.

---

## 8. COMMON MISTAKES

### ❌ DON'T
- Send vague topic ("the future")
- Expect instant results (takes 60-90 seconds)
- Use without specifying audience
- Assume 100% confidence on any insight

### ✅ DO
- Be specific ("AI regulation in financial services 2025")
- Specify audience ("CIOs at Fortune 500 companies")
- Use "deep" for high-stakes decisions
- Review open_questions section

---

## 9. TROUBLESHOOTING (30 seconds)

### Issue: Email not received
**Fix:** Check spam folder, verify Gmail credential, resend test

### Issue: Empty output
**Fix:** Make research_topic more specific, use different industry

### Issue: Agents all agree (no conflicts)
**Fix:** That's normal for simple topics. Try complex topic like "AI safety vs. speed"

### Issue: Takes >2 minutes
**Fix:** Normal for deep analysis. Check n8n logs for errors.

**Still stuck?** See full WORKFLOW_DOCUMENTATION.md → Section 9: Troubleshooting

---

## 10. EXAMPLES TO TRY

```json
// Example 1: Market entry decision
{
  "research_topic": "Should we enter the Indian SaaS market?",
  "industry": "SaaS",
  "audience": "Product executives",
  "depth_level": "deep"
}

// Example 2: Technology decision
{
  "research_topic": "Kotlin vs. Java for backend microservices",
  "industry": "Technology",
  "audience": "Engineering leads",
  "depth_level": "quick"
}

// Example 3: HR decision
{
  "research_topic": "4-day work week productivity impact",
  "industry": "HR Tech",
  "audience": "HR leaders",
  "depth_level": "deep"
}

// Example 4: Sales decision
{
  "research_topic": "AI-powered sales assistants ROI in B2B",
  "industry": "Sales Tech",
  "audience": "VP of Sales",
  "depth_level": "deep"
}
```

---

## 11. COST & RATE LIMITS

### Cost
- OpenRouter free tier: **FREE** (limited calls/day)
- Gmail API: **FREE** (15K emails/day)
- n8n self-hosted: **YOUR COST**

### Rate Limits
- OpenRouter free tier: ~5-10 requests/hour
- Gmail: 15K emails/day

### Monitor Usage
- Check n8n execution history for token count
- OpenRouter account shows tokens/month

---

## 12. NEXT STEPS

1. **Immediate:** Run 1 test workflow (this guide → Step 5)
2. **This week:** Use for 3 real research questions
3. **Next month:** Integrate with your decision-making process
4. **Future:** Add Slack bot, PDF export, or custom agents

---

## 13. GET HELP

**Documentation:** Open `WORKFLOW_DOCUMENTATION.md`

**Common Issues:** Section 9 of full docs

**Want to customize?** See Section 7: Configuration Guide in full docs

**Report bugs:** Include execution ID from n8n logs

---

**Ready?** Send your first webhook request above and wait 60 seconds for the research brief in your email. 🚀

