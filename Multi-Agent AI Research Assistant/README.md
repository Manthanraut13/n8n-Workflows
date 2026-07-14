# Multi-Agent AI Research Assistant
## Complete Documentation Package

**Status:** Production Ready  
**Version:** 1.0  
**Last Updated:** February 2026

---

## 📋 Documentation Overview

This package contains **4 comprehensive guides** for deploying and using the Multi-Agent AI Research Assistant workflow:

| Document | Purpose | Audience | Time |
|----------|---------|----------|------|
| **QUICK_START_GUIDE.md** | Get running in 5 minutes | Developers, First-time users | 5 min |
| **WORKFLOW_DOCUMENTATION.md** | Complete technical reference | Engineers, DevOps, Maintainers | 30 min |
| **ARCHITECTURE_REFERENCE.md** | Data schemas & system design | Architects, Advanced users | 20 min |
| **README.md** (this file) | Navigation & overview | Everyone | 3 min |

---

## 🚀 START HERE

### First Time? → **QUICK_START_GUIDE.md**
- 5-minute setup
- Copy-paste examples
- Troubleshooting tips
- How to send your first research request

### Need Details? → **WORKFLOW_DOCUMENTATION.md**
- Node-by-node breakdown (11 nodes)
- Complete agent specifications
- Setup instructions (detailed)
- Testing guide
- Troubleshooting (30 solutions)

### Building On It? → **ARCHITECTURE_REFERENCE.md**
- Data flow diagrams
- JSON schemas (all 7 schemas)
- API connections
- Performance metrics
- Extension points

### Just browsing? → **README.md** (you are here)
- Quick overview
- What this does
- Key features
- Getting help

---

## 💡 WHAT THIS DOES

The **Multi-Agent AI Research Assistant** automates expert-level research synthesis:

```
You:  "Research remote work trends"
   ↓
Workflow:
  1. Research Agent extracts facts
  2. Skeptic Agent challenges findings
  3. Synthesizer Agent reconciles & recommends
   ↓
You get: Decision-ready research brief in your email
```

**Key Innovation:** Agents actively disagree. Contradictions are features, not hidden.

---

## 🎯 KEY FEATURES

✅ **Three AI Agents Working Together**
- Research Agent: Extract facts (temperature 0.3)
- Skeptic Agent: Challenge findings (temperature 0.5)
- Synthesizer Agent: Recommend actions (temperature 0.4)

✅ **Explicit Contradiction Handling**
- Surface where agents disagree
- Flag unresolved conflicts
- Rate confidence for each insight

✅ **Decision-Ready Output**
- Markdown research brief
- "What we know" + "What we're unsure about"
- Concrete next actions
- Confidence scores (0-100%)

✅ **Historical Memory**
- Google Sheets archiving (optional)
- Track research evolution
- Detect patterns over time

✅ **Zero Paid Dependencies**
- OpenRouter free tier
- Gmail free API
- Google Sheets free
- n8n (self-hosted or your account)

✅ **Production-Grade Architecture**
- Sequential agent execution (not parallel)
- Automatic JSON parsing
- Error handling & retries
- Audit trail for every run

---

## 📊 WORKFLOW ARCHITECTURE

```
Webhook Trigger
    ↓
Input Validator & Memory Loader
    ↓
Research Agent (AI) ← OpenRouter Chat Model
    ├─ Extracts facts
    ├─ Evidence strength
    ├─ Contradictions
    └─ Assumptions
    ↓
JSON Parser (cleanup)
    ↓
Skeptic Agent (AI) ← OpenRouter Chat Model 1
    ├─ Weak claims
    ├─ Logical gaps
    ├─ Missing context
    └─ Alternatives
    ↓
Synthesizer Agent (AI) ← OpenRouter Chat Model 2
    ├─ Executive summary
    ├─ Key insights
    ├─ Conflicting views
    ├─ Open questions
    └─ Next actions
    ↓
Comparison Engine (track evolution)
    ↓
Markdown Formatter
    ↓
    ├─ Email Sender → Your inbox
    └─ Memory Archiver → Google Sheets
```

---

## ⚡ QUICK EXAMPLE

### Input
```json
{
  "research_topic": "Is remote work productivity real?",
  "audience": "HR leaders",
  "depth_level": "deep"
}
```

### Output (Email)
```markdown
# Research Brief: Is remote work productivity real?

**Confidence:** 71%

## ✅ What We Know

Hybrid work improves output by 10-20% but increases burnout. 
Effect varies by role and management quality.

### Key Insights

1. Hybrid > Remote (74% confidence)
2. Output gains are real but overstated (68% confidence)
3. Role matters: Engineering ↑30%, Sales ↑5% (62% confidence)

## ⚠️ What We're Unsure About

Conflicting View: +15% productivity everywhere vs. varies 0-30% by role
→ UNRESOLVED — need role-specific data

## ➡️ What to Do Next

1. Segment policy by role (30 days)
2. Measure for 8 weeks (ongoing)
3. Survey wellbeing (immediate)
```

---

## 🛠 SYSTEM REQUIREMENTS

### Minimal Setup
- n8n instance (self-hosted or SaaS)
- OpenRouter account (free tier)
- Internet connection

### Optional
- Gmail account (for email delivery)
- Google Drive account (for memory archiving)

### NOT Required
- Database
- Additional servers
- Paid APIs

---

## 📈 PERFORMANCE

| Metric | Value |
|--------|-------|
| **Quick Research** | 45-60 seconds |
| **Deep Research** | 90-120 seconds |
| **Tokens per run** | 3,000-5,000 |
| **Cost (free tier)** | $0 |
| **Concurrent requests** | 5-10 (free tier) |

---

## 🔐 SECURITY

- API keys encrypted in n8n
- OAuth 2.0 for Gmail/Sheets
- No data logging to external services
- User controls data retention
- Audit trail available in n8n history

---

## 📚 USING THIS DOCUMENTATION

### I want to... → Read this guide

**Deploy the workflow**
→ QUICK_START_GUIDE.md (Sections 1-5)

**Configure OpenRouter**
→ QUICK_START_GUIDE.md (Section 1) + WORKFLOW_DOCUMENTATION.md (Section 6)

**Understand each node**
→ WORKFLOW_DOCUMENTATION.md (Section 3)

**Fix an error**
→ WORKFLOW_DOCUMENTATION.md (Section 9: Troubleshooting)

**Test the workflow**
→ WORKFLOW_DOCUMENTATION.md (Section 8)

**Modify agent prompts**
→ WORKFLOW_DOCUMENTATION.md (Section 4: Agent Specifications)

**See all JSON schemas**
→ ARCHITECTURE_REFERENCE.md (Section: Data Schemas)

**Understand data flow**
→ ARCHITECTURE_REFERENCE.md (Section: System Architecture)

**Extend with new features**
→ ARCHITECTURE_REFERENCE.md (Section: Extension Points)

**Monitor in production**
→ ARCHITECTURE_REFERENCE.md (Section: Monitoring & Observability)

---

## 🎓 LEARNING PATH

### Day 1: Setup (30 minutes)
1. Read QUICK_START_GUIDE.md
2. Create OpenRouter account
3. Import workflow to n8n
4. Send test webhook request
5. Receive research brief in email ✓

### Day 2-3: Usage (1 hour)
1. Send 3-5 real research requests
2. Review quality of output
3. Check confidence scores
4. Note any customizations needed

### Week 1: Customization (2-3 hours)
1. Read WORKFLOW_DOCUMENTATION.md completely
2. Modify agent prompts for your use case
3. Configure Gmail + Sheets (optional)
4. Set up monitoring

### Week 2+: Operations (ongoing)
1. Integrate with your decision workflow
2. Archive research for historical comparison
3. Refine based on usage patterns
4. Document custom modifications

---

## 🆘 GETTING HELP

### Problem Type → Solution

**"I don't know where to start"**
→ Read QUICK_START_GUIDE.md, Section 1

**"Setup failed"**
→ WORKFLOW_DOCUMENTATION.md, Section 6: Configuration Guide

**"Workflow is slow"**
→ ARCHITECTURE_REFERENCE.md, Section: Performance Characteristics

**"Agent outputs are generic"**
→ WORKFLOW_DOCUMENTATION.md, Section 4: Agent Specifications (modify prompts)

**"I need to extend the workflow"**
→ ARCHITECTURE_REFERENCE.md, Section: Extension Points

**"I want to understand the architecture"**
→ ARCHITECTURE_REFERENCE.md, Section: System Architecture

**"I'm getting errors"**
→ WORKFLOW_DOCUMENTATION.md, Section 9: Troubleshooting (30 solutions listed)

---

## 📋 DEPLOYMENT CHECKLIST

```
Setup Phase (Day 1)
□ OpenRouter API key created
□ n8n credential added
□ Workflow JSON imported
□ Webhook URL noted

Testing Phase (Day 1-2)
□ Test webhook sent
□ Research brief received
□ Email delivery confirmed
□ JSON output validated

Optional Setup (Week 1)
□ Gmail OAuth configured
□ Google Sheets created
□ Memory archiver tested

Production Phase (Week 2)
□ Monitoring configured
□ Team trained
□ Documentation shared
□ Runbook created
□ Workflow activated
```

---

## 🔄 TYPICAL WORKFLOW (End-to-End)

```
Hour 1: Research Question
  → "Should we adopt AI agents for customer support?"
  → Send webhook to n8n

Hours 2-3: Workflow Execution
  → 60-120 seconds processing
  → 3 agents analyzing
  → Contradiction detection
  → Markdown formatting

Hour 4: Decision
  → Read research brief
  → Review confidence scores
  → Check open questions
  → Follow recommended actions
```

---

## 📊 OUTPUT QUALITY FACTORS

### Confidence Score (0-100%)

**80-100%** = Decision-ready
- Multiple sources agree
- Skeptic found no major issues
- Clear recommended actions

**60-79%** = Probably true
- Some conflicting evidence
- May need validation before major decisions

**40-59%** = Uncertain
- Significant disagreement
- Need more data

**0-39%** = Unknown
- Don't rely on this

### How to Improve Output Quality

1. **Be specific with research_topic**
   - ❌ "the future of work"
   - ✅ "AI impact on legal research by 2025"

2. **Specify your audience**
   - ✅ Helps synthesizer recommend relevant actions
   - Affects implications and next steps

3. **Use "deep" for important decisions**
   - Quick = 45-60s (good for exploratory)
   - Deep = 90-120s (better for decision-making)

4. **Review open_questions section**
   - These are the knowledge gaps
   - Get data on these questions to improve future runs

---

## 🚀 NEXT STEPS

1. **Right now:** Choose your guide above based on your role
2. **Next 30 mins:** Complete setup from QUICK_START_GUIDE.md
3. **Today:** Send your first research request
4. **This week:** Integrate into your workflow
5. **Ongoing:** Track output quality and refine

---

## 📞 SUPPORT RESOURCES

**Setup Issues:** QUICK_START_GUIDE.md → Section 1

**Technical Issues:** WORKFLOW_DOCUMENTATION.md → Section 9 (30 solutions)

**Understanding Architecture:** ARCHITECTURE_REFERENCE.md → Section: System Architecture

**Customization:** WORKFLOW_DOCUMENTATION.md → Section 7: Configuration Guide

**Error Messages:** WORKFLOW_DOCUMENTATION.md → Section 9: Troubleshooting

---

## 📝 VERSION HISTORY

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | Feb 2026 | Initial release |
| | | • 11 nodes (webhook → email) |
| | | • 3 specialized AI agents |
| | | • Complete documentation |
| | | • Production-ready |

---

## 🎯 PHILOSOPHY

This workflow is built on three principles:

1. **Skepticism Scales**
   - Good research requires contradiction detection
   - Disagreement is a feature, not a bug
   - Confidence scores are honest

2. **Decision-Ready Output**
   - No summaries—synthesis
   - No recommendations—actions
   - Written for busy people

3. **Zero Vendor Lock-In**
   - Uses free-tier APIs only
   - Portable across n8n instances
   - Data stays in your control

---

## 📄 FILE MANIFEST

```
Documentation Package Contents:

README.md (this file)
├─ Overview and navigation guide
├─ Quick example
└─ Learning path

QUICK_START_GUIDE.md
├─ 5-minute setup
├─ Copy-paste examples
├─ Common mistakes
└─ Troubleshooting

WORKFLOW_DOCUMENTATION.md
├─ Complete technical reference
├─ 11 node breakdown
├─ Agent specifications
├─ Setup instructions
├─ Testing guide
└─ 30 troubleshooting solutions

ARCHITECTURE_REFERENCE.md
├─ System architecture
├─ 7 JSON data schemas
├─ API specifications
├─ Performance metrics
├─ Extension points
└─ Monitoring guide

Multi-Agent_AI_Research_Assistant.json
└─ Workflow export (ready to import)

Multi-Agent_AI_Research_Assistant.png
└─ Visual workflow diagram
```

---

## 🙋 FREQUENTLY ASKED QUESTIONS

**Q: How long does each run take?**  
A: Quick = 45-60s, Deep = 90-120s (includes OpenRouter latency)

**Q: Can I run multiple requests simultaneously?**  
A: Yes, up to 5-10 with free tier (OpenRouter rate limit)

**Q: What if an agent gives bad output?**  
A: Modify the prompt (WORKFLOW_DOCUMENTATION.md Section 4). Agents are deterministic—better prompt = better output.

**Q: Can I add a 4th agent?**  
A: Yes! See ARCHITECTURE_REFERENCE.md → Extension Points

**Q: Is my research data private?**  
A: Yes. Data stays on n8n + your Google Sheets. Not sent to OpenRouter logs.

**Q: What models are supported?**  
A: Default is GLM-4.5-Air. See ARCHITECTURE_REFERENCE.md for alternatives.

**Q: Can I export research briefs as PDF?**  
A: Not built-in. Add as extension (see WORKFLOW_DOCUMENTATION.md Section 7)

---

## 💡 TIPS FOR SUCCESS

1. **Be specific:** Vague topics → vague output
2. **Set audience:** Changes recommendations significantly
3. **Use depth_level:** "deep" for high-stakes decisions
4. **Review open_questions:** These are knowledge gaps to close
5. **Track evolution:** Use memory feature to detect shifts
6. **Adjust prompts:** Each organization has unique needs
7. **Integrate gradually:** Start with exploratory questions

---

## 🎉 YOU'RE READY

You now have:
- ✅ Complete documentation
- ✅ Working workflow (ready to import)
- ✅ Multiple guides for different needs
- ✅ Troubleshooting resources

**Next: Pick your starting guide above and begin!**

---

**Questions?** Open the relevant guide above.  
**Ready to go?** → QUICK_START_GUIDE.md  
**Want deep dive?** → WORKFLOW_DOCUMENTATION.md  
**Need architecture details?** → ARCHITECTURE_REFERENCE.md

---

**Made for teams that research at scale.**  
**Built with zero vendor lock-in.**  
**Designed for decision-makers.**

