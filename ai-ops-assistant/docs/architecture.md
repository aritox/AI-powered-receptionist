# Architecture

AI Ops Assistant separates language understanding from workflow control. The LLM interprets unstructured text, while n8n applies deterministic rules to decide what happens next. The platform is generic, with fintech and trading requests used as its initial demonstration scenarios.

## Phase 2 Integration Path

```text
Normalized Input
      ↓
HTTP Request
      ↓
Groq Chat Completions API
      ↓
LLM-generated response
```

During Phase 2, n8n receives the normalized `message`, invokes Groq through an authenticated HTTP POST request, and exposes the returned assistant text. This phase proves API communication only; structured classification and business decisions are not yet implemented.

### Phase 2 responsibilities

**n8n:**

- workflow orchestration
- input preparation
- API invocation
- future deterministic business logic

**Groq:**

- hosted LLM inference

**LLM:**

- natural-language understanding and generation

## Phase 3 Structured Classification

```text
Normalized Input
      ↓
Groq API
      ↓
Structured LLM Classification
      ↓
Parsed Operational Fields
```

Phase 3 changes the model response from free-form prose into seven predictable fields. Groq's strict JSON Schema response format constrains the shape and types, and n8n parses the resulting JSON string into top-level workflow data.

The architectural responsibilities are intentionally separate:

**LLM:** understands ambiguous human language and produces a context-aware operational assessment.

**JSON Schema:** constrains the LLM output to the required field names, types, and allowed enum values.

**n8n:** invokes the API, parses the structured result, and will use it for deterministic automation in later phases.

This separation is interview-relevant: the LLM handles language ambiguity, the schema creates a reliable contract, and the workflow engine remains responsible for operational control.

## Phase 4 Department Routing

```text
Incoming Request
      ↓
Normalize
      ↓
Groq
      ↓
Structured Analysis
      ↓
Route by Department
  ┌───────┼────────┐
  ↓       ↓        ↓
Support Compliance Trading Ops
```

Phase 4 uses an n8n Switch to compare the structured `department` field with three exact allowed values. The Switch does not interpret language or make an AI decision.

**LLM:**

- interprets ambiguous human language
- classifies the department

**n8n Switch:**

- evaluates the structured `department` value
- routes the item through a deterministic exact-match rule

**Department branch:**

- acts as the destination for later operational actions
- currently contains only a temporary queue marker for testing

This boundary keeps probabilistic language understanding separate from predictable workflow control.

## Phase 5 Human-Review Safety Gate

```text
Incoming Request
      ↓
Normalize
      ↓
Groq
      ↓
Structured Analysis
      ↓
Department Routing
      ↓
Human Review Safety Gate
      │
 ┌────┴────┐
 ↓         ↓
Review   Assisted
Queue     Flow
```

The safety gate combines the LLM's `requires_human_review` recommendation with deterministic overrides for High-priority and Compliance cases. Any matching condition sends the request to the Human Review Queue marker; otherwise it enters the Assisted Flow marker.

**AI layer:**

- understands language
- recommends risk and review status

**Deterministic workflow layer:**

- enforces hard safety rules
- overrides an unsafe AI recommendation when priority is High or the department is Compliance

**Human layer:**

- remains responsible for sensitive operational decisions

The Phase 5 nodes only mark workflow state. They do not notify a reviewer, pause through an approval mechanism, send a suggested response, or perform an account action.

## Phase 6 Audit Logging

```text
Incoming Request
      ↓
Normalize
      ↓
Groq
      ↓
Structured Analysis
      ↓
Department Routing
      ↓
Human Review Safety Gate
      │
 ┌────┴────┐
 ↓         ↓
Human    Assisted
Review    Flow
   \       /
    \     /
      ↓
 Audit Logging
      ↓
ai_ops_audit_log
```

Both final Phase 5 branches feed one n8n Data Table node. Because only one branch executes for a request, the workflow inserts one minimized audit row into `ai_ops_audit_log`.

**AI layer:**

- language understanding
- classification
- recommendation

**Deterministic workflow layer:**

- department routing
- safety policy enforcement

**Human layer:**

- consequential operational review

**Logging layer:**

- selected business-level traceability
- classification, routing, review status, and workflow-state record
- no direct customer name, email, raw request, or credential storage

The audit table complements n8n's technical execution history; it is a narrow business record rather than a copy of the full workflow payload.

## Testing and Validation

AI Ops Assistant is validated end-to-end with realistic, adversarial, and unsupported-knowledge requests. AI-generated wording is evaluated semantically for relevance, grounding, and safety rather than exact phrasing. Structured schema, department routing, safety-gate behavior, branch execution, and audit-row creation are tested as exact deterministic assertions.

Audit logging provides a business-level trace of each classification and workflow outcome, while privacy checks confirm that direct customer identifiers, raw request text, and credentials are excluded. The test matrix also includes conflicting form data and prompt-injection-style input to verify that customer text remains untrusted data.

## Request Flow

1. **Receive an event:** An n8n form trigger will receive a customer or internal request.
2. **Normalize the input:** n8n will clean and organize the submitted fields into a consistent format.
3. **Send the request to Groq:** n8n will pass the request text to the Groq Chat Completions API through an authenticated HTTP POST request.
4. **Analyze the request:** A Groq-hosted LLM understands the request, summarizes it, classifies its department, and estimates its urgency.
5. **Return structured JSON:** Groq strict Structured Outputs enforce a JSON Schema containing the operational fields that n8n parses and can evaluate reliably.
6. **Route deterministically:** An n8n Switch inspects `department` and routes the request by exact match to Customer Support, Compliance, or Trading Operations.
7. **Apply the human-review safety gate:** An n8n IF node combines the AI review signal with deterministic High-priority and Compliance overrides, then marks the item for review or assisted handling.
8. **Log the result:** One shared Data Table node inserts a minimized classification, route, review decision, workflow state, and timestamp into `ai_ops_audit_log`.

## Separation of Responsibilities

### AI reasoning

The LLM is responsible for:

- understanding unstructured text
- producing a short summary
- classifying the request
- estimating urgency
- suggesting an action or response

### Workflow and business logic

n8n is responsible for:

- calling APIs
- validating and evaluating structured fields
- routing requests
- enforcing deterministic business rules
- triggering human review
- logging workflow executions

This separation keeps the workflow explainable: AI provides an assessment, but n8n and human reviewers control operational decisions.
