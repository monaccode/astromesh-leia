# WhatsApp Channel Configuration Reference

This document covers the configuration and integration details for deploying astromesh agents as WhatsApp bots via Meta's Cloud API.

## Environment Variables

These environment variables must be set on the astromesh-node running the agent. They are typically configured as Kubernetes Secrets and injected into the node pod.

| Variable | Required | Description |
|----------|----------|-------------|
| `WHATSAPP_VERIFY_TOKEN` | REQUIRED | Token you define for Meta webhook verification. Must match the value configured in the Meta App Dashboard. Any string you choose. |
| `WHATSAPP_ACCESS_TOKEN` | REQUIRED | Permanent access token from Meta Business Suite. Used to send messages via the Graph API. |
| `WHATSAPP_PHONE_NUMBER_ID` | REQUIRED | The Phone Number ID from Meta Business Suite (not the phone number itself). Found under WhatsApp > API Setup. |
| `WHATSAPP_APP_SECRET` | REQUIRED | App Secret from Meta App Dashboard > Settings > Basic. Used for webhook signature validation. |

**Example Kubernetes Secret:**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: whatsapp-credentials
  namespace: default
type: Opaque
stringData:
  WHATSAPP_VERIFY_TOKEN: "my-secure-verify-token-2026"
  WHATSAPP_ACCESS_TOKEN: "EAAx..."
  WHATSAPP_PHONE_NUMBER_ID: "123456789012345"
  WHATSAPP_APP_SECRET: "abcdef1234567890"
```

## Webhook Flow

> **Per-agent webhook endpoint.** Each agent exposes its own webhook on the node:
> `GET/POST /v1/agents/{agent-name}/channels/whatsapp/webhook`. The GET verifies the
> token; the POST receives messages. Register the public URL of this path
> (e.g. via a tunnel or ingress) in the Meta App Dashboard. The `/webhook` paths
> below are shown unqualified for brevity.
>
> **Sender name in prompts.** Incoming WhatsApp messages carry the sender's display
> name (from Meta's `contacts[]`), which the runtime passes to the agent as
> `contact_name` (astromesh v0.27.0+). Reference it in the system prompt:
> `{% if contact_name %}Address the user as {{ contact_name }}.{% endif %}`.
> Delivery/read/failed receipts are surfaced as `system`-direction channel events,
> not routed to the agent.

### GET -- Meta Verification

When you register the webhook URL in the Meta App Dashboard, Meta sends a GET request to verify ownership.

**Request from Meta:**
```
GET /webhook?hub.mode=subscribe&hub.verify_token=<your-token>&hub.challenge=<challenge-string>
```

**Expected behavior:**
1. The node checks that `hub.verify_token` matches `WHATSAPP_VERIFY_TOKEN`.
2. If it matches, respond with `200 OK` and the `hub.challenge` value as the body.
3. If it does not match, respond with `403 Forbidden`.

### POST -- Incoming Messages

When a user sends a message on WhatsApp, Meta delivers it as a POST to the webhook.

**Request from Meta:**
```
POST /webhook
Content-Type: application/json
X-Hub-Signature-256: sha256=<hmac-hex>
```

**Signature validation:**
1. Read the raw request body.
2. Compute HMAC-SHA256 using `WHATSAPP_APP_SECRET` as the key.
3. Compare with the value in `X-Hub-Signature-256` header (after stripping the `sha256=` prefix).
4. Reject the request with `401 Unauthorized` if signatures do not match.

**Webhook payload structure (simplified):**
```json
{
  "object": "whatsapp_business_account",
  "entry": [{
    "id": "BIZ_ACCOUNT_ID",
    "changes": [{
      "value": {
        "messaging_product": "whatsapp",
        "metadata": {
          "display_phone_number": "15551234567",
          "phone_number_id": "123456789012345"
        },
        "messages": [{
          "from": "15559876543",
          "id": "wamid.abc123...",
          "timestamp": "1712052600",
          "type": "text",
          "text": { "body": "Hi, I'm interested in your product" }
        }]
      },
      "field": "messages"
    }]
  }]
}
```

## Message Flow

```
User (WhatsApp)
    |
    | sends message
    v
Meta Cloud API
    |
    | POST /webhook (with X-Hub-Signature-256)
    v
astromesh-node (webhook handler)
    |
    | 1. Validate signature
    | 2. Extract message text/media
    | 3. Pass to agent runtime
    v
Agent Runtime (LLM + tools + memory)
    |
    | generates response
    v
astromesh-node (response sender)
    |
    | POST https://graph.facebook.com/v21.0/{phone_id}/messages
    v
Meta Graph API
    |
    | delivers message
    v
User (WhatsApp)
```

## Sending Messages via Graph API

**Endpoint:**
```
POST https://graph.facebook.com/v21.0/{WHATSAPP_PHONE_NUMBER_ID}/messages
```

**Headers:**
```
Authorization: Bearer {WHATSAPP_ACCESS_TOKEN}
Content-Type: application/json
```

**Text message body:**
```json
{
  "messaging_product": "whatsapp",
  "to": "15559876543",
  "type": "text",
  "text": {
    "body": "Thanks for reaching out! How can I help you today?"
  }
}
```

**Response (200):**
```json
{
  "messaging_product": "whatsapp",
  "contacts": [{"input": "15559876543", "wa_id": "15559876543"}],
  "messages": [{"id": "wamid.xyz789..."}]
}
```

## Multimedia Handling

| Media Type | Inbound Handling | Notes |
|------------|-----------------|-------|
| **Images** | Downloaded from Meta CDN, converted to base64 data URL, passed to the agent as a vision input. | Requires a vision-capable model (e.g., `gpt-4o`, `llama3.2-vision`). |
| **Audio** | Downloaded, transcribed to text (via Whisper or provider STT), passed as text. | Transcription adds latency; consider timeout adjustments. |
| **Video** | Not processed inline. A text description is generated: `"[Video received: {duration}s, {mime_type}]"`. | Full video processing is not supported in v0.1.x. |
| **Documents** | Not processed inline. A text description is generated: `"[Document received: {filename}, {mime_type}, {size}]"`. | PDF extraction planned for v0.2.x. |
| **Location** | Extracted as latitude/longitude, passed as text: `"Location shared: {lat}, {lng}"`. | -- |
| **Contacts** | Extracted as structured text with name and phone number. | -- |

## Agent YAML Recommendations for WhatsApp

When deploying an agent for WhatsApp, apply these settings for reliable message delivery:

### Timeout and Token Limits

WhatsApp users expect fast replies. Meta may retry delivery if your webhook takes too long to acknowledge (5-second ack window).

```yaml
spec:
  model:
    primary:
      timeout: 30              # Keep under 30s for user experience
      parameters:
        max_tokens: 1024       # WhatsApp messages have ~4096 char limit; keep responses concise
  orchestration:
    max_iterations: 5          # Fewer iterations = faster responses
    # No orchestration.timeout_seconds: the 0.68 runtime does not read it (no run-level
    # timeout). The model block's `timeout` (HTTP, per call) is the only limit.
```

### Vision Model for Images

If the agent needs to handle image messages, use a vision-capable model:

```yaml
spec:
  model:
    primary:
      provider: openai
      model: gpt-4o            # Vision-capable
    fallback:
      provider: ollama
      model: llama3.2-vision:11b
```

### PII and Guardrails (core 0.70+)

WhatsApp conversations frequently contain personal information (names, phone
numbers, addresses). From core **0.70.0** `spec.guardrails` is enforced, so the
architect should emit:

```yaml
spec:
  guardrails:
    input:
      - type: max_length
        max_chars: 4000          # WhatsApp caps a message at 4096
    output:
      - type: pii_detection
        action: redact           # do not repeat emails/phones in replies
      - type: cost_limit
        max_tokens_per_turn: 4000
```

Rules of thumb:

- **Do not use input `pii_detection`** if the agent or its tools need the
  customer's phone, customer ID or order number: input redaction hides emails,
  phones and 10/16-digit numbers from the model and the tools. Redact output.
- Skip output redaction when the agent must confirm a name/phone back
  (bookings, lead capture).
- PII patterns are US-format: an Argentine phone number is not detected, so
  don't promise full PII protection. Memory still stores what was sent; manage
  retention with the memory `ttl`.
- `max_length` caps the *incoming* message only. Keep replies short with the
  model's `max_tokens` and an explicit instruction in the system prompt.
- On core 0.68/0.69 none of this runs; tell the user to upgrade to 0.70+.
- See `schemas/astromesh-v1-agent.md`, *spec.guardrails*.

### Memory for Conversation Continuity

WhatsApp users expect the agent to remember context across messages:

```yaml
spec:
  memory:
    conversational:
      backend: redis           # Persistent across pod restarts; the only backend 0.68 builds
      connection:
        url: redis://redis:6379/0   # REQUIRED: without it the agent runs with no memory
      strategy: sliding_window
      max_turns: 50
      ttl: 86400               # 24 hours (runtime default is 259200 = 72 h)
```

### Recommended Labels

```yaml
metadata:
  labels:
    channel: whatsapp
    messaging-product: whatsapp
```
