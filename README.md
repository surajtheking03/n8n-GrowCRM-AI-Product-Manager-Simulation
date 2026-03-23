# GrowCRM — AI Product Manager Simulation

### v1.0 — Built with n8n, Groq API, Jira, Google Sheets, Slack & Gmail

> A fully automated AI Product Manager simulation that runs an entire SaaS product lifecycle — from client intake to governance logging — without a single human in the loop.

---

## 🧠 What is GrowCRM?

GrowCRM is an event-driven automation system built inside **n8n** that simulates how a real Product Manager operates inside a software company.

When a client submits a requirement, GrowCRM takes over:

* Structures it into a product brief using an LLM
* Runs an A/B evaluation between two competing AI-generated product concepts
* Moves the project through a full development lifecycle in Jira
* Fires Slack notifications, Gmail updates, and governance logs automatically — based on lifecycle stage
* Logs every pipeline stage to a dedicated execution log for full observability

No human intervention. No manual updates. All AI-driven.

---

## 🏗 Architecture

```
Client Requirement (Webhook)
        ↓
M0 — Execution Logging Layer (observability)
        ↓
M1 — Intake & Requirement Orchestrator
        ↓ (AI-structured product brief → Jira + Sheets)
M2 — AI Evaluation & Scoring Engine
        ↓ (Two LLMs compete, third LLM judges winner)
M3 — Development Lifecycle & Feedback Engine
        ↓ (Jira stage transitions → Slack notifications)
M4 — Communication & Governance Layer
        ↓ (Slack + Gmail + Google Sheets governance log)
```

---

## 📦 Tech Stack

### 🖥 v1.0 Local Build (Origin)
> This is where GrowCRM was born. The entire system was built, debugged, and validated locally using Ollama — no cloud costs, no API keys, just a laptop and ambition.

| Tool | Role |
|---|---|
| n8n (local) | Orchestration engine |
| Ollama — Gemma 3:4b | AI brain — Variant A + Evaluator |
| Ollama — Llama 3.2 | AI brain — Variant B |
| Jira Software Cloud | Project & lifecycle tracking |
| Google Sheets | Logging layer (projects, transitions, governance, execution) |
| Slack | Internal team notifications |
| Gmail | Client-facing communication |

### ☁️ v1.0 Cloud (Current — Groq Migration)
> After full local validation, Ollama was replaced with Groq API for cloud deployment. Same architecture, same logic — just cloud-ready LLMs.

| Tool | Role |
|---|---|
| n8n (Railway) | Orchestration engine — cloud hosted |
| Groq — llama-3.1-8b-instant | AI brain — Variant A (fast, efficient) |
| Groq — llama-3.3-70b-versatile | AI brain — Variant B + Evaluator (deeper reasoning) |
| Jira Software Cloud | Project & lifecycle tracking |
| Google Sheets | Logging layer |
| Slack | Internal team notifications |
| Gmail | Client-facing communication |

---

## 📋 Modules

### M0 — Execution Logging Layer *(New in v1.0 Cloud)*

**Google Sheet tab:** `execution_log`

The observability backbone of GrowCRM. Every pipeline run writes 6 rows to the execution log — one per stage. If a run fails, you can instantly see exactly which stage it died at.

**Columns:** `execution_id | project_id | stage | status | timestamp | error`

**Stages logged:**
```
pipeline_started → m1_completed → m2_completed → m3_completed → m4_completed → pipeline_finished
```

**Key design decisions:**
* `execution_id` ties all 6 rows of a single run together
* `project_id` shows `pending` for the first row (M1 hasn't run yet) then the real UUID from row 2 onwards
* Timestamps incrementing across rows reveal exactly how long each module takes
* Missing rows = exact failure point — no guessing needed

---

### M1 — Intake & Requirement Orchestrator

**Webhook path:** `GrowCrm load`

Accepts a raw client requirement via POST and converts it into a structured AI product brief.

**Flow:**
```
Webhook → If1 (AND validation)
→ Code Node (builds LLM prompt, bakes client_name)
→ HTTP Request (Groq llama-3.1-8b-instant)
→ JSON Parsing Node (strips markdown fences, fallback on parse failure)
→ Generate Project ID (UUID for project + transition)
→ Sheet → projects (parallel)
→ Sheet → state_transitions (parallel)
→ Create Jira Issue (parallel)
→ Move to Requirement Structured
→ Slack notification
```

**Input payload:**
```json
{
  "client_name": "Acme Corporation",
  "category": "CRM",
  "raw_requirement_text": "We need a CRM to manage leads and automate follow-ups",
  "budget_range": "₹5L - ₹10L",
  "deadline_preference": "3 months"
}
```

**Google Sheets tabs written:**
* `projects` — project_id, client_name, category, description, features, sub_features, current_state, created_at
* `state_transitions` — transition_id, project_id, from_state, to_state, validated, timestamp

**Key design decisions:**
* `client_name` is baked directly into the Groq prompt so it survives the HTTP node data wipe
* JSON Parsing Node has a hardcoded fallback in case LLM response is malformed
* Both Sheet nodes wired from `Generate Project ID`, never from the Jira node
* If1 uses AND logic — both `client_name` and `raw_requirement_text` must be non-empty

---

### M2 — AI Evaluation & Scoring Engine

**Webhook path:** `evaluation scoring engine`

Runs two LLMs in parallel to generate competing product concepts, then uses a third LLM to evaluate and pick a winner.

**Flow:**
```
Webhook → If1 (AND validation)
→ Code Node (builds structured prompt)
→ LLM llama-3.1-8b-instant (Variant A) — parallel
→ LLM llama-3.3-70b-versatile (Variant B) — parallel
→ Parse A + Parse B
→ Merge Node
→ Evaluation LLM (llama-3.3-70b-versatile judges both variants)
→ Parse Evaluation
→ Score Decision (IF node — recommended_variant)
```

**Input payload:**
```json
{
  "client_name": "Acme Corporation",
  "category": "CRM",
  "raw_requirement_text": "We need a CRM to manage leads and automate follow-ups"
}
```

**Evaluation output:**
```json
{
  "variant_A_percent": 65,
  "variant_B_percent": 35,
  "confidence_score": 82,
  "recommended_variant": "A",
  "reason": "Variant A better addresses the lead automation requirement"
}
```

**Key design decisions:**
* Two different models (instant vs versatile) run in parallel — genuine diversity in A/B outputs
* A third LLM instance acts as an impartial evaluator — simulating a governance analyst
* Merge node combines both outputs before passing to the evaluator
* One-line JSON body format used for Evaluation LLM to avoid n8n expression parsing issues

---

### M3 — Development Lifecycle & Feedback Engine

**Webhook path:** `development lifecycle`

Moves a Jira ticket through lifecycle stages and fires a Slack notification on every transition.

**Flow:**
```
Webhook → If1 (AND validation)
→ Code Node (maps status string → Jira transition ID integer)
→ Jira HTTP POST (transitions the issue)
→ Slack notification
```

**Input payload:**
```json
{
  "jira_key": "GCR-24",
  "feedback_status": "testing",
  "feedback_notes": "QA cycle started",
  "client_name": "Acme Corporation"
}
```

**Transition map:**
```
selected_for_development → 21
in_development           → 51
testing                  → 61
client_review            → 71
delivered                → 81
done                     → 41
```

**Key design decisions:**
* Transition IDs are integers — not strings. Jira rejects string IDs silently
* Slack message pins back to `$node["Code in JavaScript"].json` to survive the Jira HTTP node data wipe
* Transition map is hardcoded — IDs are board-specific and must be discovered via `GET /transitions` for your board

---

### M4 — Communication & Governance Layer

**Webhook path:** `governance-event`

Routes lifecycle events to the right communication channel — Slack for internal, Gmail for client-facing, and Google Sheets for governance logging.

**Flow:**
```
Webhook → If (AND validation)
→ Code Node (extracts + normalises all fields, adds timestamp)
→ Append row in sheet (governance_log)
→ Switch Node (routes by status)
  → selected_for_development → Return Output (silent pass-through)
  → testing       → Slack
  → client_review → Slack → Gmail
  → delivered     → Slack → Gmail
  → done          → Slack → Gmail
→ Return Output → Respond to Webhook
```

**Key design decisions:**
* `Return Output` Set node added before `Respond to Webhook` — ensures M4 always returns data to parent Orchestrator regardless of which branch fires
* `selected_for_development` routes directly to Return Output — internal stage, no client communication needed
* Switch node used instead of nested IF branches — cleaner for multi-route logic
* All Gmail and Sheets nodes pin back to `$node["Code in JavaScript"].json.xxx` — Slack node output wipes `$json`

---

### Master Orchestrator

**Webhook path:** `growcrm-orchestrator`

The brain that chains all modules together in sequence and manages data flow between them.

**Flow:**
```
Webhook
→ Log: pipeline_started
→ Set: Restore Webhook Data
→ Call M1
→ Log: m1_completed
→ Call M2
→ Log: m2_completed
→ Edit Fields (prepare M3 payload)
→ Call M3
→ Log: m3_completed
→ Set: Restore M4 Payload
→ Call M4
→ Log: m4_completed
→ Log: pipeline_finished
→ Respond to Webhook
```

**Key design decisions:**
* `Set: Restore Webhook Data` — re-maps original webhook fields before Call M1 because Log nodes overwrite `$json`
* `Set: Restore M4 Payload` — re-maps Edit Fields data before Call M4 for the same reason
* All log nodes are in-chain (not branches) — they must be traversed for the pipeline to continue
* `$execution.id` ties all 6 execution log rows together per run

---

## 🗂 Jira Board

```
Client Intake → Requirement Structured → Planning →
In Development → Testing → Client Review → Delivered → Done
```

---

## 🔑 Hard-Won Lessons

1. **HTTP nodes wipe `$json`** — always bake critical fields into the LLM prompt or use Set nodes to restore data
2. **Jira transition IDs must be integers** — strings are silently rejected
3. **Slack/Gmail nodes also wipe `$json`** — pin downstream nodes back to `$node["Code in JavaScript"].json.xxx`
4. **Sheet nodes must be wired from the data source node** — never from Jira or Slack output nodes
5. **Switch node beats nested IFs** — cleaner, more readable for multi-route logic
6. **Intermediate log nodes overwrite context** — use Set restore nodes before every sub-workflow call
7. **Groq uses `choices[0].message.content`** — not `response` like Ollama
8. **One-line JSON body** — multiline expressions in n8n HTTP nodes break JSON parsing
9. **Sub-workflow return contract** — `Respond to Webhook` sends HTTP response but doesn't return data to parent; use a Set node before it
10. **Ollama OAuth token expires** — reconnect via Settings → Credentials (local build lesson)
11. **gemma2-9b-it is deprecated on Groq** — use `llama-3.1-8b-instant` as replacement
12. **Expression vs Fixed mode** — `={{ }}` syntax only works in Expression mode in n8n JSON fields

---

## 🚀 Getting Started

### Prerequisites

* n8n running locally or on Railway
* Groq API key — get one free at `console.groq.com`
* Jira Software Cloud account with API credentials
* Google Sheets OAuth2 credentials
* Slack app with bot token
* Gmail OAuth2 credentials

### Google Sheets Setup

Create a Google Sheet with these 4 tabs:
```
projects | state_transitions | governance_log | execution_log
```

**execution_log columns:**
`execution_id | project_id | stage | status | timestamp | error`

### Setup Steps

1. Clone this repository
2. Import each workflow JSON into n8n (`Settings → Import Workflow`)
3. Re-connect credentials for Jira, Google Sheets, Slack, Gmail, Groq
4. Activate all 5 workflows (M0 is part of Master Orchestrator — not a separate workflow)
5. Test using the payload below

### Test Payload

**Endpoint:** `POST http://localhost:5678/webhook-test/growcrm-orchestrator`

```json
{
  "client_name": "YourCompany Inc",
  "category": "CRM",
  "raw_requirement_text": "We need a CRM to manage customers and order history",
  "budget_range": "10k",
  "deadline_preference": "3 months"
}
```

### Module Execution Order (standalone testing)

```
M1 → Submit new client intake
M2 → Trigger AI evaluation on the intake
M3 → Move ticket through lifecycle stages
M4 → Fire governance events at each stage
Master Orchestrator → Runs full pipeline end to end
```

---

## 📁 Repository Structure

```
growcrm-v1.0/
├── GrowCRM – Master Orchestrator.json
├── GrowCRM – M1_ Intake & Requirement Orchestrator.json
├── GrowCRM – M2_ Evaluation & Scoring Engine.json
├── GrowCRM – M3_ Development Lifecycle & Feedback Engine.json
├── GrowCRM – M4_ Communication & Governance Layer.json
└── README.md
```

---

## 🔮 Roadmap

### V1.0 (Current)
* ✅ M0 — Execution Logging Layer
* ✅ M1 — Client Intake & Requirement Orchestrator
* ✅ M2 — AI Evaluation & Scoring Engine
* ✅ M3 — Development Lifecycle & Feedback Engine
* ✅ M4 — Communication & Governance Layer
* ✅ Groq API migration (cloud-ready)
* ⏳ Railway deployment
* ⏳ Jira webhook (event-driven trigger)
* ⏳ GitHub v1.0 release tag

### V2.0 (Planned — after V1.0 deployment complete)

> ⚠️ V2.0 work begins only after full V1.0 deployment is done.

* Replace Google Sheets with **Postgres**
* Replace Jira with **Trello** (tentative)
* Full cloud-native deployment
* Persistent schema: projects, transitions, ab_results, feedback_logs, execution_logs

---

## 👥 Team

* **Suraj** — Builder, shipped all modules, Groq migration, cloud deployment
* **Jaanu** — Architecture validation, governance design, observability layer design
* **Claude** — Debugging partner, arena memory, lifecycle engine 😄

---

## 🏆 Origin Story

GrowCRM started as a local experiment — no cloud, no API costs, just n8n running on a laptop with Ollama powering the AI. Every module was built, broken, debugged, and rebuilt locally.

The switch to Groq wasn't about Ollama being inadequate — it was about making the system deployable. Ollama carried the entire v1.0 build. Groq carries the deployment.

---

*GrowCRM V1.0 — AI Product Manager Simulation*
*Built with n8n · Groq API · Jira · Google Sheets · Slack · Gmail*
