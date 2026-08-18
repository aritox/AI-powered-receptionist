# Phase 2: Groq API Integration

Phase 2 proves that AI Ops Assistant can send the normalized request message from n8n to a hosted LLM and expose the returned text. It does not add structured classification or business routing.

```text
n8n Form
    ↓
Edit Fields
    ↓
HTTP Request
    ↓
Groq Chat Completions API
    ↓
LLM Response
```

## What is an API?

An API is a defined interface that allows two applications to communicate. One application sends a request in an agreed format, and the other returns a response.

In this project, n8n orchestrates the workflow and Groq provides hosted LLM inference:

```text
n8n
 ↓
HTTP request
 ↓
Groq API
 ↓
LLM
 ↓
HTTP response
 ↓
n8n
```

## What is an HTTP POST request?

An HTTP POST request sends data to an API endpoint. This request contains:

- a URL or endpoint
- authentication
- headers describing the request
- a JSON request body

The Groq Chat Completions endpoint is:

```text
POST https://api.groq.com/openai/v1/chat/completions
```

## Authentication and Secret Handling

Groq authenticates requests with an API key sent as a Bearer token:

```text
Authorization: Bearer <MY_GROQ_API_KEY>
```

The placeholder above explains the format; it is not a credential. Enter the real value manually in n8n's credential manager only.

The real key must never appear in README files, source files, screenshots, workflow documentation, Git commits, or exported workflow JSON. Review any workflow export for secrets before committing it.

## Selected Model

Use:

```text
openai/gpt-oss-20b
```

As verified against Groq's official documentation on August 18, 2026, this is a current lightweight production text model suitable for this small request-understanding test. The earlier `llama-3.1-8b-instant` model was deprecated and shut down for free and developer tiers on August 16, 2026, so it must not be used.

References:

- [Groq supported models](https://console.groq.com/docs/models)
- [Groq model deprecations](https://console.groq.com/docs/deprecations)
- [Groq Chat Completions API](https://console.groq.com/docs/api-reference)

Model availability can change. Recheck Groq's supported-model list if the API reports that the selected model is unavailable.

## Manual n8n Configuration

The current environment does not provide direct access to the user's n8n browser instance. Configure and test the following nodes in the existing **AI Ops Assistant** workflow. Do not export an untested workflow.

### 1. Add the HTTP Request node

1. Open the existing **AI Ops Assistant** workflow.
2. Select the **+** connector after **Edit Fields (Set)**.
3. Search for and select **HTTP Request**.
4. Open the node menu and select **Rename**.
5. Rename it to **Groq - Analyze Request**.
6. Confirm the connection is `n8n Form → Edit Fields (Set) → Groq - Analyze Request`.

### 2. Configure the request

In **Groq - Analyze Request**, enter:

| Field | Value |
| --- | --- |
| Method | `POST` |
| URL | `https://api.groq.com/openai/v1/chat/completions` |
| Authentication | `Generic Credential Type` |
| Generic Auth Type | `Header Auth` |
| Send Body | On |
| Body Content Type | `JSON` |
| Specify Body | `Using JSON` |

The authentication labels may be presented as **Generic Credential Type → Header Auth** or simply **Header Auth**, depending on the installed n8n version.

### 3. Create secure Header Auth credentials

1. In the node's **Credential for Header Auth** field, select **Create New Credential**.
2. Name the credential `Groq Header Auth`.
3. Enter the following credential fields:

```text
Header Name: Authorization
Header Value: Bearer <your real Groq API key>
```

4. Replace the placeholder with the real key manually inside n8n.
5. Save the credential.
6. Never paste the resulting value into this repository, a screenshot, a workflow export, or a support message.

### 4. Configure the JSON body

The input from **Edit Fields (Set)** has already been verified to contain `message`. Before configuring the body, open the node's **Input** panel and confirm that `message` is still present.

In **Specify Body → Using JSON**, switch the JSON field to **Expression** mode and enter this object expression:

```javascript
{{ ({
  model: 'openai/gpt-oss-20b',
  messages: [
    {
      role: 'system',
      content: 'You are an AI operations assistant. Read the incoming operational request and briefly explain what the request is about in one sentence. Do not make financial decisions and do not invent information.'
    },
    {
      role: 'user',
      content: 'Customer request: ' + $json.message
    }
  ],
  temperature: 0.2
}) }}
```

This dynamically reads `message` from the preceding Edit Fields node. It does not hardcode a customer request. The equivalent request body is conceptually:

```json
{
  "model": "openai/gpt-oss-20b",
  "messages": [
    {
      "role": "system",
      "content": "You are an AI operations assistant. Read the incoming operational request and briefly explain what the request is about in one sentence. Do not make financial decisions and do not invent information."
    },
    {
      "role": "user",
      "content": "Customer request: <dynamic message from Edit Fields>"
    }
  ],
  "temperature": 0.2
}
```

n8n normally adds `Content-Type: application/json` when **Body Content Type** is JSON. If it does not appear in the request configuration, enable **Send Headers**, add a header named `Content-Type`, and set its value to `application/json`.

## Understanding the Groq Response

The generated assistant text is found at:

```text
choices[0].message.content
```

The path means:

```text
Groq response
      ↓
choices
      ↓
first completion
      ↓
message
      ↓
content
```

`content` is the natural-language text generated by the LLM.

## Extract the AI Response

Adding one more Edit Fields node makes the final Phase 2 result easier to demonstrate.

1. Select the **+** connector after **Groq - Analyze Request**.
2. Add **Edit Fields (Set)**, or **Set** in an older n8n version.
3. Rename it to **Extract AI Response**.
4. Choose **Manual Mapping**.
5. Disable **Include Other Input Fields**.
6. Add a String field named `ai_response`.
7. Set its value to this expression:

```javascript
{{ $json.choices[0].message.content }}
```

8. Confirm the connection is:

```text
n8n Form
    ↓
Edit Fields (Set)
    ↓
Groq - Analyze Request
    ↓
Extract AI Response
```

Do not add classification, priority, sentiment, routing, or review fields in Phase 2.

## Test Scenario

1. Select **Test workflow**.
2. Open the n8n form test URL.
3. Submit:

```text
Name: Alice Martin
Email: alice@example.com
Request Type: Deposit / Withdrawal
Message: My withdrawal has been pending since yesterday and I have not received the funds.
```

4. Return to the workflow editor and inspect **Edit Fields (Set)**. Confirm the normalized `message` contains the submitted sentence.
5. Execute **Groq - Analyze Request** as part of the test workflow.
6. Open its **Output** panel and expand `choices → 0 → message → content`.
7. Confirm that `content` contains a short explanation. Exact wording is not important; a response may resemble `This request concerns a customer reporting a delayed withdrawal.`
8. Execute **Extract AI Response** and confirm its output resembles:

```json
{
  "ai_response": "This request concerns a customer reporting a delayed withdrawal."
}
```

The communication test is successful when Groq returns generated text and the final node exposes it as `ai_response`.

## Common Errors

### 401 / authentication error

The API key may be invalid, expired, or missing, or the Header Auth credential may not use `Authorization` with the `Bearer ` prefix. Check the credential inside n8n without exposing it.

### 400 / bad request

The JSON body may be malformed, a parameter may be unsupported, or the model ID may be invalid. Check the node's resolved request body and Groq's current supported-model list.

### 429 / rate limit

The Groq usage limit has been reached temporarily. Wait and retry later. Sophisticated retry logic is outside Phase 2.

### Expression returns undefined

The mapping from **Edit Fields (Set)** is probably incorrect or that node did not execute in the current test. Inspect its output, confirm the field is named `message`, and select it from the expression editor.

## Phase Boundary

Stop after `ai_response` is successfully verified. Structured output, classification, routing, human review, and logging belong to later phases.
