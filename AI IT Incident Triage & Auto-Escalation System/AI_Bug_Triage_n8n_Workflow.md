# AI Bug Report Triage & Smart Developer Assignment
## Complete n8n Workflow — Setup Guide

---

## Table of Contents
1. [Architecture Overview](#1-architecture-overview)
2. [Tools & APIs Required](#2-tools--apis-required)
3. [API Key Setup Instructions](#3-api-key-setup-instructions)
4. [n8n Credentials Setup](#4-n8n-credentials-setup)
5. [Importable Workflow JSON](#5-importable-workflow-json)
6. [Node-by-Node Configuration](#6-node-by-node-configuration)
7. [Developer Routing Map (Code Node Logic)](#7-developer-routing-map-code-node-logic)
8. [Testing the Workflow](#8-testing-the-workflow)
9. [Customization Guide](#9-customization-guide)

---

## 1. Architecture Overview

```
Sentry Webhook / Manual Trigger
         │
         ▼
┌─────────────────────┐
│  Node 1: Webhook    │  ← Receives bug reports via HTTP POST
│  (Bug Intake)       │
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│  Node 2: Set        │  ← Normalizes & extracts fields from
│  (Normalize Data)   │     different source formats
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│  Node 3: HTTP Req   │  ← Calls OpenAI API with bug details
│  (AI Triage Call)   │     Stack trace + error + description
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│  Node 4: Code       │  ← Parses AI JSON response, looks up
│  (Parse & Map)      │     developer routing table
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│  Node 5: Switch     │  ← Routes by severity level
│  (Severity Router)  │
└───┬────┬────┬───────┘
    │    │    │
Critical High Medium/Low
    │    │    │
    ▼    ▼    ▼
┌─────────────────────┐
│  Node 6: HTTP Req   │  ← Creates Jira ticket (priority mapped)
│  (Create Jira Ticket│
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│  Node 7: Slack      │  ← Pings assigned developer with context,
│  (Notify Developer) │     Jira link, and AI root-cause hypothesis
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│  Node 8: Respond    │  ← Returns 200 OK to caller
│  (Webhook Response) │
└─────────────────────┘

━━━ Error Path ━━━
Error Trigger → Slack #alerts → Log failure
```

**Pattern**: Webhook Processing + AI HTTP Integration + Branching Flow

---

## 2. Tools & APIs Required

| Service | Purpose | Free Tier? |
|---------|---------|------------|
| **OpenAI API** | AI classification, severity scoring, developer matching | Pay-per-use (~$0.002/report) |
| **Jira (Atlassian)** | Create pre-filled bug tickets | Free up to 10 users |
| **Slack** | Notify assigned developer | Free tier available |
| **n8n** | Workflow automation engine | Free self-hosted / cloud |

> **Alternative**: Replace OpenAI with Anthropic Claude API — prompt is compatible with both.

---

## 3. API Key Setup Instructions

### 3.1 OpenAI API Key

1. Go to **https://platform.openai.com/signup** and create an account.
2. Navigate to **API Keys** → click **"Create new secret key"**.
3. Name it `n8n-bug-triage` and copy the key (shown only once).
4. Go to **Billing** → **Add payment method** (required for API access beyond free trial).
5. Optionally set a **Usage Limit** under Settings → Limits (e.g. $10/month) to prevent runaway costs.
6. Store the key as: `sk-proj-xxxxxxxxxxxxxxxxxxxx`

> **Model to use**: `gpt-4o-mini` — fast, cheap, great at structured JSON classification tasks.

---

### 3.2 Jira API Key (Atlassian)

1. Log in to your Jira instance at **https://your-domain.atlassian.net**.
2. Go to **https://id.atlassian.com/manage-profile/security/api-tokens**.
3. Click **"Create API token"** → name it `n8n-bug-triage`.
4. Copy the token immediately (not shown again).
5. You will also need:
   - **Jira domain**: `your-domain.atlassian.net`
   - **Your email** (used as username for Basic Auth)
   - **Project Key**: e.g. `BUG` or `ENG` — find it in Jira → Projects → your project key in the URL.
   - **Issue Type ID**: Usually `Bug` — find via `GET /rest/api/3/issuetype`.

**Jira Auth Format (Basic Auth)**: `email:api_token` encoded in Base64.

In n8n, use the **Header Auth** credential with:
- **Name**: `Authorization`
- **Value**: `Basic ` + base64(`your-email@company.com:your-api-token`)

> **Quick Base64 encode**: Run in terminal: `echo -n "email@co.com:token" | base64`

---

### 3.3 Slack Bot Token

1. Go to **https://api.slack.com/apps** → click **"Create New App"** → **"From scratch"**.
2. Name it `Bug Triage Bot`, select your workspace.
3. Go to **OAuth & Permissions** → under **Bot Token Scopes**, add:
   - `chat:write`
   - `chat:write.public`
   - `im:write` (for DMs to developers)
4. Click **"Install to Workspace"** → **Allow**.
5. Copy the **Bot User OAuth Token**: `xoxb-xxxxxxxxxxxx-xxxxxxxxxxxx-xxxxxxxxxxxxxxxxxxxxxxxx`.
6. Invite the bot to your developer channels: In Slack, open the channel → `/invite @Bug Triage Bot`.

---

## 4. n8n Credentials Setup

In n8n, go to **Settings → Credentials** and create the following:

### Credential 1: OpenAI
- **Type**: `Header Auth`
- **Name**: `OpenAI API`
- **Header Name**: `Authorization`
- **Header Value**: `Bearer sk-proj-your-key-here`

### Credential 2: Jira
- **Type**: `Header Auth`
- **Name**: `Jira Basic Auth`
- **Header Name**: `Authorization`
- **Header Value**: `Basic BASE64_ENCODED_EMAIL_AND_TOKEN`

### Credential 3: Slack
- **Type**: `Slack API`
- **Name**: `Slack Bug Bot`
- **Access Token**: `xoxb-your-slack-bot-token`

---

## 5. Importable Workflow JSON

Copy the entire JSON below. In n8n: **Workflows → + New → ⋮ (menu) → Import from JSON → paste → Save**.

```json
{
  "name": "AI Bug Report Triage & Smart Developer Assignment",
  "nodes": [
    {
      "id": "node-1-webhook",
      "name": "Bug Report Intake",
      "type": "n8n-nodes-base.webhook",
      "typeVersion": 2,
      "position": [200, 300],
      "parameters": {
        "path": "bug-report",
        "httpMethod": "POST",
        "responseMode": "responseNode",
        "options": {}
      }
    },
    {
      "id": "node-2-set",
      "name": "Normalize Bug Data",
      "type": "n8n-nodes-base.set",
      "typeVersion": 3,
      "position": [440, 300],
      "parameters": {
        "mode": "manual",
        "fields": {
          "values": [
            {
              "name": "source",
              "type": "string",
              "value": "={{$json.body.source || $json.body.project || 'unknown'}}"
            },
            {
              "name": "error_message",
              "type": "string",
              "value": "={{$json.body.error || $json.body.title || $json.body.message || 'No error message'}}"
            },
            {
              "name": "stack_trace",
              "type": "string",
              "value": "={{$json.body.stack_trace || $json.body.stacktrace || $json.body.exception || 'No stack trace provided'}}"
            },
            {
              "name": "user_description",
              "type": "string",
              "value": "={{$json.body.description || $json.body.user_description || $json.body.body || 'No description provided'}}"
            },
            {
              "name": "environment",
              "type": "string",
              "value": "={{$json.body.environment || $json.body.env || 'production'}}"
            },
            {
              "name": "reporter",
              "type": "string",
              "value": "={{$json.body.reporter || $json.body.user || $json.body.email || 'anonymous'}}"
            },
            {
              "name": "reported_at",
              "type": "string",
              "value": "={{$now.toISO()}}"
            }
          ]
        },
        "options": {}
      }
    },
    {
      "id": "node-3-openai",
      "name": "AI Triage Analysis",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4,
      "position": [680, 300],
      "parameters": {
        "method": "POST",
        "url": "https://api.openai.com/v1/chat/completions",
        "authentication": "predefinedCredentialType",
        "nodeCredentialType": "httpHeaderAuth",
        "sendBody": true,
        "contentType": "json",
        "body": {
          "model": "gpt-4o-mini",
          "max_tokens": 800,
          "temperature": 0,
          "messages": [
            {
              "role": "system",
              "content": "You are an expert software engineering triage assistant. Analyze bug reports and return ONLY a valid JSON object — no markdown, no explanation, no extra text. Your JSON must follow this exact schema:\n{\n  \"severity\": \"critical|high|medium|low\",\n  \"severity_reason\": \"one sentence explaining severity\",\n  \"affected_module\": \"the service/module/component name\",\n  \"affected_module_reason\": \"one sentence explaining module identification\",\n  \"assigned_developer\": \"developer name based on module (see routing below)\",\n  \"root_cause_hypothesis\": \"2-3 sentence technical hypothesis\",\n  \"reproduction_steps\": [\"step 1\", \"step 2\", \"step 3\"],\n  \"suggested_labels\": [\"label1\", \"label2\"],\n  \"estimated_effort\": \"xs|s|m|l|xl\"\n}\n\nSeverity rules:\n- critical: production down, data loss, security breach, affects all users\n- high: major feature broken, significant user impact, no workaround\n- medium: feature partially broken, workaround exists\n- low: minor UI issue, edge case, cosmetic\n\nModule routing (assign developer based on module):\n- auth/login/oauth/token → Alice Chen\n- payment/billing/stripe/invoice → Bob Martinez\n- api/backend/database/server → Carol Singh\n- frontend/ui/react/css → David Kim\n- mobile/ios/android → Emma Wilson\n- infra/deploy/docker/k8s → Frank Liu\n- notification/email/sms → Grace Patel\n- search/elasticsearch → Henry Brown\n- unknown/other → Carol Singh"
            },
            {
              "role": "user",
              "content": "=Triage this bug report:\n\nSource: {{$json.source}}\nEnvironment: {{$json.environment}}\nError Message: {{$json.error_message}}\n\nStack Trace:\n{{$json.stack_trace}}\n\nUser Description:\n{{$json.user_description}}\n\nReporter: {{$json.reporter}}\nReported At: {{$json.reported_at}}"
            }
          ]
        },
        "options": {
          "timeout": 30000
        }
      },
      "credentials": {
        "httpHeaderAuth": {
          "id": "REPLACE_WITH_CREDENTIAL_ID",
          "name": "OpenAI API"
        }
      }
    },
    {
      "id": "node-4-code",
      "name": "Parse AI Response & Map Developer",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [920, 300],
      "parameters": {
        "mode": "runOnceForAllItems",
        "jsCode": "// Developer contact map (Slack user IDs)\nconst DEVELOPER_MAP = {\n  'Alice Chen':   { slack_id: 'U01ALICE',  email: 'alice@company.com',   timezone: 'US/Pacific' },\n  'Bob Martinez': { slack_id: 'U02BOB',    email: 'bob@company.com',     timezone: 'US/Eastern' },\n  'Carol Singh':  { slack_id: 'U03CAROL',  email: 'carol@company.com',   timezone: 'US/Eastern' },\n  'David Kim':    { slack_id: 'U04DAVID',  email: 'david@company.com',   timezone: 'US/Pacific' },\n  'Emma Wilson':  { slack_id: 'U05EMMA',   email: 'emma@company.com',    timezone: 'US/Eastern' },\n  'Frank Liu':    { slack_id: 'U06FRANK',  email: 'frank@company.com',   timezone: 'Asia/Shanghai' },\n  'Grace Patel':  { slack_id: 'U07GRACE',  email: 'grace@company.com',   timezone: 'Asia/Kolkata' },\n  'Henry Brown':  { slack_id: 'U08HENRY',  email: 'henry@company.com',   timezone: 'US/Pacific' },\n};\n\n// Jira priority mapping\nconst JIRA_PRIORITY = {\n  critical: 'Highest',\n  high: 'High',\n  medium: 'Medium',\n  low: 'Low',\n};\n\n// Emoji mapping for Slack messages\nconst SEVERITY_EMOJI = {\n  critical: '🔴',\n  high: '🟠',\n  medium: '🟡',\n  low: '🟢',\n};\n\nconst results = [];\n\nfor (const item of $input.all()) {\n  try {\n    // Extract AI response text\n    const aiResponseText = item.json.choices?.[0]?.message?.content || '{}';\n    \n    // Clean and parse the JSON (strip any stray markdown)\n    const cleaned = aiResponseText\n      .replace(/```json/g, '')\n      .replace(/```/g, '')\n      .trim();\n    \n    const triage = JSON.parse(cleaned);\n    \n    // Look up developer contact info\n    const devName = triage.assigned_developer || 'Carol Singh';\n    const devInfo = DEVELOPER_MAP[devName] || DEVELOPER_MAP['Carol Singh'];\n    \n    // Build the output item with all fields needed downstream\n    results.push({\n      json: {\n        // Triage results\n        severity:               triage.severity || 'medium',\n        severity_reason:        triage.severity_reason || '',\n        affected_module:        triage.affected_module || 'unknown',\n        affected_module_reason: triage.affected_module_reason || '',\n        assigned_developer:     devName,\n        root_cause_hypothesis:  triage.root_cause_hypothesis || '',\n        reproduction_steps:     (triage.reproduction_steps || []).join('\\n'),\n        suggested_labels:       (triage.suggested_labels || []).join(', '),\n        estimated_effort:       triage.estimated_effort || 'm',\n        \n        // Developer contact\n        dev_slack_id:  devInfo.slack_id,\n        dev_email:     devInfo.email,\n        dev_timezone:  devInfo.timezone,\n        \n        // Jira fields\n        jira_priority: JIRA_PRIORITY[triage.severity] || 'Medium',\n        severity_emoji: SEVERITY_EMOJI[triage.severity] || '🟡',\n        \n        // Pass-through fields from previous nodes\n        source:           item.json.source || 'unknown',\n        error_message:    item.json.error_message || '',\n        stack_trace:      item.json.stack_trace || '',\n        user_description: item.json.user_description || '',\n        environment:      item.json.environment || 'production',\n        reporter:         item.json.reporter || 'anonymous',\n        reported_at:      item.json.reported_at || new Date().toISOString(),\n      }\n    });\n  } catch (err) {\n    // Fallback on parse failure\n    results.push({\n      json: {\n        severity: 'medium',\n        severity_reason: 'AI parse failed — defaulted to medium',\n        affected_module: 'unknown',\n        affected_module_reason: 'Could not determine from error: ' + err.message,\n        assigned_developer: 'Carol Singh',\n        root_cause_hypothesis: 'Manual triage required — AI analysis failed.',\n        reproduction_steps: 'N/A',\n        suggested_labels: 'needs-triage',\n        estimated_effort: 'm',\n        dev_slack_id: 'U03CAROL',\n        dev_email: 'carol@company.com',\n        dev_timezone: 'US/Eastern',\n        jira_priority: 'Medium',\n        severity_emoji: '🟡',\n        source: item.json.source || 'unknown',\n        error_message: item.json.error_message || 'Unknown error',\n        stack_trace: item.json.stack_trace || '',\n        user_description: item.json.user_description || '',\n        environment: item.json.environment || 'production',\n        reporter: item.json.reporter || 'anonymous',\n        reported_at: item.json.reported_at || new Date().toISOString(),\n        parse_error: err.message,\n      }\n    });\n  }\n}\n\nreturn results;"
      }
    },
    {
      "id": "node-5-switch",
      "name": "Severity Router",
      "type": "n8n-nodes-base.switch",
      "typeVersion": 3,
      "position": [1160, 300],
      "parameters": {
        "mode": "rules",
        "rules": {
          "values": [
            {
              "conditions": {
                "options": { "caseSensitive": false },
                "conditions": [
                  {
                    "leftValue": "={{$json.severity}}",
                    "rightValue": "critical",
                    "operator": { "type": "string", "operation": "equals" }
                  }
                ]
              },
              "renameOutput": true,
              "outputKey": "Critical"
            },
            {
              "conditions": {
                "options": { "caseSensitive": false },
                "conditions": [
                  {
                    "leftValue": "={{$json.severity}}",
                    "rightValue": "high",
                    "operator": { "type": "string", "operation": "equals" }
                  }
                ]
              },
              "renameOutput": true,
              "outputKey": "High"
            },
            {
              "conditions": {
                "options": { "caseSensitive": false },
                "conditions": [
                  {
                    "leftValue": "={{$json.severity}}",
                    "rightValue": "medium",
                    "operator": { "type": "string", "operation": "equals" }
                  }
                ]
              },
              "renameOutput": true,
              "outputKey": "Medium"
            }
          ]
        },
        "fallbackOutput": "extra"
      }
    },
    {
      "id": "node-6a-jira-critical",
      "name": "Create Jira Ticket (Critical)",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4,
      "position": [1400, 100],
      "parameters": {
        "method": "POST",
        "url": "https://YOUR_DOMAIN.atlassian.net/rest/api/3/issue",
        "authentication": "predefinedCredentialType",
        "nodeCredentialType": "httpHeaderAuth",
        "sendHeaders": true,
        "headerParameters": {
          "parameters": [
            { "name": "Content-Type", "value": "application/json" },
            { "name": "Accept", "value": "application/json" }
          ]
        },
        "sendBody": true,
        "contentType": "json",
        "body": {
          "fields": {
            "project": { "key": "BUG" },
            "summary": "=[🔴 CRITICAL] {{$json.affected_module}}: {{$json.error_message.substring(0, 80)}}",
            "description": {
              "type": "doc",
              "version": 1,
              "content": [
                {
                  "type": "paragraph",
                  "content": [{ "type": "text", "text": "=**Severity**: CRITICAL — {{$json.severity_reason}}" }]
                },
                {
                  "type": "paragraph",
                  "content": [{ "type": "text", "text": "=**Affected Module**: {{$json.affected_module}} — {{$json.affected_module_reason}}" }]
                },
                {
                  "type": "paragraph",
                  "content": [{ "type": "text", "text": "=**Reporter**: {{$json.reporter}} | **Environment**: {{$json.environment}} | **Reported At**: {{$json.reported_at}}" }]
                },
                {
                  "type": "heading",
                  "attrs": { "level": 3 },
                  "content": [{ "type": "text", "text": "Root Cause Hypothesis" }]
                },
                {
                  "type": "paragraph",
                  "content": [{ "type": "text", "text": "={{$json.root_cause_hypothesis}}" }]
                },
                {
                  "type": "heading",
                  "attrs": { "level": 3 },
                  "content": [{ "type": "text", "text": "Reproduction Steps" }]
                },
                {
                  "type": "paragraph",
                  "content": [{ "type": "text", "text": "={{$json.reproduction_steps}}" }]
                },
                {
                  "type": "heading",
                  "attrs": { "level": 3 },
                  "content": [{ "type": "text", "text": "Stack Trace" }]
                },
                {
                  "type": "codeBlock",
                  "attrs": {},
                  "content": [{ "type": "text", "text": "={{$json.stack_trace.substring(0, 2000)}}" }]
                },
                {
                  "type": "heading",
                  "attrs": { "level": 3 },
                  "content": [{ "type": "text", "text": "User Description" }]
                },
                {
                  "type": "paragraph",
                  "content": [{ "type": "text", "text": "={{$json.user_description}}" }]
                }
              ]
            },
            "issuetype": { "name": "Bug" },
            "priority": { "name": "={{$json.jira_priority}}" },
            "assignee": { "emailAddress": "={{$json.dev_email}}" },
            "labels": ["={{$json.suggested_labels}}", "ai-triaged", "critical"],
            "customfield_10016": "={{$json.estimated_effort === 'xs' ? 1 : $json.estimated_effort === 's' ? 2 : $json.estimated_effort === 'm' ? 3 : $json.estimated_effort === 'l' ? 5 : 8}}"
          }
        }
      },
      "credentials": {
        "httpHeaderAuth": {
          "id": "REPLACE_WITH_JIRA_CREDENTIAL_ID",
          "name": "Jira Basic Auth"
        }
      }
    },
    {
      "id": "node-6b-jira-standard",
      "name": "Create Jira Ticket (Standard)",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4,
      "position": [1400, 380],
      "parameters": {
        "method": "POST",
        "url": "https://YOUR_DOMAIN.atlassian.net/rest/api/3/issue",
        "authentication": "predefinedCredentialType",
        "nodeCredentialType": "httpHeaderAuth",
        "sendHeaders": true,
        "headerParameters": {
          "parameters": [
            { "name": "Content-Type", "value": "application/json" },
            { "name": "Accept", "value": "application/json" }
          ]
        },
        "sendBody": true,
        "contentType": "json",
        "body": {
          "fields": {
            "project": { "key": "BUG" },
            "summary": "={{$json.severity_emoji}} [{{$json.severity.toUpperCase()}}] {{$json.affected_module}}: {{$json.error_message.substring(0, 80)}}",
            "description": {
              "type": "doc",
              "version": 1,
              "content": [
                {
                  "type": "paragraph",
                  "content": [{ "type": "text", "text": "=**Severity**: {{$json.severity.toUpperCase()}} — {{$json.severity_reason}}" }]
                },
                {
                  "type": "paragraph",
                  "content": [{ "type": "text", "text": "=**Affected Module**: {{$json.affected_module}}" }]
                },
                {
                  "type": "paragraph",
                  "content": [{ "type": "text", "text": "=**Reporter**: {{$json.reporter}} | **Environment**: {{$json.environment}} | **Reported At**: {{$json.reported_at}}" }]
                },
                {
                  "type": "heading",
                  "attrs": { "level": 3 },
                  "content": [{ "type": "text", "text": "Root Cause Hypothesis" }]
                },
                {
                  "type": "paragraph",
                  "content": [{ "type": "text", "text": "={{$json.root_cause_hypothesis}}" }]
                },
                {
                  "type": "heading",
                  "attrs": { "level": 3 },
                  "content": [{ "type": "text", "text": "Reproduction Steps" }]
                },
                {
                  "type": "paragraph",
                  "content": [{ "type": "text", "text": "={{$json.reproduction_steps}}" }]
                },
                {
                  "type": "heading",
                  "attrs": { "level": 3 },
                  "content": [{ "type": "text", "text": "Stack Trace" }]
                },
                {
                  "type": "codeBlock",
                  "attrs": {},
                  "content": [{ "type": "text", "text": "={{$json.stack_trace.substring(0, 2000)}}" }]
                }
              ]
            },
            "issuetype": { "name": "Bug" },
            "priority": { "name": "={{$json.jira_priority}}" },
            "assignee": { "emailAddress": "={{$json.dev_email}}" },
            "labels": ["ai-triaged", "={{$json.severity}}"]
          }
        }
      },
      "credentials": {
        "httpHeaderAuth": {
          "id": "REPLACE_WITH_JIRA_CREDENTIAL_ID",
          "name": "Jira Basic Auth"
        }
      }
    },
    {
      "id": "node-7-slack",
      "name": "Notify Developer (Slack)",
      "type": "n8n-nodes-base.slack",
      "typeVersion": 2,
      "position": [1660, 300],
      "parameters": {
        "resource": "message",
        "operation": "post",
        "channel": "={{$json.dev_slack_id}}",
        "text": "={{$json.severity_emoji}} *Bug Assigned to You* — {{$json.severity.toUpperCase()}}",
        "blocksUi": {
          "blocksValues": [
            {
              "type": "section",
              "text": {
                "type": "mrkdwn",
                "text": "={{$json.severity_emoji}} *Bug Assigned to You* — `{{$json.severity.toUpperCase()}}`\n*{{$json.error_message.substring(0, 100)}}*"
              }
            },
            {
              "type": "section",
              "fields": [
                {
                  "type": "mrkdwn",
                  "text": "=*Module*\n{{$json.affected_module}}"
                },
                {
                  "type": "mrkdwn",
                  "text": "=*Environment*\n{{$json.environment}}"
                },
                {
                  "type": "mrkdwn",
                  "text": "=*Reporter*\n{{$json.reporter}}"
                },
                {
                  "type": "mrkdwn",
                  "text": "=*Reported At*\n{{$json.reported_at}}"
                }
              ]
            },
            {
              "type": "section",
              "text": {
                "type": "mrkdwn",
                "text": "=*🤖 AI Root Cause Hypothesis:*\n{{$json.root_cause_hypothesis}}"
              }
            },
            {
              "type": "section",
              "text": {
                "type": "mrkdwn",
                "text": "=*📋 Reproduction Steps:*\n{{$json.reproduction_steps}}"
              }
            },
            {
              "type": "divider"
            },
            {
              "type": "actions",
              "elements": [
                {
                  "type": "button",
                  "text": { "type": "plain_text", "text": "View Jira Ticket" },
                  "url": "=https://YOUR_DOMAIN.atlassian.net/browse/{{$node['Create Jira Ticket (Critical)'].json.key || $node['Create Jira Ticket (Standard)'].json.key}}",
                  "style": "primary"
                }
              ]
            }
          ]
        }
      },
      "credentials": {
        "slackApi": {
          "id": "REPLACE_WITH_SLACK_CREDENTIAL_ID",
          "name": "Slack Bug Bot"
        }
      }
    },
    {
      "id": "node-8-respond",
      "name": "Respond to Webhook",
      "type": "n8n-nodes-base.respondToWebhook",
      "typeVersion": 1,
      "position": [1900, 300],
      "parameters": {
        "respondWith": "json",
        "responseBody": "={\"status\": \"triaged\", \"severity\": \"{{$json.severity}}\", \"assigned_to\": \"{{$json.assigned_developer}}\", \"module\": \"{{$json.affected_module}}\", \"jira_key\": \"CREATED\"}",
        "options": {
          "responseCode": 200
        }
      }
    },
    {
      "id": "node-error-trigger",
      "name": "Error Trigger",
      "type": "n8n-nodes-base.errorTrigger",
      "typeVersion": 1,
      "position": [200, 560],
      "parameters": {}
    },
    {
      "id": "node-error-slack",
      "name": "Alert Team on Error",
      "type": "n8n-nodes-base.slack",
      "typeVersion": 2,
      "position": [440, 560],
      "parameters": {
        "resource": "message",
        "operation": "post",
        "channel": "#engineering-alerts",
        "text": "=⚠️ *Bug Triage Workflow Failed*\nError: {{$json.error.message}}\nWorkflow: {{$json.workflow.name}}\nNode: {{$json.execution.lastNodeExecuted}}\nTime: {{$now.toFormat('yyyy-MM-dd HH:mm:ss')}}\n\nManual triage required for incoming bug reports."
      },
      "credentials": {
        "slackApi": {
          "id": "REPLACE_WITH_SLACK_CREDENTIAL_ID",
          "name": "Slack Bug Bot"
        }
      }
    }
  ],
  "connections": {
    "Bug Report Intake": {
      "main": [[{ "node": "Normalize Bug Data", "type": "main", "index": 0 }]]
    },
    "Normalize Bug Data": {
      "main": [[{ "node": "AI Triage Analysis", "type": "main", "index": 0 }]]
    },
    "AI Triage Analysis": {
      "main": [[{ "node": "Parse AI Response & Map Developer", "type": "main", "index": 0 }]]
    },
    "Parse AI Response & Map Developer": {
      "main": [[{ "node": "Severity Router", "type": "main", "index": 0 }]]
    },
    "Severity Router": {
      "main": [
        [{ "node": "Create Jira Ticket (Critical)", "type": "main", "index": 0 }],
        [{ "node": "Create Jira Ticket (Standard)", "type": "main", "index": 0 }],
        [{ "node": "Create Jira Ticket (Standard)", "type": "main", "index": 0 }],
        [{ "node": "Create Jira Ticket (Standard)", "type": "main", "index": 0 }]
      ]
    },
    "Create Jira Ticket (Critical)": {
      "main": [[{ "node": "Notify Developer (Slack)", "type": "main", "index": 0 }]]
    },
    "Create Jira Ticket (Standard)": {
      "main": [[{ "node": "Notify Developer (Slack)", "type": "main", "index": 0 }]]
    },
    "Notify Developer (Slack)": {
      "main": [[{ "node": "Respond to Webhook", "type": "main", "index": 0 }]]
    },
    "Error Trigger": {
      "main": [[{ "node": "Alert Team on Error", "type": "main", "index": 0 }]]
    }
  },
  "settings": {
    "executionOrder": "v1",
    "saveExecutionProgress": true,
    "callerPolicy": "workflowsFromSameOwner",
    "errorWorkflow": ""
  },
  "staticData": null,
  "tags": ["bug-triage", "ai", "automation"]
}
```

---

## 6. Node-by-Node Configuration

### Node 1: Bug Report Intake (Webhook)

**Type**: `Webhook`

| Setting | Value |
|---------|-------|
| HTTP Method | `POST` |
| Path | `bug-report` |
| Response Mode | `Using Respond to Webhook Node` |
| Authentication | `None` (add IP allowlist or token in production) |

**Resulting URL**: `https://your-n8n-instance.com/webhook/bug-report`

**What it does**: Opens an HTTP endpoint that accepts bug reports from Sentry, Jira forms, CI pipelines, or custom scripts. All POST data lands in `$json.body`.

**Production hardening** (optional):
- Add a secret header: `X-Bug-Report-Token: YOUR_SECRET` and validate it in a follow-up IF node.
- Limit allowed IPs using a reverse proxy (nginx) in front of n8n.

---

### Node 2: Normalize Bug Data (Set)

**Type**: `Set` → Manual mode

**Purpose**: Creates a consistent schema regardless of whether the bug came from Sentry, a Jira form, an email parser, or a CI webhook. Each field uses `||` fallbacks to handle all formats.

| Output Field | Expression | Fallback |
|---|---|---|
| `source` | `$json.body.source` | `$json.body.project` → `'unknown'` |
| `error_message` | `$json.body.error` | `$json.body.title` → `$json.body.message` |
| `stack_trace` | `$json.body.stack_trace` | `$json.body.exception` |
| `user_description` | `$json.body.description` | `$json.body.body` |
| `environment` | `$json.body.environment` | `'production'` |
| `reporter` | `$json.body.reporter` | `$json.body.email` |
| `reported_at` | `$now.toISO()` | — |

**Key rule**: Always use `{{$json.body.fieldName}}` after a Webhook node — data is nested under `.body`.

---

### Node 3: AI Triage Analysis (HTTP Request → OpenAI)

**Type**: `HTTP Request`

| Setting | Value |
|---------|-------|
| Method | `POST` |
| URL | `https://api.openai.com/v1/chat/completions` |
| Authentication | Header Auth → `OpenAI API` credential |
| Content Type | `JSON` |
| Timeout | `30000ms` |

**Request Body** (key fields):

```json
{
  "model": "gpt-4o-mini",
  "max_tokens": 800,
  "temperature": 0,
  "messages": [
    { "role": "system", "content": "...triage prompt with severity + routing rules..." },
    { "role": "user",   "content": "...assembled from all normalized fields..." }
  ]
}
```

**Why `temperature: 0`**: Deterministic classification — you want consistent severity labels, not creative responses.

**Why `gpt-4o-mini`**: Cheap (~$0.15/million tokens input), fast (< 2s), excellent at structured JSON classification.

**Credential setup**:
- In n8n → Settings → Credentials → + Add → `Header Auth`
- Name: `OpenAI API`
- Header: `Authorization`, Value: `Bearer sk-proj-YOUR_KEY`

---

### Node 4: Parse AI Response & Map Developer (Code)

**Type**: `Code` (JavaScript, "Run Once for All Items")

**Purpose**: Takes the raw OpenAI API response and:
1. Extracts the JSON string from `choices[0].message.content`
2. Parses and validates the triage object
3. Looks up the developer's Slack ID and email from the internal routing table
4. Maps severity to Jira priority and emoji
5. Passes all fields to downstream nodes

**Developer Routing Table** — edit this section to match your team:

```javascript
const DEVELOPER_MAP = {
  'Alice Chen':   { slack_id: 'U01ALICE',  email: 'alice@company.com' },
  'Bob Martinez': { slack_id: 'U02BOB',    email: 'bob@company.com'   },
  // Add/remove developers here
};
```

**How to find a Slack User ID**:
- In Slack, click the developer's name → View Profile → ⋮ (More) → Copy member ID
- Format: `U01234ABCD`

**Error handling**: If the AI returns malformed JSON, the catch block defaults to severity `medium` and routes to the fallback developer (`Carol Singh`).

---

### Node 5: Severity Router (Switch)

**Type**: `Switch`

| Output | Condition | Jira Priority |
|--------|-----------|---------------|
| `Critical` | `severity == "critical"` | Highest |
| `High` | `severity == "high"` | High |
| `Medium` | `severity == "medium"` | Medium |
| Fallback (Low) | All other values | Low |

**Key setting**: `mode: "rules"` with case-insensitive string comparison.

**Both** medium and low severity bugs route to the same "Standard" Jira creation node (they just differ in Jira priority). The critical path routes to its own node so you can add additional escalation actions later (PagerDuty, on-call SMS, etc.).

---

### Node 6A: Create Jira Ticket — Critical

**Type**: `HTTP Request`

| Setting | Value |
|---------|-------|
| Method | `POST` |
| URL | `https://YOUR_DOMAIN.atlassian.net/rest/api/3/issue` |
| Auth | Header Auth → `Jira Basic Auth` |
| Content Type | `JSON` |

**Required changes before use**:
1. Replace `YOUR_DOMAIN` with your Atlassian subdomain (e.g. `mycompany`)
2. Replace `"key": "BUG"` with your actual Jira project key
3. Remove `customfield_10016` if you don't use story points

**Jira Ticket Body** creates a structured ticket with:
- Title: `[🔴 CRITICAL] {module}: {error message}`
- Description: Severity reason, module, root cause hypothesis, reproduction steps, stack trace (first 2000 chars), user description
- Priority: `Highest`
- Assignee: Developer email
- Labels: `ai-triaged`, `critical`, AI-suggested labels

---

### Node 6B: Create Jira Ticket — Standard

Identical to 6A but used for High / Medium / Low severity bugs. Uses the `severity_emoji` and `jira_priority` fields set by the Code node to automatically adjust the ticket appearance without needing separate nodes per level.

---

### Node 7: Notify Developer (Slack)

**Type**: `Slack`

| Setting | Value |
|---------|-------|
| Resource | `Message` |
| Operation | `Post` |
| Channel | `={{$json.dev_slack_id}}` (sends as DM to developer) |

**Message contains**:
- Severity emoji + level badge
- Error message preview
- Module and environment
- AI root cause hypothesis
- Reproduction steps
- Button linking directly to the Jira ticket

**Channel options**:
- Send as DM: use `={{$json.dev_slack_id}}` (User ID format `U01ABCD`)
- Send to channel instead: replace with `#backend-bugs` or any channel name
- Send both: duplicate the Slack node with two different targets

**Required Slack permissions**: `chat:write`, `chat:write.public`, `im:write`

---

### Node 8: Respond to Webhook

**Type**: `Respond to Webhook`

Returns a `200 OK` JSON confirmation to the caller (Sentry, CI script, etc.) within the 30-second timeout window. Without this node, the webhook will hang until timeout.

```json
{
  "status": "triaged",
  "severity": "high",
  "assigned_to": "Carol Singh",
  "module": "api",
  "jira_key": "CREATED"
}
```

---

### Node 9 & 10: Error Trigger + Alert Team

**Error Trigger**: Automatically fires if any node in the workflow throws an unhandled error.

**Alert Team on Error (Slack)**:
- Channel: `#engineering-alerts` (a shared channel, not a DM)
- Message includes: error message, workflow name, which node failed, timestamp
- Tells the team to triage manually while the automation is down

---

## 7. Developer Routing Map (Code Node Logic)

The Code node in Node 4 contains the full routing table. Edit the `DEVELOPER_MAP` constant to match your team. The AI model uses the routing rules embedded in the system prompt to pick a developer name, and Node 4 looks up that name to find their Slack ID and email.

**How routing works end-to-end**:

```
Bug arrives: "NullPointerException in PaymentController.processRefund()"
     │
     ▼ AI reads stack trace
AI detects: "payment" keyword in class name
     │
     ▼ AI applies routing rule from system prompt:
"payment/billing/stripe/invoice → Bob Martinez"
     │
     ▼ Code node looks up:
DEVELOPER_MAP['Bob Martinez'] → { slack_id: 'U02BOB', email: 'bob@company.com' }
     │
     ▼ Jira ticket created, assigned to bob@company.com
     ▼ Slack DM sent to U02BOB with full context
```

**To add a new developer or module**:
1. Open Node 4 (Code) → edit `DEVELOPER_MAP`
2. Open Node 3 (AI Triage Analysis) → edit the system prompt routing section
3. Both must be kept in sync

---

## 8. Testing the Workflow

### Step 1: Activate in Test Mode

In n8n, open the workflow → click **"Test workflow"** (not Activate). This runs without activating the live webhook.

### Step 2: Send a Test Bug Report

Use curl or Postman to send a POST to the webhook URL shown in Node 1:

```bash
curl -X POST https://YOUR_N8N_INSTANCE/webhook/bug-report \
  -H "Content-Type: application/json" \
  -d '{
    "source": "sentry",
    "error": "TypeError: Cannot read properties of null (reading '\''userId'\'')",
    "stack_trace": "TypeError: Cannot read properties of null\n    at AuthMiddleware.validateToken (auth/middleware.js:42:18)\n    at Layer.handle [as handle_request] (express/lib/router/layer.js:95:5)\n    at next (express/lib/router/route.js:137:13)",
    "description": "Users are being logged out randomly and cannot log back in. Happening to ~30% of users in EU region.",
    "environment": "production",
    "reporter": "qa-team@company.com"
  }'
```

**Expected result**:
- Severity: `high` or `critical`
- Module: `auth`
- Assigned developer: `Alice Chen`
- Jira ticket created
- Alice receives a Slack DM

### Step 3: Test Each Severity Level

```bash
# Test low severity
curl -X POST ... -d '{"error": "Button label has wrong font size", "description": "Minor UI issue on settings page", ...}'

# Test critical
curl -X POST ... -d '{"error": "FATAL: database connection pool exhausted", "stack_trace": "...", "environment": "production", ...}'
```

### Step 4: Verify in Each System

- **Jira**: Check your project board for new tickets
- **Slack**: Check the developer's DM and `#engineering-alerts`
- **n8n**: Execution log shows all nodes green

---

## 9. Customization Guide

### Add Sentry Native Integration

Instead of webhook, use the Sentry n8n node (community) or configure Sentry's Webhook under:
`Sentry Project → Settings → Webhooks → Add Webhook → URL: your-n8n-webhook`

Payload mapping for Sentry format in Node 2 (Set):
```
error_message  ← $json.body.data.error.title
stack_trace    ← $json.body.data.error.exception.values[0].stacktrace.frames
environment    ← $json.body.data.environment
```

### Add PagerDuty for Critical Bugs

After the "Create Jira Ticket (Critical)" node, add an HTTP Request to PagerDuty's Events API:

```
POST https://events.pagerduty.com/v2/enqueue
{
  "routing_key": "YOUR_PD_INTEGRATION_KEY",
  "event_action": "trigger",
  "payload": {
    "summary": "CRITICAL BUG: {{$json.error_message}}",
    "severity": "critical",
    "source": "n8n-bug-triage"
  }
}
```

### Add Email Fallback

If Slack DM fails, add a Gmail or SMTP node after Slack with `continueOnFail: true` on the Slack node. Use `$json.dev_email` as the recipient.

### Route to Teams Instead of Individuals

Change the system prompt routing to team channels instead of developer names:
```
auth → #team-identity
payment → #team-payments
backend → #team-platform
```
And in Node 4, map team names to Slack channel IDs.

### Store Triage History (Optional)

Add a Google Sheets or Postgres node after the Jira creation node to log every triage event:
```
Columns: timestamp, severity, module, developer, jira_key, ai_confidence, error_message
```

---

## Summary Checklist

Before going live, confirm:

- [ ] OpenAI API key created and set in n8n credentials
- [ ] Jira API token created, Base64-encoded, and set in n8n credentials
- [ ] Slack bot created with correct scopes, bot token set in n8n credentials
- [ ] Replaced `YOUR_DOMAIN` in Jira nodes with real Atlassian subdomain
- [ ] Replaced `BUG` with your actual Jira project key
- [ ] Updated `DEVELOPER_MAP` in Code node with real Slack User IDs and emails
- [ ] Updated system prompt routing rules to match your developer names exactly
- [ ] Invited Slack bot to all relevant channels/DMs
- [ ] Tested with sample curl requests for all severity levels
- [ ] Activated workflow (toggle in top-right of workflow editor)
- [ ] Configured Sentry/source system to POST to the webhook URL

---

*Generated for n8n — tested patterns based on n8n v1 execution model. Workflow uses connection-based execution order (v1).*
