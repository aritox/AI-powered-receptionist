# Phase 1: Incoming Request Form and Input Normalization

Phase 1 establishes the workflow entry point and a predictable data shape. It contains only an n8n Form Trigger followed by an Edit Fields (Set) node.

```text
Incoming User
      ↓
n8n Form Trigger
      ↓
Edit Fields / Set
      ↓
Normalized Request Data
```

## What is a trigger?

A trigger is an event that starts an n8n workflow. For AI Ops Assistant, the initial trigger is a user submitting an operational request through a form.

## Why use an n8n Form Trigger?

The n8n Form Trigger provides a simple input interface without requiring a separate frontend application. It simulates requests arriving from customers or internal employees and keeps this prototype small.

## Why normalize input?

Form labels are designed to be readable by people. Downstream systems work more reliably with predictable, machine-friendly field names.

For example, this form input:

```text
Name
Email
Request Type
Message
```

becomes:

```json
{
  "customer_name": "...",
  "customer_email": "...",
  "request_type": "...",
  "message": "..."
}
```

This normalized structure will be sent to the Groq API in a later phase.

```text
Human-friendly input
        ↓
Normalized machine-friendly data
        ↓
AI processing
```

## Manual n8n Configuration

n8n is not installed in the current project environment, so the workflow must be configured and tested in the user's n8n browser instance. Do not add any other nodes in Phase 1.

### 1. Create the workflow

1. Open n8n at `http://localhost:5678`.
2. Select **Create Workflow** (or **New Workflow**, depending on the installed version).
3. Rename the workflow to **AI Ops Assistant**.
4. Save the workflow.

### 2. Add the Form Trigger

1. Select **Add first step**.
2. Search for **n8n Form Trigger** and select it.
3. Set **Form Title** to `AI Operations Request`.
4. In **Form Elements**, add the following four fields:

| Field label | Field type | Required | Options or notes |
| --- | --- | --- | --- |
| Name | Text | Yes | Single-line text |
| Email | Email | Yes | Use Text if Email is unavailable |
| Request Type | Dropdown | Yes | Add the seven options listed below |
| Message | Textarea / multi-line Text | Yes | Main unstructured request text |

Add these **Request Type** options exactly:

1. `General Question`
2. `Account Issue`
3. `Verification / KYC`
4. `Deposit / Withdrawal`
5. `Trading Issue`
6. `Security Concern`
7. `Other`

### 3. Inspect the real trigger output

1. Select **Test workflow** or **Execute step** on the Form Trigger.
2. Open the displayed test form.
3. Submit the test values from the testing section below.
4. Return to the editor and inspect the Form Trigger's **Output** panel.
5. Confirm the actual output keys for Name, Email, Request Type, and Message. n8n normally uses the form labels as keys, but the displayed output is the source of truth.

### 4. Add Edit Fields (Set)

1. Select the **+** connector after the Form Trigger.
2. Search for **Edit Fields (Set)**. In older n8n versions, select **Set**.
3. Keep the connection as `Form Trigger → Edit Fields (Set)`.
4. Select **Manual Mapping** mode.
5. Enable **Keep Only Set Fields** or disable **Include Other Input Fields**, if that option exists in the installed version. This makes the output contain only the four normalized fields.
6. Add four String fields using the expression picker:

| Output field | Expression after verifying trigger output |
| --- | --- |
| `customer_name` | `{{ $json.Name }}` |
| `customer_email` | `{{ $json.Email }}` |
| `request_type` | `{{ $json['Request Type'] }}` |
| `message` | `{{ $json.Message }}` |

Do not hardcode sample values. If the trigger output shows different key spelling or capitalization, drag the corresponding value from the Form Trigger output into the expression field so n8n generates the real path.

## Test the Workflow

1. Start **Test workflow**.
2. Open the Form Trigger test URL.
3. Submit:

```text
Name: Alice Martin
Email: alice@example.com
Request Type: Verification / KYC
Message: My passport verification has failed twice even though my passport is valid.
```

4. Return to the workflow editor.
5. Open **Edit Fields (Set)** and inspect its output.
6. Confirm these four fields and values are present:

```json
{
  "customer_name": "Alice Martin",
  "customer_email": "alice@example.com",
  "request_type": "Verification / KYC",
  "message": "My passport verification has failed twice even though my passport is valid."
}
```

n8n metadata may also appear unless the node is configured to keep only set fields. Phase 1 is complete only after the four normalized values have been verified in a real execution.

## Phase Boundary

Stop after the successful Edit Fields output. Groq integration, AI analysis, routing, human review, and logging belong to later phases.
