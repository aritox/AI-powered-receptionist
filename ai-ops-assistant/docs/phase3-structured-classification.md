# Phase 3: Structured AI Classification

Phase 3 replaces the free-text AI response with a strict, machine-readable operational analysis. It ends with seven structured fields; routing and other business actions remain outside this phase.

```text
n8n Form
      ↓
Normalize Input
      ↓
Groq - Analyze Request
      ↓
Structured JSON
      ↓
Extract Structured Analysis
```

## Free Text vs Structured Output

A free-text result such as:

```json
{
  "ai_response": "This appears to be a withdrawal issue."
}
```

is readable and useful to a person, but an automation cannot reliably infer a department, priority, or review decision from arbitrary wording.

A structured result is easier to automate:

```json
{
  "department": "Customer Support",
  "priority": "High",
  "requires_human_review": true
}
```

n8n can evaluate these predictable fields programmatically in later phases.

```text
Unstructured request
        ↓
LLM understanding
        ↓
Structured business data
        ↓
Deterministic automation
```

## Groq Structured Outputs

Phase 3 uses Groq's JSON Schema response format instead of relying only on a prompt that asks the model to return JSON:

```json
{
  "response_format": {
    "type": "json_schema",
    "json_schema": {
      "name": "operations_triage",
      "strict": true,
      "schema": {}
    }
  }
}
```

Groq's strict mode uses constrained decoding to enforce the schema. For `strict: true`, every property must be listed in `required`, and every object must set `additionalProperties` to `false`. No fields are optional in this phase.

As verified against [Groq's Structured Outputs documentation](https://console.groq.com/docs/structured-outputs) on August 18, 2026, `openai/gpt-oss-20b` supports strict JSON Schema output.

## Operational Schema

The model must return exactly these seven fields:

```json
{
  "type": "object",
  "properties": {
    "department": {
      "type": "string",
      "enum": [
        "Customer Support",
        "Compliance",
        "Trading Operations"
      ]
    },
    "category": {
      "type": "string"
    },
    "priority": {
      "type": "string",
      "enum": [
        "Low",
        "Medium",
        "High"
      ]
    },
    "sentiment": {
      "type": "string",
      "enum": [
        "Neutral",
        "Positive",
        "Frustrated",
        "Concerned",
        "Angry"
      ]
    },
    "summary": {
      "type": "string"
    },
    "requires_human_review": {
      "type": "boolean"
    },
    "suggested_response": {
      "type": "string"
    }
  },
  "required": [
    "department",
    "category",
    "priority",
    "sentiment",
    "summary",
    "requires_human_review",
    "suggested_response"
  ],
  "additionalProperties": false
}
```

## Prototype Classification Rules

These are demonstration rules for workflow testing, not legally authoritative policies.

### Customer Support

Use for general questions, account questions, deposits, withdrawals, ordinary platform usage, product availability, and customer assistance.

### Compliance

Use for KYC, identity verification, suspicious transactions, AML-related concerns, account security concerns, unauthorized activity, and regulatory-sensitive issues.

### Trading Operations

Use for order execution issues, trading interface failures, market or trading-platform incidents, order submission errors, execution anomalies, and trading-system technical problems.

## Priority Rules

### High

Use for unauthorized activity, security concerns, suspicious transactions, major trading or execution problems, potentially sensitive compliance issues, and significant customer-funds issues requiring prompt investigation.

### Medium

Use when employee investigation is required but the issue is not immediately critical, such as repeated KYC failure, a delayed operational process, or an account issue requiring manual review.

### Low

Use for informational questions, product questions, normal usage questions, and other non-urgent requests.

## Human-Review Rules

`requires_human_review` should normally be `true` for security issues, unauthorized transactions, suspicious activity, KYC or compliance decisions, account restrictions, financial disputes, trading execution problems, sensitive withdrawal or deposit investigations, and anything requiring access to private account information.

It may be `false` for low-risk informational requests.

The LLM must not claim to have checked an account, verified a transaction, approved KYC, processed a withdrawal, executed an order, or accessed internal company systems unless that information was actually supplied.

## Suggested-Response Rules

`suggested_response` is an employee-facing draft. It should acknowledge the request, remain concise, avoid unsupported promises or invented account information, avoid claiming an action was completed, avoid financial advice, and recommend escalation when appropriate.

Acceptable example:

```text
We understand that your withdrawal has been delayed. The case should be reviewed by the appropriate support team to verify its status.
```

Unsupported claim to avoid:

```text
Your withdrawal will arrive in 30 minutes.
```

## Manual n8n Configuration

The current environment cannot access the user's running n8n browser instance. Modify and test the existing nodes manually; do not import an unverified workflow export.

### 1. Open the existing HTTP Request node

1. Open the **AI Ops Assistant** workflow.
2. Select **Groq - Analyze Request**.
3. Preserve these existing settings:

| Field | Existing value |
| --- | --- |
| Method | `POST` |
| URL | `https://api.groq.com/openai/v1/chat/completions` |
| Authentication | Existing secure Header Auth credential |
| Send Body | On |
| Body Content Type | `JSON` |
| Specify Body | `Using JSON` |

4. Do not open, replace, copy, or expose the existing credential value.
5. Open the node's **Input** panel and verify that the preceding **Edit Fields** node supplies both `request_type` and `message`.

### 2. Replace the JSON body

In **Specify Body → Using JSON**, keep the field in **Expression** mode and replace the previous Phase 2 body with this exact object expression:

```javascript
{{ ({
  model: 'openai/gpt-oss-20b',
  messages: [
    {
      role: 'system',
      content: [
        'You are an AI operations triage assistant.',
        'Analyze incoming operational requests and convert them into structured information for workflow automation.',
        'Classify each request into exactly one department: Customer Support, Compliance, or Trading Operations.',
        'Customer Support covers general and account questions, deposits, withdrawals, ordinary platform usage, product availability, and customer assistance.',
        'Compliance covers KYC, identity verification, suspicious transactions, AML-related concerns, account security, unauthorized activity, and regulatory-sensitive issues.',
        'Trading Operations covers order execution issues, trading interface failures, trading-platform incidents, order submission errors, execution anomalies, and trading-system technical problems.',
        'Assign priority Low, Medium, or High. High includes unauthorized activity, security concerns, suspicious transactions, major execution problems, sensitive compliance issues, and significant customer-funds issues requiring prompt investigation. Medium requires employee investigation but is not immediately critical. Low is informational or non-urgent.',
        'Determine sentiment, write a concise factual summary, determine whether human review is required, and generate a short employee-facing suggested response.',
        'Human review should normally be required for security, unauthorized or suspicious activity, KYC or compliance decisions, account restrictions, financial disputes, trading execution problems, sensitive deposits or withdrawals, and requests requiring private account access.',
        'Do not invent account information. Do not claim to have checked an account, verified a transaction, approved KYC, processed a withdrawal, executed an order, or accessed internal systems.',
        'Do not make financial decisions or provide investment advice. Avoid unsupported promises. Recommend escalation when appropriate.'
      ].join('\n')
    },
    {
      role: 'user',
      content: 'User-selected request type:\n' + $json.request_type + '\n\nCustomer message:\n' + $json.message
    }
  ],
  temperature: 0.1,
  response_format: {
    type: 'json_schema',
    json_schema: {
      name: 'operations_triage',
      strict: true,
      schema: {
        type: 'object',
        properties: {
          department: {
            type: 'string',
            enum: ['Customer Support', 'Compliance', 'Trading Operations']
          },
          category: {
            type: 'string'
          },
          priority: {
            type: 'string',
            enum: ['Low', 'Medium', 'High']
          },
          sentiment: {
            type: 'string',
            enum: ['Neutral', 'Positive', 'Frustrated', 'Concerned', 'Angry']
          },
          summary: {
            type: 'string'
          },
          requires_human_review: {
            type: 'boolean'
          },
          suggested_response: {
            type: 'string'
          }
        },
        required: [
          'department',
          'category',
          'priority',
          'sentiment',
          'summary',
          'requires_human_review',
          'suggested_response'
        ],
        additionalProperties: false
      }
    }
  }
}) }}
```

The two dynamic mappings are:

```javascript
{{ $json.request_type }}
{{ $json.message }}
```

They are included in the object expression as `$json.request_type` and `$json.message`. Do not replace them with test values.

### 3. Execute and inspect the HTTP response

1. Use the form's test mode to submit the withdrawal scenario below.
2. Return to the editor and confirm **Edit Fields** outputs the submitted `request_type` and `message`.
3. Execute **Groq - Analyze Request**.
4. Open its **Output** panel.
5. Expand `choices → 0 → message → content`.
6. Confirm that `content` is a JSON string containing exactly the seven schema fields.

The HTTP response still includes Groq metadata such as `id`, `object`, `model`, `choices`, and `usage`. The structured classification is inside:

```text
choices[0].message.content
```

Although it represents JSON, `content` is a string, conceptually:

```text
"{\"department\":\"Customer Support\", ...}"
```

It must be parsed before individual fields can be exposed to later n8n nodes.

### 4. Replace the extraction configuration

1. Select the existing **Extract AI Response** Edit Fields node.
2. Rename it to **Extract Structured Analysis**.
3. Keep **Manual Mapping** selected.
4. Keep **Include Other Input Fields** disabled.
5. Delete the old `ai_response` field.
6. Add exactly these seven fields and expressions:

| Field name | n8n type | Expression |
| --- | --- | --- |
| `department` | String | `{{ JSON.parse($json.choices[0].message.content).department }}` |
| `category` | String | `{{ JSON.parse($json.choices[0].message.content).category }}` |
| `priority` | String | `{{ JSON.parse($json.choices[0].message.content).priority }}` |
| `sentiment` | String | `{{ JSON.parse($json.choices[0].message.content).sentiment }}` |
| `summary` | String | `{{ JSON.parse($json.choices[0].message.content).summary }}` |
| `requires_human_review` | Boolean | `{{ JSON.parse($json.choices[0].message.content).requires_human_review }}` |
| `suggested_response` | String | `{{ JSON.parse($json.choices[0].message.content).suggested_response }}` |

Use the **Input** panel to confirm the observed response path before saving. The path above matches the already verified Chat Completions response structure, but the running node output is the source of truth.

The workflow should now be:

```text
n8n Form
      ↓
Edit Fields
      ↓
Groq - Analyze Request
      ↓
Extract Structured Analysis
```

Do not add a Switch, IF, routing branch, notification, or logging node.

## Withdrawal Test

Run the workflow in test mode with:

```text
Name: Alice Martin
Email: alice@example.com
Request Type: Deposit / Withdrawal
Message: My withdrawal has been pending since yesterday and I have not received the funds.
```

Verify that **Extract Structured Analysis** returns exactly seven top-level fields with the correct types. An acceptable result resembles:

```json
{
  "department": "Customer Support",
  "category": "Withdrawal Issue",
  "priority": "Medium",
  "sentiment": "Concerned",
  "summary": "Customer reports a withdrawal pending since yesterday and has not received the funds.",
  "requires_human_review": true,
  "suggested_response": "Acknowledge the delay and have the withdrawal status reviewed by the appropriate support team."
}
```

Exact wording and borderline priority choices may vary. Verify schema validity, types, sensible classification, and the absence of invented facts.

## Troubleshooting

### HTTP 400 after adding `response_format`

Check that the model is `openai/gpt-oss-20b`, all seven properties appear in `required`, the root object has `additionalProperties: false`, and the JSON/expression syntax is valid.

### A dynamic value is undefined

Inspect the preceding **Edit Fields** output and confirm it contains `request_type` and `message`. Reinsert both values using n8n's expression picker if necessary.

### `JSON.parse` fails

Inspect `choices[0].message.content`. Confirm the HTTP request used `response_format.type: json_schema` and that `content` contains a JSON string rather than an error message.

### A field has the wrong n8n type

Configure `requires_human_review` as Boolean. Configure the other six fields as String.

## Phase Boundary

Phase 3 is complete only after a real n8n test produces the seven parsed fields. Department routing, human-review branching, notifications, and logging belong to later phases.
