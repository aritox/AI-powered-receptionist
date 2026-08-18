<img width="1765" height="885" alt="workflow" src="https://github.com/user-attachments/assets/c9fae462-27fc-43aa-824c-35aceaec7648" />
# AI Ops Assistant

## Project Overview

An AI-powered operations automation prototype that analyzes incoming requests, structures information, assists with routing and prioritization, and keeps humans involved in sensitive decisions.

The project is intentionally lightweight, generic, and interview-focused. It uses fintech and trading requests as its initial demonstration domain, but the workflow can be adapted to other companies and operational teams.

## Business Problem

Operational teams receive many unstructured customer and internal requests. Team members must repeatedly understand each request, classify it, assess its priority, and send it to the correct destination.

This prototype demonstrates how AI can accelerate those repetitive triage tasks while keeping people responsible for sensitive decisions. Financial, compliance, security, and trading scenarios provide realistic initial examples.

## Planned Architecture

```text
Incoming Request
      ↓
n8n Form Trigger
      ↓
Input Preparation / Normalization
      ↓
Groq API
      ↓
LLM Analysis
      ↓
Structured JSON Output
      ↓
Department Routing
 ┌──────────┼───────────┐
 │          │           │
Support  Compliance  Trading Ops
 └──────────┼───────────┘
            ↓
Human Review Decision
       ┌────┴────┐
       │         │
      YES        NO
       │         │
   Escalate   Normal Flow
       └────┬────┘
            ↓
          Logging
```

## Planned Stack

- **n8n** — workflow orchestration
- **Groq API** — hosted LLM inference
- **LLM** — request understanding and classification
- **JSON structured output** — machine-readable AI results
- **n8n Switch / IF** — deterministic business logic
- **Logging** — workflow traceability

## Planned Departments

- Customer Support
- Compliance
- Trading Operations

## Human-in-the-Loop Principle

AI will assist with classification, summarization, prioritization, and suggested actions or responses. Sensitive financial, compliance, security, or trading decisions will require human review before action is taken.

## Planned Phases

1. Environment and repository setup
2. Incoming request form
3. Groq API integration
4. Structured AI classification
5. Department routing
6. Human-in-the-loop decision logic
7. Logging and traceability
8. Testing with realistic scenarios
9. Interview documentation and demo preparation

## Progress

- Phase 0 — Environment & repository setup ✅
- Phase 0.5 — Groq architecture migration ✅
- Phase 1 — Incoming request form ✅
- Phase 2 — Groq API integration ✅
- Phase 3 — Structured AI classification ✅
- Phase 4 — Department routing ✅
- Phase 5 — Human-in-the-loop logic ✅
- Phase 6 — Logging & traceability ✅
- Phase 7 — End-to-end testing ⏳ Manual execution required
- Phase 8 — Interview demo preparation ⏳

## Security

- API keys must never be committed to Git.
- API keys must never appear in screenshots.
- Credentials should be stored using n8n credential management or environment variables.
- Workflow exports must be reviewed for secrets before they are pushed to GitHub.

## Project Scope

This is an interview prototype that favors a clear, explainable workflow and free or low-cost development tools. No credentials, production integrations, or automated business decisions are included at this stage.
