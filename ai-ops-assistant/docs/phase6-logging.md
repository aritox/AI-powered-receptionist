# Phase 6: Audit Logging and Traceability

Phase 6 stores one minimized business-level audit row after either final Phase 5 branch. It uses n8n's built-in Data Table capability and does not add an external database or service.

```text
Human Review Queue ──┐
                     ├──→ Log Request
Assisted Flow ───────┘        ↓
                        ai_ops_audit_log
```

## Why Logging Matters

An AI workflow should not operate as a black box. For each processed request, the audit record should show:

- how the request was classified
- where it was routed
- how urgent it was considered
- whether human review was required
- which final workflow state was reached
- what response the model suggested
- when the workflow processed it, using n8n's automatic `createdAt` timestamp

This supports debugging, auditing, monitoring, quality review, and later evaluation of AI behavior.

## Privacy and Data Minimization

The audit table intentionally does not store:

- `customer_name`
- `customer_email`
- the full raw customer `message`
- API keys or credentials

The purpose is operational traceability, not duplication of all request data. Only the selected structured analysis and workflow-state fields are logged.

Model-generated fields such as `summary` and `suggested_response` can still contain sensitive details if the model repeats them. A production implementation should define retention periods, access controls, masking or redaction, encryption, and PII policies according to company requirements.

## Data Table Support

n8n's current Data Table capability stores structured data inside the n8n project. The Data Table node supports the **Row → Insert** operation and manual mapping of incoming values to table columns.

Official references:

- [Data tables overview](https://docs.n8n.io/build/work-with-data/data-tables/)
- [Data Table node](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.datatable/)
- [Data Table row operations](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.datatable/rows/)

If **Data Tables** or the **Data Table** node is absent from the installed UI, check the installed n8n version and its matching documentation before continuing. Do not substitute Google Sheets or an external database for this phase.

## Audit Table Schema

Create one table named:

```text
ai_ops_audit_log
```

Use these columns exactly:

| Column | Type | Purpose |
| --- | --- | --- |
| `department` | String | AI-classified department |
| `category` | String | AI-classified request category |
| `priority` | String | AI-assigned priority |
| `sentiment` | String | AI-assigned sentiment |
| `summary` | String | Concise operational summary |
| `requires_human_review` | Boolean | AI review recommendation |
| `department_route` | String | Deterministic department route |
| `routing_status` | String | Department-routing result |
| `review_status` | String | Safety-gate review result |
| `workflow_status` | String | Final Phase 5 workflow state |
| `suggested_response` | String | Employee-facing AI draft |

These are the 11 user-defined audit columns. n8n's built-in `createdAt` field supplies the row creation timestamp and does not need to be added to the table schema.

## Timestamp Strategy

The tested workflow uses the automatic Data Table field:

```text
createdAt
```

n8n sets `createdAt` when it inserts the row. Do not create or map a separate `logged_at` column, and do not hardcode a timestamp.

## Manual n8n Configuration

The current environment cannot access the user's running n8n browser instance. Apply and test these changes manually in the existing **AI Ops Assistant** workflow. Do not import an unverified workflow export.

### 1. Create `ai_ops_audit_log`

1. Open the n8n project containing **AI Ops Assistant**.
2. From the project Overview, select the **Data Tables** tab.
3. Select the split button in the upper-right corner, then select **Create Data table**.
4. Enter the table name `ai_ops_audit_log`.
5. Select **From scratch**.
6. Create all 11 columns from the schema table above.
7. Set `requires_human_review` to **Boolean**.
8. Set the other 10 columns to **String**.
9. Save the table.
10. Verify that n8n displays its automatic `createdAt` field.
11. Verify that the table does not contain `logged_at`, `customer_name`, `customer_email`, `message`, or any credential column.

### 2. Add one shared Data Table node

1. Return to the workflow canvas.
2. Add a **Data Table** node.
3. Open its node menu, select **Rename**, and enter `Log Request`.
4. Connect **Human Review Queue** to **Log Request**.
5. Connect **Assisted Flow** to the same **Log Request** node.
6. Confirm both connections terminate at this one logging node.

Only one Phase 5 branch executes for an item, so the executed branch inserts one row. The Data Table node does not wait for both branches and no Merge node is needed.

### 3. Select Insert

Configure **Log Request** as follows:

| Field | Value |
| --- | --- |
| Resource | `Row` |
| Operation | `Insert` |
| Data table selection | `From list` |
| Data table | `ai_ops_audit_log` |
| Mapping Column Mode | `Map Each Column Manually` |

Do not select Update or Upsert. Each workflow request should create a new audit row.

### 4. Map every audit column

Open the node's **Input** panel and verify the current item contains the structured, routing, and review fields. Then map:

| Data Table column | Type | n8n expression |
| --- | --- | --- |
| `department` | String | `{{ $json.department }}` |
| `category` | String | `{{ $json.category }}` |
| `priority` | String | `{{ $json.priority }}` |
| `sentiment` | String | `{{ $json.sentiment }}` |
| `summary` | String | `{{ $json.summary }}` |
| `requires_human_review` | Boolean | `{{ $json.requires_human_review }}` |
| `department_route` | String | `{{ $json.department_route }}` |
| `routing_status` | String | `{{ $json.routing_status }}` |
| `review_status` | String | `{{ $json.review_status }}` |
| `workflow_status` | String | `{{ $json.workflow_status }}` |
| `suggested_response` | String | `{{ $json.suggested_response }}` |

Use Expression mode for every mapped value. Do not enter test values as fixed text. Do not add mappings for customer name, customer email, or the raw message.

## Test 1: Human-Review Case

Use a fresh full-form submission:

```text
Name: Alice Martin
Email: alice@example.com
Request Type: Security Concern
Message: I noticed several transactions in my account that I do not recognize.
```

Expected path:

```text
Compliance Queue
→ Human Review Required? true
→ Human Review Queue
→ Log Request
```

After execution:

1. Open **Log Request** and confirm the Insert operation succeeded once.
2. Open **Data Tables → ai_ops_audit_log**.
3. Confirm one new row broadly contains:

```text
department = Compliance
priority = High
requires_human_review = true
department_route = Compliance
review_status = Human review required
workflow_status = Paused for human review
```

4. Confirm n8n's automatic `createdAt` field contains the row creation timestamp.
5. Confirm no Alice name, email address, or full raw message was written to a column.

## Test 2: Assisted Case

Use another fresh full-form submission:

```text
Name: Bob Turner
Email: bob@example.com
Request Type: General Question
Message: Can I trade gold and EUR/USD on the platform?
```

Expected path:

```text
Customer Support Queue
→ Human Review Required? false
→ Assisted Flow
→ Log Request
```

After execution, confirm the second row broadly contains:

```text
department = Customer Support
priority = Low
requires_human_review = false
department_route = Customer Support
review_status = No human review required
workflow_status = Eligible for assisted handling
```

## Verify Both Rows

After the two tests, verify that `ai_ops_audit_log` contains two separate new rows:

1. One represents the human-review case.
2. One represents the assisted-handling case.
3. Their automatic `createdAt` timestamps differ.
4. Their structured classifications are retained.
5. Their review and workflow statuses differ correctly.
6. Neither row contains customer name, customer email, or the full raw message.
7. Neither row contains an API key or credential value.

If the table already contained rows before testing, compare the row count before and after and confirm that it increased by exactly two.

## Audit Table vs n8n Execution History

**n8n execution history** contains technical workflow execution and debugging information, such as node inputs, outputs, timings, and errors.

**`ai_ops_audit_log`** is a deliberately selected business-level record of how an operational request was classified and handled.

The audit table contains only fields intentionally chosen for traceability. It does not replace execution history, and execution history should also be governed by appropriate retention and access policies because technical payloads may contain more data.

## Troubleshooting

### Data Table node is missing

Check the installed n8n version and the matching official documentation. Upgrade through the user's normal n8n installation process if Data Tables are required and unavailable; do not introduce an external database as a workaround in Phase 6.

### A mapped value is undefined

Inspect the executed **Human Review Queue** or **Assisted Flow** output. Confirm it preserves all structured, routing, and review fields, then remap the value from the Data Table node's Input panel.

### The timestamp is not visible

Open the Data Table view and inspect n8n's built-in `createdAt` field. Do not add a custom timestamp unless a later requirement explicitly calls for one.

### Duplicate rows are inserted

Confirm there is only one **Log Request** node, each final branch has one connection to it, and only one final branch executes per workflow item.

## Phase Boundary

Phase 6 is complete only after the local Data Table contains one verified human-review row and one verified assisted-flow row with no direct customer identifiers, raw message, or credentials. Broader test coverage and evaluation belong to Phase 7.
