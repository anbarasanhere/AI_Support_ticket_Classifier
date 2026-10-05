# Support Ticket Classifier

A map of the project: what it does, how the LangGraph pipeline works, and the best order to understand this codebase.

---

## 1. Project Overview

This is a **production-aware AI support ticket classifier**. You send raw customer ticket text; it returns structured routing data — category, team, priority, sentiment, and confidence — via GPT-4o-mini, wrapped in reliability layers:

- PII redaction
- Prompt-injection guard
- Schema / business-rule validation
- Retries and safe fallback
- Versioned prompts
- Cost tracking

**Example input:**
```
"I was charged twice for order #9981. Please refund immediately!"
```

**Example output:**
```json
{
  "issue_category": "payment_issue",
  "assigned_team": "payments_team",
  "priority": "high",
  "user_sentiment": "angry",
  "confidence_score": 0.97,
  "reasoning": "Customer explicitly reports duplicate charge and requests refund",
  "requires_human_review": false
}
```

**Stack:** Python · FastAPI · LangGraph · LangChain/OpenAI · Pydantic · tiktoken · tenacity

---

## 2. Architecture Layers

Three layers, with thin orchestration on top of self-contained modules:

```
demo_ui/index.html
        │
        ▼
   main.py (FastAPI)     ← HTTP: validate request, call pipeline, return JSON
        │
        ▼
   graph.py (LangGraph)  ← Orchestration: nodes + edges only — no LLM wiring
        │
        ▼
 production_modules/*    ← Business logic: one concern per file
        │
        ├── schema.py    ← Contracts: enums + TicketClassification + TicketState
        └── OpenAI (GPT-4o-mini)
```

| Layer | File(s) | Role |
|-------|---------|------|
| HTTP | [`main.py`](../main.py) | Validates request, calls `run_pipeline()`, returns JSON |
| Orchestration | [`graph.py`](../graph.py) | LangGraph nodes + edges only — no LLM wiring |
| Contracts | [`schema.py`](../schema.py) | Enums + `TicketClassification` + `TicketState` |
| Business logic | [`production_modules/`](../production_modules/) | Each concern in one copyable module |

### Design principle

`graph.py` is intentionally thin. Each node:

1. Reads from the shared state dict
2. Calls one production-module function
3. Returns `{**state, ...updated fields}`

LLM classification calls live only in `structured_output.py`. The injection guard LLM lives in `prompt_injection.py`. Nodes never build LangChain chains themselves.

---

## 3. End-to-End Workflow

```
POST /classify  { ticket_text, channel }
        │
        ▼
   pii_redact          strip email / phone / credit card
        │
        ▼
   injection_check     guard LLM on raw ticket
        │
        ▼
   classify            versioned prompt + JSON-mode LLM (skips if blocked)
        │
        ▼
   validate            Pydantic + business rules (skips if blocked)
        │
        ├── pass / blocked ──► cost_log ──► END
        │
        └── fail ──► fallback (retry up to 3×) ──► cost_log ──► END
        │
        ▼
ClassifyResponse JSON (main.py)
```


## 4. Quick Start (run locally)

```bash
pip3 install -r requirements.txt
cp .env.example .env   # set OPENAI_API_KEY
python3 -m uvicorn main:app --reload --port 8000
```

Then open `demo_ui/index.html` in a browser, or:

```bash
curl -X POST http://localhost:8000/classify \
  -H "Content-Type: application/json" \
  -d '{"ticket_text": "I was charged twice for order #9981!", "channel": "web_form"}'
```

Interactive docs: `http://localhost:8000/docs`

