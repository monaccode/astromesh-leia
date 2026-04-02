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
| `provider` | string | REQUIRED | -- | LLM provider. One of: `ollama`, `openai`, `openai_compat`, `azure_openai`. |
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
| `max_tokens` | integer | `2048` | Maximum tokens in the response. |

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

### spec.model.routing (optional)

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `strategy` | string | optional | `"quality_first"` | Routing strategy between primary and fallback. One of: `cost_optimized`, `latency_optimized`, `quality_first`, `capability_match`. |
| `health_check_interval` | integer | optional | `30` | Seconds between health check pings to each model endpoint. |

**Strategy descriptions:**
- `cost_optimized` -- prefer the cheaper model when both are healthy.
- `latency_optimized` -- prefer the model with lower observed latency.
- `quality_first` -- always prefer primary; use fallback only on failure.
- `capability_match` -- route based on task complexity (requires orchestration metadata).

---

## spec.prompts (optional)

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `system` | string | optional | `""` | System prompt. Supports Jinja2 template syntax with variables: `{{agent_name}}`, `{{current_date}}`, `{{tools_list}}`, `{{context}}`. |
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

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `backend` | string | optional | `"in_memory"` | Storage backend. One of: `in_memory`, `sqlite`, `redis`. |
| `strategy` | string | optional | `"sliding_window"` | Memory management strategy. One of: `sliding_window`, `summary`, `token_budget`. |
| `max_turns` | integer | optional | `50` | Maximum conversation turns to retain (for `sliding_window`). |
| `ttl` | integer | optional | `86400` | Time-to-live in seconds for memory entries. Default is 24 hours. |

### spec.memory.semantic (optional)

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `backend` | string | optional | `"chromadb"` | Vector store backend. One of: `chromadb`, `pinecone`. |
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
| `pii_detection` | input, output | `action` (mask, block, warn) | Detects personally identifiable information. |
| `max_length` | input, output | `max_chars` (integer) | Rejects messages exceeding the character limit. |
| `topic_filter` | input | `blocked_topics` (string array), `action` (block, warn) | Blocks or warns on off-topic conversations. |
| `cost_limit` | output | `max_cost_per_request` (number), `currency` (string) | Limits per-request LLM cost. |
| `content_filter` | output | `categories` (string array), `action` (block, warn) | Filters harmful or inappropriate content. |

**Example:**
```yaml
guardrails:
  input:
    - type: pii_detection
      action: mask
    - type: max_length
      max_chars: 4096
    - type: topic_filter
      blocked_topics: [politics, religion]
      action: warn
  output:
    - type: content_filter
      categories: [hate_speech, self_harm]
      action: block
    - type: cost_limit
      max_cost_per_request: 0.10
      currency: USD
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
        action: mask
      - type: max_length
        max_chars: 4096
      - type: topic_filter
        blocked_topics: [politics, religion, competitors]
        action: warn
    output:
      - type: content_filter
        categories: [hate_speech, self_harm, explicit]
        action: block
      - type: cost_limit
        max_cost_per_request: 0.10
        currency: USD

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
