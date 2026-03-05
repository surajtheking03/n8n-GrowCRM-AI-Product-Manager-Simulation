# GrowCRM — AI Product Manager Simulation
### v1.0 — Built with n8n, Ollama, Jira, Google Sheets, Slack & Gmail

> A fully automated AI Product Manager simulation that runs an entire SaaS product lifecycle — from client intake to governance logging — without a single human in the loop.

---

## 🧠 What is GrowCRM?

GrowCRM is an event-driven automation system built inside **n8n** that simulates how a real Product Manager operates inside a software company.

When a client submits a requirement, GrowCRM takes over:
- Structures it into a product brief using a local LLM
- Runs an A/B evaluation between two competing AI-generated product concepts
- Moves the project through a full development lifecycle in Jira
- Fires Slack notifications, Gmail updates, and governance logs automatically — based on lifecycle stage

No human intervention. No manual updates. All AI-driven.

---

## 🏗 Architecture

```
Client Requirement (Webhook)
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

| Tool | Role |
|------|------|
| n8n (local) | Orchestration engine |
| Ollama — Gemma 3:4b | AI brain — Variant A + Evaluator |
| Ollama — Llama 3.2 | AI brain — Variant B |
| Jira Software Cloud | Project & lifecycle tracking |
| Google Sheets | Logging layer (projects, transitions, governance) |
| Slack | Internal team notifications |
| Gmail | Client-facing communication |

---

## 📋 Modules

### M1 — Intake & Requirement Orchestrator
**Webhook path:** `GrowCrm load`

Accepts a raw client requirement via POST and converts it into a structured AI product brief.

**Flow:**
```
Webhook → If1 (AND validation)
→ Code Node (builds LLM prompt, bakes client_name)
→ HTTP Request (Ollama Gemma 3:4b)
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
- `projects` — project_id, client_name, category, description, features, sub_features, current_state, created_at
- `state_transitions` — transition_id, project_id, from_state, to_state, validated, timestamp

**Key design decisions:**
- `client_name` is baked directly into the Ollama prompt so it survives the HTTP node data wipe
- JSON Parsing Node has a hardcoded fallback in case LLM response is malformed
- Both Sheet nodes wired from `Generate Project ID`, never from the Jira node
- If1 uses AND logic — both `client_name` and `raw_requirement_text` must be non-empty

---

### M2 — AI Evaluation & Scoring Engine
**Webhook path:** `evaluation scoring engine`

Runs two LLMs in parallel to generate competing product concepts, then uses a third LLM to evaluate and pick a winner.

**Flow:**
```
Webhook → If1 (AND validation)
→ Code Node (builds structured prompt)
→ LLM Gemma 3:4b (Variant A) — parallel
→ LLM Llama 3.2 (Variant B)  — parallel
→ Parse A + Parse B
→ Merge Node
→ Evaluation LLM (Gemma judges both variants)
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
- Two LLMs run in parallel branches, each generating a distinct product concept
- A third LLM instance acts as an impartial evaluator — simulating a governance analyst
- Merge node combines both outputs before passing to the evaluator
- Score Decision IF node should reference `={{$json.recommended_variant}}` (not hardcoded string)

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
  "feedback_notes": "QA cycle started"
}
```

**Transition map (hardcoded integers — discovered via Jira API):**
```
selected_for_development → 21
in_development           → 51
testing                  → 61
client_review            → 71
delivered                → 81
done                     → 41
```

**Jira API endpoint:**
```
POST https://<your-domain>.atlassian.net/rest/api/3/issue/{jira_key}/transitions
Body: { "transition": { "id": 61 } }
```

**Key design decisions:**
- Transition IDs are integers — not strings. Jira rejects string IDs silently
- Slack message references `$node["Code in JavaScript"].json` directly to survive the Jira HTTP node data wipe
- Transition map is hardcoded — IDs are board-specific and must be discovered via `GET /transitions` for your board

---

### M4 — Communication & Governance Layer
**Webhook path:** `governance-event`

Routes lifecycle events to the right communication channel — Slack for internal, Gmail for client-facing, and Google Sheets for governance logging.

**Flow:**
```
Webhook → If1 (AND validation)
→ Code Node (extracts + normalises all fields, adds timestamp)
→ Switch Node (routes by status)
  → Output 0: testing       → Slack
  → Output 1: client_review → Slack → Gmail
  → Output 2: delivered     → Slack → Gmail
  → Output 3: done          → Slack → Gmail → Google Sheets (governance_log)
```

**Input payloads:**
```json
// Testing
{ "jira_key": "GCR-24", "status": "testing", "notes": "QA started!", "client_name": "Acme Corporation", "project_name": "CRM Project" }

// Client Review
{ "jira_key": "GCR-24", "status": "client_review", "notes": "Demo ready!", "client_name": "Acme Corporation", "project_name": "CRM Project" }

// Delivered
{ "jira_key": "GCR-24", "status": "delivered", "notes": "Product delivered!", "client_name": "Acme Corporation", "project_name": "CRM Project" }

// Done
{ "jira_key": "GCR-24", "status": "done", "notes": "Project signed off!", "client_name": "Acme Corporation", "project_name": "CRM Project" }
```

**Google Sheets — governance_log tab:**
```
jira_key | status | client_name | project_name | notes | timestamp
```

**Key design decisions:**
- Switch node used instead of nested IF branches — cleaner for 4-route logic
- All Gmail and Sheets nodes reference `$node["Code in JavaScript"].json.xxx` — Slack node output wipes `$json`, so you must pin back to the Code node
- Testing branch is Slack-only — internal QA stage, no client communication needed
- Governance log only fires on `done` — the final signed-off stage

---

## 🗂 Jira Board

```
Client Intake → Requirement Structured → Planning →
In Development → Testing → Client Review → Delivered → Done
```

Active tickets: GCR-24 to GCR-30

---

## 🔑 Hard-Won Lessons

1. **HTTP nodes wipe `$json`** — always bake critical fields into the LLM prompt so they come back in the response
2. **Jira transition IDs must be integers** — strings are silently rejected
3. **Slack/Gmail nodes also wipe `$json`** — pin downstream nodes back to `$node["Code in JavaScript"].json.xxx`
4. **Sheet nodes must be wired from the data source node** — never from Jira or Slack output nodes
5. **Switch node beats nested IFs** — cleaner, more readable for multi-route logic
6. **Event-driven architecture = safe resets** — no shared state between modules, each can be tested independently

---

## 🚀 Getting Started

### Prerequisites
- n8n running locally or on cloud
- Ollama installed with `gemma3:4b` and `llama3.2` models pulled
- Jira Software Cloud account with API credentials
- Google Sheets OAuth2 credentials
- Slack app with bot token
- Gmail OAuth2 credentials

### Setup
1. Clone this repository
2. Import each module JSON into n8n (`Settings → Import Workflow`)
3. Re-connect credentials for Jira, Google Sheets, Slack, Gmail
4. Activate all 4 workflows
5. Test each module using the payloads above (Hoppscotch or Postman)

### Module Execution Order
```
M1 → Submit new client intake
M2 → Trigger AI evaluation on the intake
M3 → Move ticket through lifecycle stages
M4 → Fire governance events at each stage
```

---

## 📁 Repository Structure

```
growcrm-v1.0/
├── M1_Intake_Requirement_Orchestrator.json
├── M2_Evaluation_Scoring_Engine.json
├── M3_Development_Lifecycle_Feedback_Engine.json
├── M4_Communication_Governance_Layer.json
└── README.md
```

---

## 🔮 Roadmap

### V1.0 (Current)
- [x] M1 — Client Intake & Requirement Orchestrator
- [x] M2 — AI Evaluation & Scoring Engine
- [x] M3 — Development Lifecycle & Feedback Engine
- [x] M4 — Communication & Governance Layer
- [ ] Full combined end-to-end run
- [ ] Deploy on Railway / Render
- [ ] GitHub release tag

### V2.0 (Planned — after V1.0 is fully complete)
> ⚠️ V2.0 work begins only after the full V1.0 process is done — combined run, deployment, and GitHub release all complete.

- Replace Google Sheets with **Postgres**
- Replace Jira with **Trello** (tentative)
- Full cloud-native deployment
- Persistent schema: projects, transitions, ab_results, feedback_logs

---

## 👥 Team

- **Suraj** — Builder, architect, shipped all 4 modules
- **Jaanu** — Architecture validation, governance design, guardrails
- **Claude** — Debugging partner, lifecycle engine, arena memory 😄

---

*GrowCRM V1.0 — AI Product Manager Simulation*
*Built with n8n · Ollama · Jira · Google Sheets · Slack · Gmail*
