# Medical Consultation POC
### AWS Bedrock · LangChain · LangGraph · Human-in-the-Loop

A prototype illustrating a person-in-the-loop agentic architecture for remote medical consultation. An AI agent handles patient intake and prescription recording; a doctor steps in for diagnosis. Built to demonstrate how LangChain, LangGraph, and AWS Bedrock compose into a stateful, interruptible agentic workflow — at a cost viable for charity hospitals.

---

## Documentation

| Document | What it covers |
|----------|---------------|
| [`demo.md`](demo.md) | Screenshot walkthrough of a complete consultation |
| [`implementation_plan.md`](implementation_plan.md) | Workflow design, state schema, tools, cost model |
| [`architecture.md`](architecture.md) | C4 diagrams, LangGraph flowchart, ADRs, key tradeoffs |
| [`roadmap.md`](roadmap.md) | POC-to-production phases, MCP migration, cost projection |
| [`CONTRIBUTING.md`](CONTRIBUTING.md) | Setup, code conventions, PR guidelines |

---

## How It Works

```
Patient → AI Intake Agent → [Emergency?] → Emergency alert (999/911)
                          ↓ (normal)
                     Doctor Queue  ← triage-sorted by severity score
                          ↓ interrupt()
                     Doctor Reviews
                          ↓ (clarification needed?)
                     Patient answers doctor's question → back to doctor
                          ↓ (diagnosis submitted)
                     AI Prescription Agent → checks pharmacy stock → Patient
```

1. Patient describes symptoms; the intake agent fetches their medical history and assigns a **triage score (1–5)**
2. If symptoms indicate an emergency, the patient sees an immediate red-banner alert — no queue
3. Otherwise, the case joins the **doctor's prioritised queue**, sorted by severity then wait time
4. The graph pauses (`interrupt()`); the doctor reviews the AI-generated intake summary
5. The doctor can submit a diagnosis **or** send a clarifying question back to the patient
6. The prescription agent checks **pharmacy inventory** before finalising — suggests alternatives if out of stock
7. The patient sees the prescription in their chat window

---

## Setup

### 1. Create a virtual environment and install dependencies

```bash
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

### 2. Configure AWS credentials

```bash
aws configure
```

Or export environment variables:

```bash
export AWS_ACCESS_KEY_ID=...
export AWS_SECRET_ACCESS_KEY=...
export AWS_DEFAULT_REGION=us-east-1
```

### 3. Enable Bedrock model access

In the [AWS Bedrock console](https://console.aws.amazon.com/bedrock/) → **Model access** → request access to:
- `us.anthropic.claude-haiku-4-5-20251001-v1:0` (cross-region inference profile; covers `us-east-1`)

### 4. Seed the database

```bash
python seed_db.py
```

Creates `patients.db` with dummy patients, medical history, knowledge base, and pharmacy inventory (Amoxicillin seeded as out-of-stock to demo the substitution flow).

---

## Run

```bash
streamlit run app.py
```

Open `http://localhost:8501` in your browser.

**Screens:**
- **Login** — select role (Patient / Doctor) and enter a dummy PIN
- **Patient view** — chat with the intake agent, see status updates, receive prescription
- **Doctor desktop** — triage-sorted alert queue (🔴🟡🟢), intake summaries with triage reasoning, diagnosis or clarification actions

---

## File Structure

```
├── README.md               ← this file
├── demo.md                 ← screenshot walkthrough of a complete consultation
├── implementation_plan.md  ← workflow design, state, tools, cost model
├── architecture.md         ← C4 diagrams, LangGraph flowchart, ADRs, tradeoffs
├── roadmap.md              ← POC-to-production phases, MCP migration, cost projection
├── CONTRIBUTING.md         ← how to contribute, code conventions, PR guidelines
├── CLAUDE.md               ← guidance for Claude Code
├── requirements.txt
├── seed_db.py              ← creates and seeds patients.db
├── tools.py                ← LangChain @tool wrappers (patient record, history, KB, pharmacy check, Rx)
├── graph.py                ← LangGraph workflow (nodes, edges, interrupt, checkpointing)
├── app.py                  ← Streamlit frontend (login, patient chat, doctor desktop)
```

---

## Tool Access and MCP

In this POC, tools are LangChain `@tool` functions that query a local SQLite database. This is the fastest path to a working prototype.

In production, each tool would be replaced by a call to an **MCP (Model Context Protocol) server** — a standardised interface that lets agents securely access enterprise systems (patient databases, EHRs, pharmacy systems) without bespoke API integrations per model. AWS Verified Permissions (Cedar policies) would enforce that agents only access data the requesting user is authorised to see.

The LangGraph graph nodes would not change; only the tool implementations swap.

---

## Cost

| Item | Cost |
|------|------|
| Claude Haiku 4.5 (~2,500 tokens/consultation) | ~$0.001 |
| SQLite (POC) | $0 |
| Streamlit (local) | $0 |
| **Per consultation** | **< $0.01** |

50 consultations/day → **< $20/month** total in production (including DynamoDB + Fargate).

---

## Troubleshooting

**`ValidationException: Invocation of model ID … with on-demand throughput isn't supported`**

Claude Haiku 4.5 (and newer models) require a cross-region inference profile, not a direct model ID. The default in `graph.py` is already set to `us.anthropic.claude-haiku-4-5-20251001-v1:0` (note the `us.` prefix). If you've overridden `BEDROCK_MODEL_ID` with a bare model ID, switch it to the `us.` prefixed version.

---

**`ResourceNotFoundException: … marked as Legacy`**

Your account has not used the model recently, or model access was never granted. Go to the [AWS Bedrock console](https://console.aws.amazon.com/bedrock/) → **Model access** and request access to the cross-region inference profile. To list currently active models in your account:

```bash
aws bedrock list-foundation-models --by-provider anthropic --region us-east-1 \
  --query 'modelSummaries[?modelLifecycle.status==`ACTIVE`].modelId'
```

---

**`TypeError: unsupported operand type(s) for |: 'type' and 'NoneType'`**

Python 3.9 does not support `X | None` union syntax at runtime. Both `app.py` and `graph.py` include `from __future__ import annotations` at the top to handle this. If you add new type hints, keep that import in place.

---

**Graph state not updating between patient and doctor tabs**

Both tabs must be open in the **same browser session** on the same Streamlit server. The `MemorySaver` checkpointer lives in-process and is shared via `@st.cache_resource`. Opening a second browser window in a private/incognito tab will start a separate server process with a separate memory store.

---

**`patients.db` not found**

Run `python seed_db.py` before starting the app. The database file is gitignored and must be created locally.

---

## AWS Credentials Note

This project uses AWS Bedrock (pay-per-token). No provisioned throughput is required — there are no idle costs. The stack can be stood up and torn down without any sunk infrastructure cost.

A `.env` file is gitignored. Never commit credentials.
