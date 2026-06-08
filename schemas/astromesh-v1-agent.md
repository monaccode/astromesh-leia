# astromesh/v1 Agent YAML Schema Reference

This document is the authoritative schema reference for `astromesh/v1 Agent` manifests. Subagents (architect, operator, doctor) MUST consult this file when generating or validating agent YAML.

## Top-Level Structure

Every agent manifest requires exactly these four top-level keys:

```yaml
apiVersion: astromesh/v1    # REQUIRED, must be exactly "astromesh/v1"
kind: Agent                  # REQUIRED, must be exactly "Agent"
metadata:                    # REQUIRED, see metadata section
spec:                        # REQUIRED, see spec section
```

All four keys are REQUIRED. Any manifest missing one of these is invalid.

---

## metadata (REQUIRED)

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `name` | string | REQUIRED | -- | Unique agent identifier. Must conform to RFC 1123 label: lowercase alphanumeric and hyphens, 1-63 characters, must start and end with alphanumeric. Regex: `[a-z0-9]([a-z0-9\-]{0,61}[a-z0-9])?` |
| `version` | string | optional | `"0.1.0"` | Semantic version of the agent definition. |
| `namespace` | string | optional | `"default"` | Kubernetes namespace for deployment. Must already exist in the cluster. |
| `labels` | object | optional | `{}` | Key-value string pairs for organizing and selecting agents. Keys and values must be strings. |

**Validation rules:**
- `name` is rejected if it contains uppercase letters, underscores, dots, or starts/ends with a hyphen.
- `namespace` must be a valid Kubernetes namespace (same RFC 1123 rules as `name`).
- `labels` keys must match: `([a-z0-9A-Z][-a-z0-9A-Z_.]*)?[a-z0-9A-Z]`

---

## spec.identity (REQUIRED)

Human-readable metadata about the agent. The section itself is required but all fields within are optional.

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `display_name` | string | optional | value of `metadata.name` | Human-friendly name shown in dashboards and logs. |
| `description` | string | optional | `""` | One-line description of what the agent does. Used in agent-to-agent tool descriptions. |
| `avatar` | string | optional | `""` | URL or emoji identifier for the agent avatar. |

---

## spec.model (REQUIRED)

Configures which LLM(s) the agent uses.

### spec.model.primary (REQUIRED)

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `provider` | string | REQUIRED | -- | LLM provider. **Runtime-wired values** (safe to deploy): `ollama`, `openai`, `openai_compat`, `azure_openai`. The schema *also* recognizes `vllm`, `llamacpp`, `huggingface`, `onnx`, but these are **not yet wired in the runtime engine** (deploy logs a warning and they won't serve). To reach a vLLM / llama.cpp / TGI server today, use `openai_compat` with its `endpoint`. |
| `model` | string | REQUIRED | -- | Model identifier (e.g., `llama3.1:8b`, `gpt-4o`, `qwen2.5:7b`). |
| `endpoint` | string | optional | provider-dependent | API endpoint URL. Defaults: `http://localhost:11434` for `ollama`, `https://api.openai.com/v1` for `openai`. Required for `openai_compat` and `azure_openai`. |
| `api_key` | string | optional | -- | API key as a literal string. Mutually exclusive with `api_key_env`. Avoid committing secrets; prefer `api_key_env`. |
| `api_key_env` | string | optional | provider-dependent | Environment variable name containing the API key. Default: `OPENAI_API_KEY` for `openai` provider. |
| `timeout` | number | optional | `120` | Request timeout in seconds. |
| `parameters` | object | optional | `{}` | Model generation parameters. See below. |

**parameters sub-fields:**

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `temperature` | number | `0.7` | Sampling temperature, 0.0-2.0. Lower = more deterministic. |
| `top_p` | number | `1.0` | Nucleus sampling threshold, 0.0-1.0. |
| `top_k` | integer | provider default | Top-k sampling. Honored by `ollama`/`openai_compat` backends that support it. |
| `max_tokens` | integer | `2048` | Maximum tokens in the response. |
| `frequency_penalty` | number | `0.0` | Penalize token frequency, -2.0 to 2.0. |
| `presence_penalty` | number | `0.0` | Penalize token presence, -2.0 to 2.0. |
| `stop` | string \| string[] | -- | One or more stop sequences that halt generation. |

> `temperature` and `max_tokens` may also be set at the top level of a provider block (shorthand) in addition to nesting them under `parameters`.

**Validation rules:**
- `provider` must be one of the four allowed values.
- `model` must be a non-empty string.
- For `openai_compat` and `azure_openai`, `endpoint` is effectively required (no default).
- `api_key` and `api_key_env` are mutually exclusive; specifying both is an error.
- `timeout` must be a positive number.
- `temperature` must be between 0.0 and 2.0 inclusive.
- `top_p` must be between 0.0 and 1.0 inclusive.
- `max_tokens` must be a positive integer.

### spec.model.fallback (optional)

Same structure as `spec.model.primary`. Used when the primary model is unavailable or returns errors. The runtime tries the fallback after exhausting retries on the primary.

### spec.model.extra (optional) — added in astromesh v0.28.0

A map of **additional named providers** registered alongside `primary` and `fallback`. Every registered slot (primary + fallback + each `extra` entry) is ranked together by the configured routing `strategy` — so `extra` lets one agent fan out across more than two models without being limited to the primary/fallback pair.

| Field | Type | Description |
|-------|------|-------------|
| `<slot_name>` | model block | Each key is an arbitrary slot name (e.g. `cheap`, `vision`, `local`); each value is a full provider block with the same shape as `primary`. |

**Validation rules:**
- `extra` must be a mapping (object). A non-object value is ignored with a warning.
- The reserved slot names `primary` and `fallback` are **rejected** inside `extra` (they would shadow the canonical slots) and that entry is skipped with a warning.

**Example:**
```yaml
model:
  primary:
    provider: openai
    model: gpt-4o
    api_key_env: OPENAI_API_KEY
  fallback:
    provider: ollama
    model: llama3.1:8b
  extra:
    cheap:
      provider: openai
      model: gpt-4o-mini
      api_key_env: OPENAI_API_KEY
    local-vision:
      provider: ollama
      model: llama3.2-vision:11b
      endpoint: http://localhost:11434
  routing:
    strategy: cost_optimized
```

### spec.model.routing (optional)

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `strategy` | string | optional | `"cost_optimized"` | Routing strategy across all registered model slots (primary + fallback + `extra`). One of: `cost_optimized`, `latency_optimized`, `quality_first`, `round_robin`, `capability_match`. |
| `health_check_interval` | integer | optional | `30` | Seconds between health check pings to each model endpoint. |

**Strategy descriptions:**
- `cost_optimized` *(default)* -- rank by each provider's estimated cost; prefer the cheapest healthy model.
- `latency_optimized` -- rank by observed average latency; prefer the fastest healthy model.
- `round_robin` -- rotate requests across all healthy models in turn.
- `capability_match` -- filter to models that satisfy required capabilities (e.g. tools, vision) for the task.
- `quality_first` -- recognized, but currently a no-op ranking: slots keep their declared order (primary first), so it behaves like "prefer primary, fall back on failure".

> The router has a circuit breaker: 3 consecutive failures on a provider open it for a 60s cooldown (then half-open retry).

---

## spec.prompts (optional)

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `system` | string | optional | `""` | System prompt. Supports Jinja2 template syntax with variables: `{{agent_name}}`, `{{current_date}}`, `{{tools_list}}`, `{{context}}`, `{{memory}}`, and `{{contact_name}}` (the sender's display name on channels that provide it, e.g. WhatsApp — added in astromesh v0.27.0). Guard optional vars: `{% if contact_name %}Address the user as {{ contact_name }}.{% endif %}`. |
| `templates` | object | optional | `{}` | Named prompt templates. Keys are template names, values are Jinja2 template strings. Referenced in tools and orchestration. |

**Example:**
```yaml
prompts:
  system: |
    You are {{agent_name}}, a helpful assistant.
    Today is {{current_date}}.
  templates:
    greeting: "Hello! I'm {{agent_name}}. How can I help you today?"
    escalation: "Let me transfer you to a human agent for further assistance."
```

---

## spec.orchestration (optional)

Controls how the agent processes tasks and uses tools.

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `pattern` | string | optional | `"react"` | Orchestration pattern. One of: `react`, `plan_and_execute`, `parallel_fan_out`, `pipeline`, `supervisor`, `swarm`. See `schemas/orchestration-patterns.md` for details. |
| `max_iterations` | integer | optional | `10` | Maximum reasoning/action loops before the agent stops. Prevents infinite loops. |
| `timeout_seconds` | integer | optional | `30` | Maximum wall-clock seconds for a single request. |

**Validation rules:**
- `pattern` must be one of the six allowed values.
- `max_iterations` must be a positive integer, max 100.
- `timeout_seconds` must be a positive integer, max 300.

---

## spec.tools (optional)

Array of tool definitions. Each tool must have a `type` field. Three types are supported:

### Type: builtin

Pre-packaged tools provided by the astromesh runtime.

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `type` | string | REQUIRED | -- | Must be `"builtin"`. |
| `name` | string | REQUIRED | -- | Built-in tool name (e.g., `web_search`, `calculator`, `datetime`, `http_request`). |
| `config` | object | optional | `{}` | Tool-specific configuration. |
| `rate_limit` | object | optional | -- | Rate limiting: `{max_calls: int, period_seconds: int}`. |

### Type: agent

Delegates to another deployed agent as a tool.

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `type` | string | REQUIRED | -- | Must be `"agent"`. |
| `name` | string | REQUIRED | -- | Tool name as exposed to the LLM. |
| `agent` | string | REQUIRED | -- | `metadata.name` of the target agent. Must be deployed in the same namespace. |
| `description` | string | optional | target agent's `spec.identity.description` | Override description for the tool. |
| `parameters` | object | optional | -- | JSON Schema defining the input parameters for the agent tool call. |
| `context_transform` | string | optional | -- | Jinja2 template to transform context before passing to the sub-agent. |
| `rate_limit` | object | optional | -- | Rate limiting: `{max_calls: int, period_seconds: int}`. |

### Type: internal

Custom tools with inline logic defined by JSON Schema parameters. The runtime generates the tool interface; the agent's LLM decides when to call it.

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `type` | string | REQUIRED | -- | Must be `"internal"`. |
| `name` | string | REQUIRED | -- | Tool name as exposed to the LLM. |
| `description` | string | REQUIRED | -- | Description of what the tool does. |
| `parameters` | object | REQUIRED | -- | JSON Schema object defining the tool's input parameters. |

### Type: mcp_stdio / mcp_sse / mcp_http

Connect the agent to an external **MCP (Model Context Protocol) server** so its tools become callable by the agent's LLM. Pick the transport that matches the server: `mcp_stdio` (spawns a local process), `mcp_sse` (Server-Sent Events endpoint), or `mcp_http` (streamable HTTP endpoint).

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | REQUIRED | One of `mcp_stdio`, `mcp_sse`, `mcp_http`. |
| `name` | string | REQUIRED | Logical name for the MCP connection. |
| `config` | object | REQUIRED | Transport config. For `mcp_stdio`: `{command, args, env}`. For `mcp_sse`/`mcp_http`: `{url, headers}`. |

### Type: webhook

Calls an external HTTP endpoint as a tool.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | REQUIRED | Must be `"webhook"`. |
| `name` | string | REQUIRED | Tool name as exposed to the LLM. |
| `description` | string | REQUIRED | What the webhook does. |
| `config` | object | REQUIRED | `{url, method, headers}`. |
| `parameters` | object | optional | JSON Schema for the request payload. |

### Type: rag

Exposes a retrieval-augmented-generation knowledge source as a tool the LLM can query.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | REQUIRED | Must be `"rag"`. |
| `name` | string | REQUIRED | Tool name as exposed to the LLM. |
| `description` | string | REQUIRED | What knowledge the source contains. |
| `config` | object | REQUIRED | RAG source config (collection / index + retrieval settings). |

> Common per-tool fields apply to every type: `rate_limit` (`{max_calls, period_seconds}`), `requires_approval` (bool), and `timeout_seconds`.

**Example tools array:**
```yaml
tools:
  - type: builtin
    name: web_search
    rate_limit:
      max_calls: 10
      period_seconds: 60
  - type: agent
    name: lookup_inventory
    agent: inventory-agent
    description: "Check product availability and pricing"
  - type: internal
    name: score_lead
    description: "Score a lead from 0-100 based on qualification criteria"
    parameters:
      type: object
      properties:
        company_size:
          type: string
          enum: [startup, smb, enterprise]
        budget_range:
          type: string
        timeline:
          type: string
      required: [company_size]
```

---

## spec.memory (optional)

Configures agent memory backends.

### spec.memory.conversational (optional)

> **Structure matters:** the runtime reads conversational settings **nested under `conversational`** (i.e. `spec.memory.conversational.strategy`, `spec.memory.conversational.max_turns`). A flat block like `memory: {type: conversational, backend: ...}` is silently ignored and defaults are used. Always nest.

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `backend` | string | optional | `"in_memory"` | Storage backend. One of: `in_memory`, `sqlite`, `redis`, `postgres`. |
| `strategy` | string | optional | `"sliding_window"` | Memory management strategy. One of: `sliding_window`, `summary`, `token_budget`. |
| `max_turns` | integer | optional | `50` | Maximum conversation turns to retain (for `sliding_window`). This is the exact key the runtime reads (`max_messages` is an accepted alias). |
| `ttl` | integer | optional | `86400` | Time-to-live in seconds for memory entries. Default is 24 hours. |

### spec.memory.semantic (optional)

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `backend` | string | optional | `"chromadb"` | Vector store backend. One of: `chromadb`, `pgvector`, `qdrant`, `in_memory`. |
| `similarity_threshold` | number | optional | `0.75` | Minimum cosine similarity for retrieval, 0.0-1.0. |
| `max_results` | integer | optional | `10` | Maximum number of results to return per query. |

### spec.memory.episodic (optional)

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `backend` | string | optional | `"sqlite"` | Storage backend for episodic memory. |

---

## spec.guardrails (optional)

Input and output validation rules. Both `input` and `output` are arrays of rule objects.

### Rule types

| Rule Type | Applicable To | Fields | Description |
|-----------|--------------|--------|-------------|
| `pii_detection` | input, output | `action` (`redact`, `block`, `warn`, `log`) | Detects and handles emails, phone numbers, SSNs, credit cards. `redact` masks them in place. |
| `max_length` | input, output | `max_chars` (integer) | Truncates/rejects content over the limit. **Field is `max_chars`** — `limit` is ignored. |
| `topic_filter` | input | `blocked_topics` (string array), `action` | Blocks or warns on off-topic conversations. |
| `cost_limit` | output | `max_tokens_per_turn` (integer) | Caps output tokens per turn. (There is no per-request USD field in the runtime.) |
| `content_filter` | output | `blocked_patterns` (regex string array), `action` | Filters content matching any regex pattern. |

> The action vocabulary is `redact`, `block`, `warn`, `log`. `prompt_injection` appears in some schema examples but is **not implemented** in the current runtime — don't rely on it.

**Example:**
```yaml
guardrails:
  input:
    - type: pii_detection
      action: redact
    - type: max_length
      max_chars: 4096
    - type: topic_filter
      blocked_topics: [politics, religion]
      action: warn
  output:
    - type: content_filter
      blocked_patterns:
        - "(?i)\\b(kill yourself|kys)\\b"
      action: block
    - type: cost_limit
      max_tokens_per_turn: 1024
```

---

## spec.permissions (optional)

Controls what actions the agent is allowed to perform.

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `allowed_actions` | string array | optional | `["respond"]` | High-level action whitelist. Values: `respond`, `use_tools`, `delegate`, `escalate`. |
| `filesystem` | object | optional | `{}` | Filesystem access controls. |
| `filesystem.read` | string array | optional | `[]` | Glob patterns for allowed read paths. |
| `filesystem.write` | string array | optional | `[]` | Glob patterns for allowed write paths. |
| `network` | object | optional | `{}` | Network access controls. |
| `network.allowed` | string array | optional | `[]` | Allowed outbound hostnames or CIDR ranges. |
| `execution` | object | optional | `{}` | Execution controls. |
| `execution.dry_run` | boolean | optional | `false` | When true, tools log intended actions without executing. |

---

## Channels (deployment-time, not in the agent spec)

There is **no `spec.channels` block read by the runtime**. An agent is bound to a channel like WhatsApp at deployment time, through:

1. **Environment variables** on the astromesh-node running the agent (e.g. `WHATSAPP_VERIFY_TOKEN`, `WHATSAPP_ACCESS_TOKEN`, `WHATSAPP_PHONE_NUMBER_ID`, `WHATSAPP_APP_SECRET`), typically injected as Kubernetes Secrets.
2. **A per-agent webhook endpoint** exposed by the node: `GET/POST /v1/agents/{name}/channels/whatsapp/webhook` (GET verifies the token, POST receives messages). Register that URL in the Meta App Dashboard.

Use `metadata.labels` (e.g. `channel: whatsapp`) to *tag* a channel agent, but do not put channel credentials in the manifest. See `schemas/whatsapp-config.md` for the full WhatsApp flow. Supported channels today: **WhatsApp** (production-ready). Telegram is recognized by tooling but not yet implemented in the runtime.

---

## Complete Example

A sales lead qualification agent deployed on WhatsApp with all sections populated:

```yaml
apiVersion: astromesh/v1
kind: Agent
metadata:
  name: sales-qualifier
  version: "1.0.0"
  namespace: sales-team
  labels:
    team: sales
    channel: whatsapp
    tier: production

spec:
  identity:
    display_name: "Sales Qualifier"
    description: "Qualifies inbound leads through conversational assessment and scores them for the sales team."
    avatar: "https://cdn.example.com/avatars/sales-bot.png"

  model:
    primary:
      provider: openai
      model: gpt-4o
      api_key_env: OPENAI_API_KEY
      timeout: 30
      parameters:
        temperature: 0.4
        top_p: 0.95
        max_tokens: 1024
    fallback:
      provider: ollama
      model: llama3.1:8b
      endpoint: http://ollama.sales-team.svc.cluster.local:11434
      timeout: 60
      parameters:
        temperature: 0.4
        max_tokens: 1024
    routing:
      strategy: quality_first
      health_check_interval: 30

  prompts:
    system: |
      You are {{agent_name}}, a friendly sales qualification assistant for Acme Corp.
      Today is {{current_date}}.

      Your goal is to understand the prospect's needs, company size, budget, and timeline
      through natural conversation. Score every lead using the score_lead tool.

      Be professional but conversational. Never be pushy. If someone is not interested,
      thank them politely and end the conversation.

      Available tools: {{tools_list}}
    templates:
      greeting: |
        Hi there! Thanks for reaching out to Acme Corp. I'd love to learn more about
        what you're looking for. Could you tell me a bit about your company?
      qualified_handoff: |
        Great news! Based on our conversation, I think we have a great solution for you.
        I'm connecting you with {{sales_rep}} who specializes in {{product_area}}.
      not_qualified: |
        Thanks for your time! Based on what you've shared, our enterprise solution might
        not be the best fit right now. I'd recommend checking out our self-serve plan at
        https://acme.example.com/plans.

  orchestration:
    pattern: react
    max_iterations: 10
    timeout_seconds: 30

  tools:
    - type: internal
      name: score_lead
      description: "Score a lead from 0-100 based on qualification criteria"
      parameters:
        type: object
        properties:
          company_size:
            type: string
            enum: [startup, smb, mid_market, enterprise]
            description: "Size category of the prospect's company"
          budget_range:
            type: string
            description: "Stated or inferred budget range (e.g., '$5k-$20k/month')"
          timeline:
            type: string
            enum: [immediate, this_quarter, this_year, exploring]
            description: "Purchase timeline"
          use_case:
            type: string
            description: "Primary use case or pain point"
          decision_maker:
            type: boolean
            description: "Whether the contact is a decision maker"
        required: [company_size, timeline]
    - type: builtin
      name: http_request
      config:
        base_url: "https://crm.acme.example.com/api"
        headers:
          Authorization: "Bearer ${CRM_API_TOKEN}"
      rate_limit:
        max_calls: 5
        period_seconds: 60
    - type: agent
      name: lookup_pricing
      agent: pricing-agent
      description: "Look up product pricing for a given tier and feature set"

  memory:
    conversational:
      backend: redis
      strategy: sliding_window
      max_turns: 50
      ttl: 86400
    semantic:
      backend: chromadb
      similarity_threshold: 0.75
      max_results: 10
    episodic:
      backend: sqlite

  guardrails:
    input:
      - type: pii_detection
        action: redact
      - type: max_length
        max_chars: 4096
      - type: topic_filter
        blocked_topics: [politics, religion, competitors]
        action: warn
    output:
      - type: content_filter
        blocked_patterns:
          - "(?i)\\b(kill yourself|kys)\\b"
        action: block
      - type: cost_limit
        max_tokens_per_turn: 1024

  permissions:
    allowed_actions: [respond, use_tools]
    filesystem:
      read: []
      write: []
    network:
      allowed:
        - "crm.acme.example.com"
        - "api.openai.com"
    execution:
      dry_run: false
```
