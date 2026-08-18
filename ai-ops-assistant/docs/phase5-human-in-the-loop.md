# Phase 5: Human-in-the-Loop Safety Gate

Phase 5 adds a deterministic safety decision after department routing. It marks sensitive cases for human review and allows low-risk cases to enter an assisted path. It does not notify a reviewer, approve a request, send a response, or perform an operational action.

```text
Customer Support Queue ─┐
Compliance Queue ───────┼──→ Human Review Required?
Trading Operations ─────┘
                                  │
                           ┌──────┴──────┐
                           │             │
                          TRUE          FALSE
                           │             │
                           ↓             ↓
                  Human Review Queue   Assisted Flow
```

## What Is Human-in-the-Loop?

Human-in-the-loop means AI assists with a process while a person remains responsible for sensitive or consequential decisions.

In AI Ops Assistant, AI helps with:

- interpreting requests
- classification
- summarization
- prioritization
- suggested responses

It does not authorize or perform sensitive actions. This matters in financial and operational systems because LLMs can misclassify or hallucinate, account-specific facts may be unavailable, security and compliance decisions may be consequential, and financial actions should not be performed solely from generated text.

## Do Not Trust Only the LLM Boolean

The structured AI output contains `requires_human_review`. This is useful as an AI recommendation, but it is not the workflow's only safety control.

n8n applies this deterministic policy:

```text
requires_human_review == true
OR
priority == "High"
OR
department == "Compliance"
```

Conceptually:

```text
LLM recommendation
        +
deterministic safety policy
        ↓
final workflow decision
```

The two deterministic overrides protect High-priority and Compliance cases even if the LLM incorrectly returns `requires_human_review = false`.

## Safety Examples

### Example A — Low-risk support request

```text
department = Customer Support
priority = Low
requires_human_review = false
```

Result: **FALSE — no human review required**.

### Example B — High-priority override

```text
department = Customer Support
priority = High
requires_human_review = false
```

Result: **TRUE — human review required**, because High priority overrides the AI recommendation.

### Example C — Compliance override

```text
department = Compliance
priority = Medium
requires_human_review = false
```

Result: **TRUE — human review required**, because Compliance cases are deterministically protected.

### Example D — AI review recommendation

```text
department = Trading Operations
priority = Medium
requires_human_review = true
```

Result: **TRUE — human review required**, because the AI explicitly recommends review.

## Core Architecture Principle

```text
AI layer:
understands language and recommends risk/review status

Deterministic workflow layer:
enforces hard safety rules

Human layer:
makes sensitive operational decisions
```

Generative AI should not be used for every decision simply because it is available. Once the LLM has produced structured fields, predictable policy checks belong in deterministic workflow logic.

## Manual n8n Configuration

The current environment cannot access the user's running n8n browser instance. Apply and test these changes manually in the existing **AI Ops Assistant** workflow. Do not import an unverified workflow export.

### 1. Add one shared IF node

1. Open the existing workflow.
2. Add an **IF** node to the canvas.
3. Open its node menu, select **Rename**, and enter `Human Review Required?`.
4. Connect the output of **Customer Support Queue** to the input of **Human Review Required?**.
5. Connect the output of **Compliance Queue** to the same IF input.
6. Connect the output of **Trading Operations Queue** to the same IF input.
7. Confirm that all three queue connections terminate at this single IF node.

The Switch executes only one department branch per request, so only the queue on that branch sends an item into the IF node. The IF does not merge or wait for all three queues.

### 2. Configure OR logic

1. Open **Human Review Required?**.
2. In **Conditions**, set the combination mode to **OR**, shown in some versions as **When any condition is met**.
3. Do not use AND / **When all conditions are met**.

The current n8n IF node supports adding conditions and choosing OR when an item should pass if any condition matches.

### 3. Condition 1 — AI review signal

1. Add the first condition.
2. Select the **Boolean** data type.
3. Set **Value 1** to Expression mode.
4. Enter:

```javascript
{{ $json.requires_human_review }}
```

5. Select the Boolean operation **is true**.

Do not compare this value with the String `"true"`; the Phase 3 field is a Boolean.

### 4. Condition 2 — High-priority override

1. Select **Add condition**.
2. Select the **String** data type.
3. Set **Value 1** to Expression mode and enter:

```javascript
{{ $json.priority }}
```

4. Select **is equal to** (or **equals**, depending on the version).
5. Keep **Value 2** in Fixed mode and enter `High`.

### 5. Condition 3 — Compliance override

1. Select **Add condition**.
2. Select the **String** data type.
3. Set **Value 1** to Expression mode and enter:

```javascript
{{ $json.department }}
```

4. Select **is equal to** / **equals**.
5. Keep **Value 2** in Fixed mode and enter `Compliance`.

Before saving, verify that the IF reads:

```text
requires_human_review is true
OR priority equals High
OR department equals Compliance
```

## Add the TRUE Branch Marker

1. Select the **true** output of **Human Review Required?**.
2. Add **Edit Fields (Set)**, or **Set** in an older n8n version.
3. Rename it to `Human Review Queue`.
4. Select **Manual Mapping**.
5. Enable **Include Other Input Fields**. If the installed version shows **Include in Output**, select **All Input Fields**. In an older version, keep **Keep Only Set Fields** disabled.
6. Add these two String fields in Fixed mode:

| Field | Fixed value |
| --- | --- |
| `review_status` | `Human review required` |
| `workflow_status` | `Paused for human review` |

This node marks the case as requiring review. It does not mean a person has reviewed it, and it does not mean an escalation or notification has occurred.

## Add the FALSE Branch Marker

1. Select the **false** output of **Human Review Required?**.
2. Add **Edit Fields (Set)**, or **Set** in an older n8n version.
3. Rename it to `Assisted Flow`.
4. Select **Manual Mapping**.
5. Enable **Include Other Input Fields** / select **All Input Fields**. In an older version, keep **Keep Only Set Fields** disabled.
6. Add these two String fields in Fixed mode:

| Field | Fixed value |
| --- | --- |
| `review_status` | `No human review required` |
| `workflow_status` | `Eligible for assisted handling` |

This branch does not resolve the request and must not send `suggested_response` automatically.

## Preserved Output Contract

Both branch markers should preserve:

- `department`
- `category`
- `priority`
- `sentiment`
- `summary`
- `requires_human_review`
- `suggested_response`
- `department_route`
- `routing_status`

and add:

- `review_status`
- `workflow_status`

## Full-Form Tests

Use a fresh submission from the n8n Form for every test. Inspect the execution path on the canvas and verify which IF output and final marker executed.

### Test 1 — Security / Compliance TRUE path

Submit:

```text
Name: Alice Martin
Email: alice@example.com
Request Type: Security Concern
Message: I noticed several transactions in my account that I do not recognize.
```

Expected path:

```text
Compliance Queue
      ↓
Human Review Required?
      ↓ true
Human Review Queue
```

Expected broadly:

```text
department = Compliance
priority = High
review_status = Human review required
workflow_status = Paused for human review
```

Verify that **Assisted Flow** does not execute.

### Test 2 — Low-risk General Question FALSE path

Submit:

```text
Name: Bob Turner
Email: bob@example.com
Request Type: General Question
Message: Can I trade gold and EUR/USD on the platform?
```

Expected path:

```text
Customer Support Queue
      ↓
Human Review Required?
      ↓ false
Assisted Flow
```

Expected broadly:

```text
department = Customer Support
priority = Low
requires_human_review = false
review_status = No human review required
workflow_status = Eligible for assisted handling
```

Verify that **Human Review Queue** does not execute.

### Test 3 — Trading Issue TRUE path

Submit:

```text
Name: Emma Stone
Email: emma@example.com
Request Type: Trading Issue
Message: The BTC trading interface returns an error every time I try to submit an order.
```

Expected broadly:

```text
Trading Operations Queue
      ↓
Human Review Required?
      ↓ true
Human Review Queue
```

The TRUE path should be selected if `priority = High` or `requires_human_review = true`. Inspect the actual structured output to identify which policy condition matched.

## Troubleshooting

### A sensitive case goes to Assisted Flow

Confirm the conditions are combined with OR, not AND. Verify that the expressions use the top-level fields from the department queue output.

### The Boolean condition does not match

Confirm `requires_human_review` is a Boolean and use the **is true** operation rather than comparing it with a text value.

### Preserved analysis fields disappear

Enable **Include Other Input Fields**, choose **All Input Fields**, or disable **Keep Only Set Fields**, depending on the Edit Fields version.

### Both final markers appear to run

Inspect one individual workflow execution. Each item should leave the IF through only its true or false output, not both.

## Phase Boundary

Phase 5 is complete only after real n8n tests validate both the TRUE and FALSE paths. The two final nodes are status markers only. Notifications, approvals, customer responses, external integrations, and logging belong to later work.
