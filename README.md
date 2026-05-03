# AI Ops Request Triage Bot

An end-to-end AI automation pipeline that automatically triages internal operations requests by category, effort, priority, and routing — eliminating manual triage work for AI Ops teams.

Built with **n8n**, **Claude API (Anthropic)**, **Zapier**, and **Google Sheets**.

---

## The Problem

When employees across GTM, Engineering, and G&A want to request an automation or AI tool, there is no structured way to submit or prioritize those requests. Someone has to manually read each one, determine what type it is, how complex it would be to build, and who should own it. This creates a bottleneck and inconsistent prioritization.

## The Solution

Employees submit requests through a structured Typeform intake. Claude automatically classifies each request and routes it to the right team — no human triage required.

---

## Architecture

```
Typeform → n8n Webhook → Set Node → Claude API → Code Node → Switch Node → Zapier → Google Sheets
```

---

## Pipeline Walkthrough

### 1. Typeform Intake
Employees submit a request through a 6-question form capturing:
- Problem statement
- Team (GTM / EPD / G&A)
- Frequency of the problem
- Pain level
- Success criteria
- Requester name

### 2. n8n Webhook Trigger
The moment the form is submitted, Typeform fires a POST request to the n8n webhook endpoint. n8n receives the full form response payload.

### 3. Set Node — Field Extraction
The raw Typeform payload is deeply nested. This node extracts the 6 relevant fields and renames them cleanly for downstream use.

### 4. Claude API — AI Classification
The cleaned request is sent to **Claude Haiku** via the Anthropic API with a structured prompt. Claude returns a JSON object with 6 fields:

| Field | Values |
|---|---|
| `category` | AUTOMATION, PROMPT_ENGINEERING, DATA_PIPELINE, BUG_FIX, INTEGRATION, OTHER |
| `effort` | SMALL (under 2hrs), MEDIUM (2-8hrs), LARGE (8+ hrs) |
| `priority` | HIGH, MEDIUM, LOW |
| `suggested_owner` | AI_OPS, ENGINEERING, SELF_SERVE |
| `one_line_summary` | Plain English summary, max 15 words |
| `reasoning` | 1 sentence explaining the priority call |

### 5. Code Node — JSON Parsing
Claude occasionally wraps its response in markdown code fences despite prompt instructions. This node strips backticks and parses the raw text into a clean JSON object.

```javascript
const rawText = $input.first().json.content[0].text;
const cleaned = rawText.replace(/```json\n/g, '').replace(/```/g, '').trim();
const parsed = JSON.parse(cleaned);
```

### 6. Switch Node — Conditional Routing
Based on Claude's `suggested_owner` field, the request is routed down one of 3 paths:

| Path | Owner | Meaning |
|---|---|---|
| Fast Track | AI_OPS | Simple automation, buildable with no-code tools within 1-2 days |
| Needs Review | ENGINEERING | Complex build requiring a developer |
| Self Serve | SELF_SERVE | Solution already exists — user just needs to be pointed to it |

### 7. Zapier Webhook — Dispatch
Each routing path sends the enriched payload to a Zapier webhook, tagged with its route identifier (`fast_track`, `needs_review`, `self_serve`).

### 8. Google Sheets — Audit Trail
Zapier appends a new row to the **AI Ops Request Tracker** spreadsheet with all 11 fields, creating a persistent, searchable log of every request and how it was classified.

---

## Tech Stack

| Tool | Role |
|---|---|
| Typeform | Employee-facing intake form |
| n8n | Workflow orchestration and logic |
| Claude Haiku (Anthropic) | AI classification and triage |
| Zapier | Delivery layer and app integration |
| Google Sheets | Persistent audit trail |

---

## Prompt Design

Claude is prompted to act as an AI Ops analyst and return structured JSON. Key design decisions:

- **Temperature:** Low (0.2) for classification consistency over creativity
- **Model:** Claude Haiku for cost efficiency (~$0.0001 per request)
- **Output format:** Strict JSON with enumerated values to prevent hallucination on categories
- **Fallback:** Switch node defaults to AI_OPS output if `suggested_owner` doesn't match any condition

---

## Files

| File | Description |
|---|---|
| `Internal_Ops_Triage.json` | Full n8n workflow — import directly into n8n |
| `zapfile.json` | Zapier Zap configuration |

---

## Setup Instructions

### 1. n8n Workflow
1. Import `Internal_Ops_Triage.json` into your n8n instance
2. Add your Anthropic API key under **Credentials → Anthropic**
3. Update the Zapier webhook URL in the Fast Track, Needs Review, and Self Serve nodes
4. Publish the workflow — it will switch from test to production webhook URL automatically

### 2. Typeform
1. Create a form with 6 questions matching the field order in the Set Node
2. Go to **Connect → Webhooks** and point it to your n8n production webhook URL:
```
https://your-n8n-instance.app.n8n.cloud/webhook/ai-ops-triage
```

### 3. Zapier
1. Import `zapfile.json` into Zapier or recreate the Zap manually
2. Trigger: **Webhooks by Zapier → Catch Hook**
3. Action: **Google Sheets → Create Spreadsheet Row**
4. Map all 11 fields from the webhook payload to your sheet columns
5. Publish the Zap

### 4. Google Sheets
Create a spreadsheet called **AI Ops Request Tracker** with these headers in row 1:

```
Title | Category | Effort | Priority | Owner | Team | Requester | Route | Reasoning | Original Problem | Date
```

---

## Example Output

**Input (Typeform submission):**
> "Every week I manually copy win/loss data from Salesforce into a Google Sheet. Takes 30 minutes."

**Claude Classification:**
```json
{
  "category": "AUTOMATION",
  "effort": "SMALL",
  "priority": "HIGH",
  "suggested_owner": "AI_OPS",
  "one_line_summary": "Automate weekly Salesforce to Google Sheets data sync",
  "reasoning": "Recurring manual task with clear success criteria and low technical complexity makes this a high-priority fast track candidate."
}
```

**Route:** Fast Track → logged to Google Sheets immediately

---

## Key Design Decisions

**Why Claude Haiku over GPT-4?**
Cost efficiency. At ~$0.0001 per classification, this pipeline can triage thousands of requests per month for under $1. GPT-4 would cost 100x more for the same task.

**Why a JavaScript sanitization layer?**
Claude occasionally wraps JSON responses in markdown code fences even when instructed not to. Rather than relying on prompt engineering alone, the Code node defensively strips formatting before parsing — making the pipeline production-reliable regardless of model behavior.

**Why Zapier for delivery instead of n8n native Google Sheets?**
In a production environment, Zapier's delivery layer allows non-technical ops teammates to manage and modify the logging destination without touching the n8n workflow. Separation of concerns between logic (n8n) and delivery (Zapier) makes the system easier to maintain.
