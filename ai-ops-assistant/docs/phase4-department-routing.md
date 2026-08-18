# Phase 4: Deterministic Department Routing

Phase 4 routes each structured analysis to one of three simulated department queues. The LLM has already interpreted and classified the request; the Switch node only evaluates the resulting `department` string.

```text
Extract Structured Analysis
          ↓
  Switch — Route by Department
     ┌────────┼────────┐
     ↓        ↓        ↓
 Customer  Compliance  Trading
 Support               Operations
```

## Why Use Deterministic Routing After AI?

An LLM is useful when the input is ambiguous human language. For example:

```text
"My passport keeps getting rejected."
```

The LLM can interpret that message and produce:

```text
department = Compliance
```

Once `department` is a structured field, there is no reason to ask the LLM where to send the request again. n8n can apply an exact rule:

```text
department == Compliance
→ Compliance branch
```

The complete separation is:

```text
Human language
      ↓
LLM interpretation
      ↓
Structured department
      ↓
Deterministic Switch routing
```

This design provides predictable behavior, easier debugging, lower AI usage, clear business rules, and easier auditing.

## Switch vs IF

An **IF** node is appropriate for a binary decision, such as true versus false. A **Switch** node is more appropriate when one value can lead to several routes.

AI Ops Assistant has three possible `department` values:

- Customer Support
- Compliance
- Trading Operations

A single Switch therefore expresses the routing more clearly than several chained IF nodes.

## Routing Contract

The Switch receives `department` from **Extract Structured Analysis** and applies exact string equality. It does not perform AI reasoning, fuzzy matching, or classification.

| Rule order | Incoming value | Output name | Destination node |
| --- | --- | --- | --- |
| 1 | `Customer Support` | `Customer Support` | Customer Support Queue |
| 2 | `Compliance` | `Compliance` | Compliance Queue |
| 3 | `Trading Operations` | `Trading Operations` | Trading Operations Queue |

## Manual n8n Configuration

The current environment cannot access the user's running n8n browser instance. Apply and test these changes manually in the existing workflow. Do not import an unverified workflow export.

### 1. Add and rename the Switch

1. Open the existing **AI Ops Assistant** workflow.
2. Select the **+** connector after **Extract Structured Analysis**.
3. Search for **Switch** and add it.
4. Open the node menu, select **Rename**, and enter `Route by Department`.
5. Confirm the connection is `Extract Structured Analysis → Route by Department`.
6. Open the Switch node's **Input** panel and verify the current item contains `department` as a top-level String field.

### 2. Select Rules mode

1. In **Route by Department**, set **Mode** to `Rules`.
2. Under **Routing Rules**, create three rules in the exact order below.
3. If the node has these options, configure:

| Option | Value |
| --- | --- |
| Fallback Output | `None` |
| Ignore Case | Off |
| Less Strict Type Validation | Off |
| Send data to all matching outputs | Off |

With `Ignore Case` off, the comparisons use exact capitalization. With **Send data to all matching outputs** off, an item stops at its first matching rule.

### 3. Configure Rule 1 — Customer Support

1. In the first routing rule, choose the **String** data type.
2. Set **Value 1** to Expression mode.
3. Enter:

```javascript
{{ $json.department }}
```

4. Set the comparison operation to **is equal to** (shown as **equals** in some versions).
5. Keep **Value 2** in Fixed mode and enter `Customer Support`.
6. Enable **Rename Output**.
7. Set **Output Name** to `Customer Support`.

### 4. Configure Rule 2 — Compliance

1. Select **Add Routing Rule**.
2. Choose the **String** data type.
3. Set **Value 1** to:

```javascript
{{ $json.department }}
```

4. Select **is equal to** / **equals**.
5. Set fixed **Value 2** to `Compliance`.
6. Enable **Rename Output** and set **Output Name** to `Compliance`.

### 5. Configure Rule 3 — Trading Operations

1. Select **Add Routing Rule**.
2. Choose the **String** data type.
3. Set **Value 1** to:

```javascript
{{ $json.department }}
```

4. Select **is equal to** / **equals**.
5. Set fixed **Value 2** to `Trading Operations`.
6. Enable **Rename Output** and set **Output Name** to `Trading Operations`.

### 6. Understand the outputs

The Switch creates one output for each rule. Because the rules were added and named in order:

```text
Output 1 / Customer Support  → Customer Support Queue
Output 2 / Compliance        → Compliance Queue
Output 3 / Trading Operations → Trading Operations Queue
```

Use the displayed output names when connecting nodes. Do not infer a branch from connector position alone.

## Add the Temporary Queue Markers

These Edit Fields nodes only prove which branch ran. They do not send data to any external system.

### Customer Support Queue

1. Select the **Customer Support** output connector from **Route by Department**.
2. Add **Edit Fields (Set)**, or **Set** in an older n8n version.
3. Rename it to `Customer Support Queue`.
4. Select **Manual Mapping**.
5. Enable **Include Other Input Fields**. If the version instead shows **Include in Output**, select **All Input Fields**. In older versions, keep **Keep Only Set Fields** disabled.
6. Add these two String fields in Fixed mode:

| Field | Fixed value |
| --- | --- |
| `department_route` | `Customer Support` |
| `routing_status` | `Routed successfully` |

### Compliance Queue

1. Select the **Compliance** output connector.
2. Add **Edit Fields (Set)** and rename it to `Compliance Queue`.
3. Select **Manual Mapping**.
4. Enable **Include Other Input Fields** / select **All Input Fields**.
5. Add these two String fields in Fixed mode:

| Field | Fixed value |
| --- | --- |
| `department_route` | `Compliance` |
| `routing_status` | `Routed successfully` |

### Trading Operations Queue

1. Select the **Trading Operations** output connector.
2. Add **Edit Fields (Set)** and rename it to `Trading Operations Queue`.
3. Select **Manual Mapping**.
4. Enable **Include Other Input Fields** / select **All Input Fields**.
5. Add these two String fields in Fixed mode:

| Field | Fixed value |
| --- | --- |
| `department_route` | `Trading Operations` |
| `routing_status` | `Routed successfully` |

Each queue output should retain:

- `department`
- `category`
- `priority`
- `sentiment`
- `summary`
- `requires_human_review`
- `suggested_response`
- `department_route`
- `routing_status`

The completed Phase 4 shape is:

```text
Extract Structured Analysis
          ↓
    Route by Department
   ┌────────┼───────────┐
   ↓        ↓           ↓
Customer  Compliance  Trading Operations
Support     Queue          Queue
 Queue
```

## Full-Form Tests

Use a fresh full form submission for each test. After every execution, inspect the execution path on the canvas and confirm that only the expected queue node ran.

### Test 1 — Compliance

Submit:

```text
Name: Alice Martin
Email: alice@example.com
Request Type: Security Concern
Message: I noticed several transactions in my account that I do not recognize.
```

Expected path:

```text
Form
→ Normalize
→ Groq
→ Structured Analysis
→ Route by Department
→ Compliance Queue
```

Verify:

```text
department = Compliance
department_route = Compliance
routing_status = Routed successfully
```

**Customer Support Queue** and **Trading Operations Queue** must not execute for this item.

### Test 2 — Customer Support

Submit:

```text
Name: Bob Turner
Email: bob@example.com
Request Type: General Question
Message: Can I trade gold and EUR/USD on the platform?
```

Expected destination: **Customer Support Queue**.

Verify:

```text
department = Customer Support
department_route = Customer Support
routing_status = Routed successfully
```

The other two queue nodes must not execute.

### Test 3 — Trading Operations

Submit:

```text
Name: Emma Stone
Email: emma@example.com
Request Type: Trading Issue
Message: The BTC trading interface returns an error every time I try to submit an order.
```

Expected destination: **Trading Operations Queue**.

Verify:

```text
department = Trading Operations
department_route = Trading Operations
routing_status = Routed successfully
```

The other two queue nodes must not execute.

## Troubleshooting

### No Switch output executes

Inspect the Switch input and compare the exact `department` value with the three fixed rule values. Check capitalization and whitespace, and confirm the expression is `{{ $json.department }}`.

### The wrong queue executes

Check each rule's output name and trace its connector to the queue node. Verify the fixed `department_route` value in that queue.

### Structured fields disappear

Enable **Include Other Input Fields**, choose **All Input Fields**, or disable **Keep Only Set Fields**, depending on the installed Edit Fields version.

### More than one branch executes

Turn off **Send data to all matching outputs** and verify that the three fixed values are distinct.

## Phase Boundary

Phase 4 is complete only after all three full-form tests reach exactly one correct queue and preserve the structured analysis. Human-review routing, external actions, and logging belong to later phases.
