MAIN README (GrowCRM)
# 🚀 GrowCRM – Agentic AI Workflow System (n8n)

GrowCRM is an event-driven, AI-powered workflow system built using n8n that simulates a real-world product lifecycle — from requirement intake to execution, evaluation, tracking, and communication.

This project demonstrates how AI, automation, and orchestration can work together to create a fully connected system that responds to inputs, processes decisions, and keeps stakeholders informed in real time.

---

## 💡 Problem

In most workflows, task tracking and communication are fragmented:

- Updates are manual
- Visibility is limited
- Completion tracking is inconsistent
- Teams rely on follow-ups instead of systems

---

## ⚙️ Solution

GrowCRM solves this by creating a unified automation pipeline that:

- Captures requirements
- Generates structured outputs using AI
- Evaluates and scores variants
- Creates and manages tasks in Jira
- Tracks every movement in real time
- Notifies stakeholders automatically

---

## 🧠 Architecture Overview

GrowCRM is built as a modular system:

- **M1 – Intake & Requirement Orchestrator**  
  Captures input and generates structured product requirements

- **M2 – Evaluation & Scoring Engine**  
  Compares AI-generated variants and selects the best outcome

- **M3 – Development Lifecycle Manager**  
  Creates and manages Jira tickets and transitions

- **M4 – Communication & Governance Layer**  
  Sends updates via Slack and Gmail and logs activity

- **Jira Listener (Standalone Module)**  
  Tracks real-time Jira status changes and triggers updates

---

## 🔄 Workflow Flow

Webhook → AI Processing → Evaluation → Jira →  
Google Sheets → Slack → Gmail

---

## 🧪 Example Output

- Jira ticket created and transitioned automatically  
- Each status movement logged in Google Sheets  
- Slack messages triggered for every transition  
- Gmail notification sent when task reaches "Done"  

---

## 🛠 Tech Stack

- n8n (Workflow Orchestration)
- Groq / LLM APIs
- Jira Cloud API
- Google Sheets API
- Slack API
- Gmail API

---

## ⚡ Setup Instructions

1. Import all workflow JSON files into n8n  
2. Configure credentials:
   - LLM API (Groq or OpenAI-compatible)
   - Jira API
   - Google Sheets
   - Slack
   - Gmail  
3. Replace placeholders:
   - YOUR_GOOGLE_SHEET_URL  
   - YOUR_SLACK_CHANNEL_ID  
   - YOUR_JIRA_DOMAIN  
   - YOUR_EMAIL_HERE  
4. Activate workflows  

---

## 🧱 Project Structure


GrowCRM/
│
├── workflows/
│ ├── M1_Intake.json
│ ├── M2_Evaluation.json
│ ├── M3_Lifecycle.json
│ ├── M4_Communication.json
│ ├── Master_Orchestrator.json
│
├── jira-listener/
│ ├── Jira_Listener.json
│ └── README.md
│
└── README.md


---

## 💎 Key Highlights

- Event-driven architecture using webhooks  
- AI-powered decision making  
- Modular workflow design  
- Real-time tracking and observability  
- Multi-channel communication system  

---

## 🚀 Note

Check out the whole demo video here: https://www.loom.com/share/17b00155a51742ea89a06eb52882d848

The Jira Listener module is designed as a standalone system and can be used independently for real-time task tracking and notifications.

---

## 💭 Final Thought

This project reflects a shift from using automation tools to designing intelligent systems that can observe, decide, and act.

GrowCRM is not just a workflow — it is a foundation for building Agentic AI systems.
