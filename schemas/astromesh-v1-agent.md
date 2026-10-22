# astromesh/v1 Agent YAML Schema Reference

This document is the authoritative schema reference for `astromesh/v1 Agent` manifests. Subagents (architect, operator, doctor) MUST consult this file when generating or validating agent YAML.

**Synced to astromesh core v0.74.1.** Two sources of truth are cited throughout, and they do not always agree:

- **The editor schema** — `vscode-extension/schemas/agent.schema.json` in the core repo. Validators (VS Code, CI linters) use it.
- **What the runtime loads** — `AgentRuntime._build_agent` in `astromesh/runtime/engine.py`. This is what actually happens on a node.

Where the runtime reads something the editor schema does not declare (or rejects), this file says so explicitly. When the two disagree, **the runtime wins** for "will it work"; the editor schema only decides whether a linter shows a red squiggle. Version gates are written as "core v0.NN+" — a node older than that ignores the field silently unless stated otherwise.

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
| `name` | string | REQUIRED | -- | Unique agent identifier, kebab-case. Editor-schema regex: `^[a-z][a-z0-9-]*[a-z0-9]$` — must **start with a letter** (not a digit), end with a letter or digit, at least 2 characters. Keep it ≤ 63 characters so it is also a valid RFC 1123 label for Kubernetes. |
| `version` | string | optional | `"0.1.0"` | Semantic version of the agent definition. Editor-schema regex: `^[0-9]+\.[0-9]+\.[0-9]+$` (quote it: `"1.0.0"`). |
| `namespace` | string | optional | `"default"` | Logical namespace (regex `^[a-z][a-z0-9-]*$`). Informational for the runtime; on Nexus/Kubernetes it must already exist in the cluster. |
| `labels` | object | optional | `{}` | Key-value string pairs for organizing and selecting agents. Keys and values must be strings. |

**Validation rules:**
- `name` is rejected if it contains uppercase letters, underscores, dots, or starts/ends with a hyphen.
- `namespace` must be a valid Kubernetes namespace (same RFC 1123 rules as `name`).
- `labels` keys must match: `([a-z0-9A-Z][-a-z0-9A-Z_.]*)?[a-z0-9A-Z]`

---

## spec.identity (optional, recommended)

Human-readable metadata about the agent. The editor schema requires nothing inside `spec` (no `required` list), and the runtime reads `spec.identity` with a default. Always emit it anyway — `description` is what other agents see when this one is used as a `type: agent` tool.

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `display_name` | string | optional | value of `metadata.name` | Human-friendly name shown in dashboards and logs. |
| `description` | string | optional | `""` | One-line description of what the agent does. Used in agent-to-agent tool descriptions. |
| `avatar` | string | optional | `""` | URL or emoji identifier for the agent avatar. |

---

## spec.model (required in practice)

Configures which LLM(s) the agent uses. The editor schema does not mark it required, but an agent with no model candidate builds a `default` router with zero providers (`engine.py:801-805`) and cannot answer. Always emit one.

### spec.model.primary (the legacy single-router shape)

A **model block** — the same shape is used by `primary`, `fallback`, each `extra` entry and each per-role `candidates` entry (editor schema `$defs/modelConfig`, `additionalProperties: false`, so an unknown key is a schema error).

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `source` | string | optional | inferred | Adapter that serves the model — **preferred over `provider`** in new agents. One of `ollama`, `openai`, `openai_compat`, `azure_openai`, `litellm`, `centinela`. Omitted, it is inferred: `litellm` when `model` contains `/`, otherwise `openai_compat`. |
| `provider` | string | optional | -- | Legacy alias of `source`, same values. In the legacy `primary`/`fallback`/`extra` shape a block with **neither** `provider` nor `source` is dropped (`engine.py:699-709`) — always set one of the two there. |
| `providerRef` | string | optional | -- | Name of an entry in the node's provider registry; the registry supplies `source`/`endpoint`/`api_key`/`contract`, and keys set in the block override it. Accepted by the editor schema since core v0.36.0. |
| `model` | string | REQUIRED | -- | Model identifier (e.g., `llama3.1:8b`, `gpt-4o`, `qwen2.5:7b`). For `litellm`, the provider-prefixed form (`anthropic/claude-opus-4-8`). |
| `endpoint` | string (URI) | optional | source-dependent | API endpoint URL. Defaults: `http://localhost:11434` for `ollama`. Required for `openai_compat` and `azure_openai`. Ignored by `litellm` (routes on the model prefix). |
| `endpoint_name` | string | optional | -- | Named endpoint — `centinela` source only. |
| `api_key` | string | optional | -- | API key as a literal string. Avoid committing secrets; prefer `api_key_env`. |
| `api_key_env` | string | optional | -- | Environment variable name containing the API key. |
| `contract`, `invalid_policy`, `max_retries` | object / string / integer | optional | -- | Label contract, what to do with a response that violates it, and retries — `centinela` source only. |
| `timeout` | number | optional | `120` | HTTP client timeout in seconds. Raise it for reasoning models with a slow time-to-first-token. |
| `temperature`, `max_tokens` | number / integer | optional | -- | Top-level shorthand for `parameters.temperature` / `parameters.max_tokens`. |
| `context_window` | integer | optional | detected | **core v0.64.0+.** Input tokens the model accepts; sizes the conversational-history budget (90% of the window minus system prompt, tool schemas, `prompts.context` and response `max_tokens`). Without it: Ollama's `parameters.num_ctx`, then litellm's model table, else unknown — and with an unknown window history is trimmed only by `max_turns`, with a load-time warning (`engine.py:907-914`). With several candidates the smallest window wins. `parameters.num_ctx` is read but is not in the editor schema's `parameters` list, so prefer `context_window`. Consumed by the runtime for **every** source, not passed to the provider (`engine.py:310-312`). |
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
- `source`/`provider` must be one of the six values above. `vllm`, `llamacpp`, `huggingface`, `hf_tgi` and `onnx` are **not** in the 0.68 editor schema — reach a vLLM / llama.cpp / TGI server with `openai_compat` + its `endpoint`.
- `model` must be a non-empty string.
- For `openai_compat` and `azure_openai`, `endpoint` is effectively required (no default).
- `api_key` and `api_key_env` are mutually exclusive; specifying both is an error.
- `timeout` must be a positive number.
- `temperature` must be between 0.0 and 2.0 inclusive.
- `top_p` must be between 0.0 and 1.0 inclusive.
- `max_tokens` must be a positive integer.

### spec.model — Moonshot / Kimi (via `openai_compat`) — astromesh v0.28.6+

Moonshot's **Kimi** models are reached through the `openai_compat` provider (there is no dedicated `kimi`/`moonshot` provider value). Point `endpoint` at Moonshot's OpenAI-compatible base URL and pass the key via `api_key_env`:

| Field | Value |
|-------|-------|
| `provider` | `openai_compat` |
| `endpoint` | `https://api.moonshot.ai/v1` |
| `api_key_env` | `MOONSHOT_API_KEY` |
| `model` | `kimi-k2.5` or `kimi-k2.6` |

```yaml
model:
  primary:
    provider: openai_compat
    model: kimi-k2.6
    endpoint: https://api.moonshot.ai/v1
    api_key_env: MOONSHOT_API_KEY
```

**Thinking models (`reasoning_content`).** `kimi-k2.5` and `kimi-k2.6` are reasoning models: they emit their chain-of-thought in a separate `reasoning_content` field. The runtime handles this automatically — it preserves `reasoning_content` on assistant tool-call turns, so Kimi works inside `react` (and the other tool-calling patterns) with no extra config. Dropping it triggers `400 Bad Request — thinking is enabled but reasoning_content is missing in assistant tool call message`, a client error the runtime already guards against.

**Cost & cache-aware pricing (astromesh v0.28.8–v0.28.9).** Kimi pricing is built into the router's cost estimation, so `cost_optimized` ranks Kimi correctly against OpenAI/Ollama slots. Moonshot's context cache is priced too: cache-read input tokens bill at a lower rate and surface as `cache_read_input_tokens` in `response.usage`. Costs are attributed under a derived provider label (`kimi` for kimi/moonshot models, `anthropic` for claude, `openai` for gpt/o-series, else `openai_compat`).

**Muse Spark Open via OpenRouter (astromesh v0.72.0+).** `model: meta/muse-spark-open`
with `provider: openai_compat`, `endpoint: https://openrouter.ai/api/v1` and an
`api_key_env` for the OpenRouter key. The provider must be explicit (the id carries a `/`).
The router prices it at 1.25 / 4.25 USD per 1M input/output tokens (cached input 0.15),
labelled `meta`; before 0.72.0 its cost was estimated as 0.

### spec.model.fallback (optional)

Same structure as `spec.model.primary`. Used when the primary model is unavailable or returns errors. The runtime tries the fallback after exhausting retries on the primary.

### spec.model.extra (optional) — added in astromesh v0.28.0

A map of **additional named providers** registered alongside `primary` and `fallback`. Every registered slot (primary + fallback + each `extra` entry) is ranked together by the configured routing `strategy` — so `extra` lets one agent fan out across more than two models without being limited to the primary/fallback pair.

| Field | Type | Description |
|-------|------|-------------|
| `<slot_name>` | model block | Each key is an arbitrary slot name (e.g. `cheap`, `vision`, `local`); each value is a full provider block with the same shape as `primary`. |

**Validation rules (core v0.68.0, `engine.py:697-711`):**
- `extra` must be a mapping (object); anything else is ignored.
- Each entry joins the `default` role after `primary` and `fallback`, in declaration order. An entry with neither `provider` nor `source` is **dropped silently** — always set one.

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
- `cost_optimized` *(default)* -- rank by each provider's estimated cost; prefer the cheapest healthy model. Cache-aware for providers that expose cached tokens (e.g. Kimi's context cache — see the Moonshot/Kimi note above), so cache-read input is priced at its lower rate.
- `latency_optimized` -- rank by observed average latency; prefer the fastest healthy model.
- `round_robin` -- rotate requests across all healthy models in turn.
- `capability_match` -- filter to models that satisfy required capabilities (e.g. tools, vision) for the task.
- `quality_first` -- recognized, but currently a no-op ranking: slots keep their declared order (primary first), so it behaves like "prefer primary, fall back on failure".

> The router has a circuit breaker: 3 consecutive failures on a provider open it for a 60s cooldown (then half-open retry).

### spec.model per-role selection (`default` / `roles`) — astromesh v0.29.0+

Everything above (`primary` / `fallback` / `extra` / `routing`) configures **one** router shared by the whole agent. As of core **v0.29.0** an agent can instead bind a distinct model — and source — to **each orchestration role**, so a `plan_and_execute` agent can plan with a frontier cloud model and execute with a cheap local one. Use this shape **instead of** `primary`/`fallback`/`extra` (do not mix the two shapes in one `model` block):

```yaml
spec:
  model:
    default:                 # REQUIRED with per-role: the router every unconfigured role falls back to
      candidates:
        - {source: ollama, model: "llama3.1:8b", endpoint: "http://localhost:11434"}
      strategy: cost_optimized
    roles:                   # optional: role name -> its own router
      planner:
        candidates:
          - {source: litellm, model: "anthropic/claude-opus-4-8", api_key_env: ANTHROPIC_API_KEY}
        strategy: quality_first
      worker:
        candidates:
          - {source: ollama, model: "llama3.1:8b"}
        strategy: cost_optimized
```

- `default` is REQUIRED whenever this shape is used — it is the `ModelRouter` every unrecognized or unconfigured role falls back to. Each role in `roles` becomes its own independent router (its own circuit breaker), ranked by its own `strategy` (same strategy values as `spec.model.routing`).

**Candidate fields** (each entry in a `candidates` array): exactly the model block documented under `spec.model.primary` above — `source`/`provider`, `providerRef`, `model`, `endpoint`, `api_key_env`, `timeout`, `temperature`/`max_tokens`, `context_window`, `parameters`, and the `centinela`-only keys. `litellm` needs the runtime's optional `litellm` extra. Each role block takes only `candidates` and `strategy` (editor schema `$defs/roleConfig`); `strategy` defaults to `cost_optimized` (`engine.py:682-695`). The history budget (`context_window`) is computed from the **`default`** role's candidates (`engine.py:903-907`).

#### Which keys each source consumes (astromesh core v0.36.0+)

`parameters` and `timeout` are honored by **all** wired sources — until v0.36.0
several branches accepted them in the schema and then silently dropped them, so
a `temperature` or `timeout` set in YAML could have no effect. That is fixed:

| Key | `ollama` | `openai_compat` | `litellm` |
|-----|----------|-----------------|-----------|
| `parameters.temperature` / `top_p` / `max_tokens` | ✅ (routed into ollama's nested `options`; `max_tokens`→`num_predict`) | ✅ | ✅ |
| `parameters.presence_penalty` / `frequency_penalty` | ⚠️ warns — not on ollama's native surface | ✅ | ✅ |
| `timeout` | ✅ | ✅ (fixed v0.35.1) | ✅ |
| `endpoint` | ✅ | ✅ | ⚠️ ignored — litellm routes on the model prefix |

A model block that declares a key its source ignores now logs a `WARNING`
naming the source and the key, instead of dropping it in silence. Declare only
what the chosen source consumes.

If a `source: litellm` candidate is configured but the `litellm` package isn't installed on the node, the runtime skips **only that candidate** and logs a warning — startup does not fail.

**Role vocabulary** — each pattern requests these roles; any role not defined under `roles` falls back to `default`:

| Pattern | Roles requested |
|---------|------------------|
| `react` | `reasoner` |
| `plan_and_execute` | `planner`, `worker`, `synthesizer` |
| `parallel_fan_out` | `planner`, `worker`, `synthesizer` |
| `pipeline` | `stage:<name>` per configured stage (default stages: `analyze`, `process`, `synthesize`) |
| `supervisor` | `supervisor` |
| `swarm` | `reasoner` |
| *(memory)* `conversational.strategy: summary` | `summarizer` — falls back to `default` (`engine.py:344-346`) |

**Remapping** — `spec.orchestration.role_map` points a pattern's built-in role at one of your own role names without renaming it in `roles`:

```yaml
spec:
  orchestration:
    pattern: react
    role_map:
      reasoner: planner   # ReAct's "reasoner" requests route to the "planner" router
```

Resolution order for a requested role: `role_map[role]` (if remapped) → resolved name looked up in `spec.model.roles` → `default`.

> `role_map` is read by the runtime (`engine.py:1577`, core v0.29.0+) but is **not declared** in the 0.68 editor schema, whose `orchestration` block has `additionalProperties: false` — a JSON-schema validator will flag it even though it works. Emit it only when the remap is genuinely needed, and say so.

**Backward compatibility:** the legacy `primary` / `fallback` / `extra` / `routing` shape still works unchanged — internally it is normalized into a single `default` role. No migration needed; adopt `default`/`roles` only where you want per-role selection. **Per-role selection requires the deployed runtime at core `v0.29.0+`**; on older nodes emit the legacy `primary`/`fallback` shape instead.

---

## spec.prompts (optional)

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `system` | string | optional | `""` | System prompt, a Jinja2 template (sandboxed since core v0.59.1). Keep it **stable between turns** — see *Prompt caching* below. |
| `context` | string | optional | `""` | **core v0.64.0+.** Jinja2 template for what changes on every query (RAG `knowledge`, `prefetch`). Rendered per run and placed in front of the **current user message**, so system → tools → history stay a byte-identical, cacheable prefix. The turn stored in memory is the original query, without the context. |
| `templates` | object | optional | `{}` | Named Jinja2 templates registered per agent (`engine.py:1232-1233`). Not in the 0.68 editor schema (`prompts` has `additionalProperties: false`, so a validator flags it). No orchestration pattern reads them — do **not** use them to define pipeline stages. Rarely worth emitting. |

### Template variables

What the runtime passes to `prompts.system` and `prompts.context` (`engine.py:1733-1739`):

| Variable | Contents |
|---|---|
| *caller context keys* | Whatever the caller put in the run's `context`, at top level. On WhatsApp channels: `contact_name` (sender display name, core v0.27.0+). Integrations such as PRAXIS also look at `sender_phone`. Keys starting with `_` are reserved for the runtime. |
| `memory` | `memory.conversation` (history), `memory.semantic`, `memory.episodic`. |
| `knowledge` | Chunks retrieved from `spec.knowledge.pipeline` for this query. Per-query → belongs in `prompts.context`. |
| `prefetch` | `{name: {success, data, metadata, error}}` from `spec.prefetch` (core v0.51.0+). Per-query → belongs in `prompts.context`. |
| `bundle_index` | **core v0.68.0+.** The root index of `spec.knowledge.bundle` (empty string without a bundle, or if the index cannot be read). **Stable between queries** while the bundle does not change → belongs in `prompts.system`. |

There are **no** built-in `agent_name`, `current_date` or `tools_list` variables: the runtime does not set them and undefined variables render as an empty string (`SilentUndefined`), so `Today is {{current_date}}` renders as `Today is `. Write the agent's name literally, and pass dates through the caller context if needed. Guard optional vars: `{% if contact_name %}Address the user as {{ contact_name }}.{% endif %}`. Guard `prefetch` with `prefetch is defined` if the agent may run on a node older than v0.51.0 — reading an attribute of an undefined variable is not silenced.

### Prompt caching (core v0.64.0+)

The provider's prompt cache only helps if everything before the last message is identical between calls. The runtime enforces the layout `system → tools → history → [glyph grammar] → context + query` and warns at load time when:

- `prompts.system` uses `knowledge`, `prefetch`, `memory.semantic` or `memory.episodic` inside a Jinja block (`{{ }}` / `{% %}`) — move it to `prompts.context` (`engine.py:1219-1225`);
- `prompts.system` uses `memory.conversation` — the legacy path still works (history is rendered into the system prompt, no messages are sent) but breaks the cache; drop it, history now travels as messages (`engine.py:920-925`);

Do not iterate `memory.conversation` inside `prompts.context` either: the history already travels as messages.

**Example:**
```yaml
prompts:
  system: |
    You are Lucia, the sales analyst for Acme. Answer only with the data provided.
    {% if bundle_index %}Knowledge map (open concepts with kb_read):
    {{ bundle_index }}{% endif %}
  context: |
    {% if knowledge %}RELEVANT DOCUMENTS:
    {{ knowledge }}{% endif %}
    {% if prefetch is defined and prefetch.stock and prefetch.stock.success %}CURRENT STOCK: {{ prefetch.stock.data }}{% endif %}
```

---

## spec.orchestration (optional)

Controls how the agent processes tasks and uses tools.

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `pattern` | string | optional | `"react"` | Orchestration pattern. One of: `react`, `plan_and_execute`, `parallel_fan_out`, `pipeline`, `supervisor`, `swarm`, `glyph` (core v0.39.0+). See `schemas/orchestration-patterns.md`. |
| `max_iterations` | integer | optional | `10` | Maximum reasoning/tool-call loops. In `plan_and_execute`, `pipeline`, `parallel_fan_out` it bounds **each** step/stage/subtask's tool loop, not the whole run. |
| `timeout_seconds` | number > 0 | optional | -- | **Applied since core 0.71.0.** Cuts the whole run: HTTP **504** `{"error": "timeout", "agent", "timeout_seconds"}` on REST and `POST /v1/workflows/{name}/run`, WebSocket `error` event with `error: "timeout"`; the run leaves a `timeout` episode, an in-flight tool closes as `{ok: false, cancelled: true}`. A value <= 0, a bool or a non-number is an `AgentConfigError`. **Use >= 120 or omit it**: the provider's HTTP timeout is 120 s and values of 30-90 s cut runs with tools or slow models. `timeout` is its deprecated alias. The model block's `timeout` is still the per-call limit. |
| `max_handoffs` | integer | optional | `3` | **`pattern: swarm` only (core 0.71.0+).** Integer >= 1; caps chained handoffs. The runtime carries the counter to the child agent (every sub-agent inherits it); at the cap handoffs are no longer offered or executed. |
| `max_tool_result_tokens` | integer | optional | `8000` | **core v0.66.0+.** Default cap, in tokens, on each tool result entering the conversation. Overridden per tool by `tools[].max_result_tokens`. A value that is not an integer > 0 warns at load and uses the default (`engine.py:1235-1246`). See *Tool result budget* below. |
| `stages` | string array | optional | `[analyze, process, synthesize]` | **core v0.65.0+, `pattern: pipeline` only.** 2 to 6 stage names of 1–40 characters. Each name also becomes the model role `stage:<name>`. Anything else out of range → the agent does not load (`engine.py:1366-1376`). |
| `narrate` | boolean | optional | `true` | **`pattern: glyph` only.** `false` returns the program's result as JSON and skips the second model call that turns it into prose — use it when the agent feeds another agent, not a person. |
| `max_repairs` | integer | optional | `2` | **`pattern: glyph` only.** Retries when the model-written program does not compile. Ignored with `spec.program`. |
| `role_map` | object | optional | -- | Remap a pattern's built-in role to one of your role names (see *Remapping* above). Read by the runtime, absent from the editor schema. |

**Validation rules:**
- `pattern` must be one of the seven values. Since core v0.65.0 an unknown pattern is an `AgentConfigError` — **the agent does not load** (before, it fell back to `react` silently). An explicit `pattern: null` is `react`.
- `pattern: supervisor` needs at least one `type: agent` tool (its workers) or the agent does not load (core v0.65.0+, `engine.py:1354-1365`).
- `max_iterations` and `max_tool_result_tokens` must be integers ≥ 1.

---

## spec.program (optional) — astromesh core v0.39.0+

A fixed **Glyph** program, written ahead of time. Only valid with
`orchestration.pattern: glyph`. The runtime compiles it against the agent's
`tools` when the agent loads, then runs it on every request **without calling
the model** to plan (the model is called once more to narrate the result unless
`orchestration.narrate: false`).

```yaml
spec:
  orchestration:
    pattern: glyph
    narrate: false
  program: |
    customer = find_customer(email=context.email)
    devices  = list_devices(customer_id=customer.id)
    return {customer, devices}
```

| Situation | Result |
|---|---|
| `program` set and `pattern` is not `glyph` | **Agent does not load** (`engine.py:1302-1306`) |
| `program` does not compile (unknown tool, syntax) | **Agent does not load**, with line and message |
| `pattern: glyph` but the node lacks the `glyph` extra, no `program` | Falls back to `react` with a warning |
| Same, **with** `program` | **Agent does not load** — a fixed program must not silently become a full ReAct run (`engine.py:1312-1326`) |

**Read before choosing Glyph.** As a runtime pattern (the model writing the
program every run) Glyph was measured at +164% to +2839% the cost of ReAct.
It pays off only with `spec.program`: written once, reviewed by a human, run
forever with zero planning calls. The `astromesh-glyph` package is not on PyPI
yet, so most nodes do not have the extra — confirm before emitting it.

---

## spec.output_schema (optional) — astromesh core v0.38.1+

Declares the shape of the agent's structured output. The runtime asks the model
for it through the prompt, parses it out of the answer, validates it, and places
it in `result.data` alongside the prose `answer`.

Two shapes are accepted, and both mean the same thing:

```yaml
# shorthand — the same one tool `parameters` use
output_schema:
  score:  {type: integer}
  urgent: {type: boolean}

# or full JSON Schema, kept verbatim
output_schema:
  type: object
  properties:
    score:  {type: integer}
    urgent: {type: boolean}
  required: [score]
```

**What it changes in the run response:** `answer` is left untouched, prose and
all — nothing that reads it today breaks. Two keys are *added*: `data` (the
validated object, or `null`) and `data_error` (a string describing why parsing
or validation failed, or `null`). An agent without `output_schema` gets neither
key.

**A validation failure does not abort the run.** `data` becomes `null`,
`data_error` carries the detail, and the answer is returned normally. Retrying
the model because it emitted bad JSON is not implemented.

**Validator coverage.** The runtime uses a purpose-built validator with no
external dependency, so it understands a documented subset:

| Supported | Ignored silently |
|---|---|
| `type` (`object`, `string`, `integer`, `number`, `boolean`, `array`, `null`), `properties`, `required`, `enum`, `items` | `allOf`, `anyOf`, `oneOf`, `not`, `$ref`, `patternProperties`, `additionalProperties`, `format`, `minimum`, `maximum`, `minLength`, `maxLength`, `pattern`, `minItems`, `maxItems`, `uniqueItems` |

Do not author an `output_schema` whose correctness depends on an ignored
keyword — it will validate as if the keyword were not there. One rule that *is*
enforced: `true` does not pass as an `integer`.

**Why it matters:** it is the prerequisite for trustworthy `spec.chain`
conditions. Without it, a `when` can only match on free prose.

---

## spec.chain (optional) — astromesh core v0.38.1+

Declares which other agents to fire when this one finishes, and under which
conditions. The runtime compiles the declaration into a workflow when it boots.

**`chain` is NOT an orchestration pattern.** `spec.orchestration.pattern` still
takes one of the seven pattern names and `chain` is not among them. This is a
separate, top-level section.

```yaml
spec:
  output_schema:
    score: {type: integer}

  chain:
    mode: sequential        # sequential | parallel   (default: sequential)
    max_depth: 5            # default: 5

    on_complete:
      - agent: email-composer
        when: "{{ output.data.score > 7 }}"
        input: "{{ output.answer }}"        # default: "{{ output.answer }}"
        retry:
          max_attempts: 3
          backoff: exponential              # fixed | exponential
          initial_delay_seconds: 2
        timeout_seconds: 30
        on_error: continue                  # stop (default) | continue | <agent>

      - agent: crm-logger                   # no `when` = always fires

      - agent: human-triage
        default: true                       # only if no `when` matched
```

### Fields

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `mode` | string | optional | `sequential` | `sequential` runs the links one after another; `parallel` runs them at once. |
| `max_depth` | integer | optional | `5` | How deep chains may nest before the runtime refuses to start. |
| `on_complete` | array | REQUIRED | — | The links. Must not be empty. |

Each entry of `on_complete`:

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `agent` | string | REQUIRED | — | Name of the agent to fire. It must exist, or the runtime will not start. |
| `when` | string | optional | — | Jinja condition. Fires when it renders `true`, `1` or `yes`. Omit it to fire always. |
| `input` | string | optional | `{{ output.answer }}` | What the link receives. |
| `default` | boolean | optional | `false` | Fires only if no `when` in this chain matched. Cannot be combined with `when`. |
| `retry` | object | optional | — | `max_attempts`, `backoff` (`fixed`/`exponential`), `initial_delay_seconds`. |
| `timeout_seconds` | integer | optional | — | Wall-clock ceiling for this link. |
| `on_error` | string | optional | stop | `continue` records the failure and carries on; an agent name jumps there; omitted stops the chain. |

### Rules that are easy to get wrong

- **Every matching rule fires, not just the first.** This is not an if/elif
  cascade. With `score = 2` in the example above, `email-composer` does not
  fire, `crm-logger` does (it has no `when`), and `human-triage` does too —
  because no `when` matched.
- **Rules without `when` do not count as a match** for the purposes of `default`.
- **At most one `default` rule**, and it cannot carry a `when`.
- **Under `mode: parallel`, every guard is evaluated before any link starts**, so
  a `when` referencing a sibling would always be false. The runtime rejects that
  at startup instead of letting it fail quietly — use `sequential` when one link
  needs to see another's result.

### Context available in `when` and `input`

| Variable | Contents |
|---|---|
| `output.answer` | Prose answer of the immediately preceding agent |
| `output.data` | Its validated `output_schema` object, or `None` |
| `output.steps` | Its orchestration steps |
| `trigger` | Original payload: `query`, `session_id`, `context` |
| `steps` | Outputs of links already executed, by name |
| `when` | Boolean result of each guard already evaluated, by step name |

`output` always refers to the **immediately preceding agent**, not the one that
started the chain. In `A → B → C`, the `when` in B's chain sees B's output.

### Failure modes to design around

**`spec.chain` and `confirm` cannot coexist.** If any tool of the agent declares
`confirm`, a `spec.chain` on the same agent is a `ValueError` and **the agent does
not load** (`engine.py:872-878`): a chain step runs as non-human and can never
confirm, so the pending proposal would wait forever. Split the confirming agent
from the chained one.

Authoring mistakes are caught **when the runtime starts**, not mid-run: a cycle,
an exceeded `max_depth`, or a link naming an agent that does not exist all
prevent the node from booting, with the offending path in the message. A chain
that references an agent you have not deployed yet will take the whole node
down — deploy the targets first.

Conditions in a chain are evaluated strictly: a `when` that references a field
that does not exist fails that link with an explicit error instead of quietly
evaluating to false.

### What the run response gains

`POST /v1/agents/{name}/run` adds a `chain` block next to the usual keys. The
`answer` stays the **invoked agent's** answer, so existing consumers are
unaffected; the links appear alongside it.

```json
{
  "answer": "Lead qualified 8/10...",
  "data": {"score": 8},
  "chain": {
    "run_id": "wf-9c2...",
    "status": "completed",
    "mode": "sequential",
    "links": [
      {"agent": "email-composer", "depth": 1, "via": null,
       "status": "success", "answer": "Sent to ana@acme.com"}
    ]
  }
}
```

`chain.status` is `completed`, `partial` (errors but the chain continued) or
`failed`. Each link is `success`, `error` (with `error`) or `skipped` (with
`reason`: `condition_false`, `cycle`, `max_depth`, `upstream_stopped`). Links
that did not fire are listed too, so "why wasn't the email sent?" is answerable
from the response alone.

`GET /v1/agents/{name}/chain` returns the expanded graph without running
anything — useful to confirm a deployed chain resolves the way you expect.

### Runtime version gate — read before emitting a chain

`spec.chain` and `spec.output_schema` need the node at **astromesh core
v0.38.1+**. Older runtimes do **not** error on them: unknown `spec` keys are
ignored, so the agent deploys, answers normally, and simply never chains —
silently.

That matters for Nexus specifically. Nexus's spec validation is permissive and
accepts `spec.chain` without complaint, whatever core version its managed
runtime pool runs. A chain published to a pool older than v0.38.1 is stored,
injected, and ignored. Confirm the target node's version (`GET /v1/health`)
before promising a user that chaining will happen. The same rule applies to
every version-gated field in this file.

---

## spec.tools (optional)

Array of tool definitions. Each tool must have `name` and `type`. **Seven** tool
types load from an agent YAML in core v0.74.1 (`_build_agent` in `engine.py`):

| `type` | Since | What it is |
|---|---|---|
| `builtin` | — | A tool shipped with the runtime (`web_search`, `http_request`, …). |
| `agent` | — | Another agent, callable as a tool. |
| `client` | v0.35.0 | Announced to the model, executed by whoever is listening. |
| `integration` | v0.37.0 | Actions of a catalog integration (`praxis`, `gmail`, `google_sheets`, …). |
| `api` | v0.57.0 | A tenant's own HTTP API, declared inline. |
| `mcp` | v0.58.0 | A tenant's MCP server, with a snapshot of its tools inline. |
| `a2a` | v0.74.0 | A remote agent that speaks A2A 1.0, called through its Agent Card. Needs the `a2a` extra. |

Any other `type` (`internal` — also the default when `type` is omitted —,
`mcp_stdio`/`mcp_sse`/`mcp_http`, `webhook`, `rag`) is **not loadable**: the
runtime logs a `WARNING` naming the agent, the tool and the unsupported type,
and skips the tool (never registered, never reaches the model). From core 1.0
an unsupported type becomes a hard error. Do not author agents with those types.

> **Editor schema.** Since core 0.73.0 `agent.schema.json` enumerates
> `integration`, `api` and `mcp` for `tools[].type`, and since 0.74.0 `a2a`
> (with its keys). Before 0.73 a JSON-schema validator flagged those types even
> though the runtime loaded them. It still does not declare `spec.prefetch` or
> `orchestration.role_map`: mention the lint warning if the user validates in VS Code.

### Keys each type reads — everything else is ignored with a WARNING

Since core v0.43.0 a key the runtime does not read for that type is reported at
load time ("declara la tool … con claves que este runtime no lee") and ignored
(`engine.py:189-228`). Common to every type: `type`, `name`, `confirm`,
`max_result_tokens`.

| `type` | Type-specific keys |
|---|---|
| `builtin` | `config`, `rate_limit` |
| `agent` | `agent`, `description`, `parameters`, `context_transform` |
| `client` | `description`, `parameters`, `rate_limit` |
| `integration` | `connection`, `actions`, `rate_limit` (no `description` — the integration manifest's is used) |
| `api` | `connection`, `description`, `auth`, `operations`, `rate_limit` |
| `mcp` | `connection`, `path`, `auth`, `tools`, `rate_limit` |
| `a2a` | `card_url`, `auth`, `policy`, `description`, `rate_limit` |

So `rate_limit` on an `agent` tool, `description` on an `integration`, and
`requires_approval` / `timeout_seconds` on any tool do nothing.

### Common keys

| Field | Type | Applies to | Description |
|-------|------|-----------|-------------|
| `max_result_tokens` | integer | all types | **core v0.66.0+.** Cap, in tokens, on this tool's result when it enters the conversation. For `integration`, `api` and `mcp` it applies to every tool that definition registers. Falls back to `orchestration.max_tool_result_tokens`, then `8000`. |
| `confirm` | string array | `integration` only | **core v0.42.0+.** Actions that do not run until the person confirms in the next turn (see *Confirmation gate* below). On any other type it is ignored with a warning. |

### Type: builtin

Pre-packaged tools provided by the astromesh runtime.

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `type` | string | REQUIRED | -- | Must be `"builtin"`. |
| `name` | string | REQUIRED | -- | Built-in tool name (e.g., `web_search`, `calculator`, `datetime`, `http_request`). Its description and parameters come from the tool itself. |
| `config` | object | optional | `{}` | Tool-specific configuration. |
| `rate_limit` | object | optional | -- | Rate limiting: `{max_calls: int, window_seconds: int}` (defaults 10 and 60, `core/tools.py:409-422`; `period_seconds` is ignored). |

### Type: agent

Delegates to another deployed agent as a tool.

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `type` | string | REQUIRED | -- | Must be `"agent"`. |
| `name` | string | REQUIRED | -- | Tool name as exposed to the LLM. |
| `agent` | string | REQUIRED | -- | `metadata.name` of the target agent. Must be deployed in the same runtime. |
| `description` | string | optional | `Invoke agent '<agent>'` | Description for the model. Write a real one. |
| `parameters` | object | optional | -- | Input parameters. Accepts the shorthand `{param: {type, description}}` (normalized into JSON Schema) or a full JSON Schema object. |
| `context_transform` | string | optional | -- | Jinja2 template to transform context before passing to the sub-agent. |

Since core v0.61.0 a sub-agent that proposes (`mode: propose`) writes into its
parent's `propuestas`, tagged with `via`.

Since core v0.73.0 a sub-agent runs in its parent's trace, and its usage (every
model, tokens and cost, at any depth) is **added to the root run**: `usage` and
`usage.by_model` include it, and the root trace lists the children in
`subagentes`. Two consequences for the author:

- The parent's `cost_limit` guardrail counts what its sub-agents spent, so a
  supervisor or swarm needs a budget for the whole tree, not only its own turns.
- **Nexus bills it** (pool on core 0.73.0+, Nexus 0.27.1+): runs with sub-agents
  cost more than before, because the child usage used to be lost.

### Type: client (astromesh core v0.35.0+)

A tool the runtime **announces to the model but never executes**. The point of
the call is the call itself — "show this chart", "open this form" — and what it
means is the consumer's business, not the runtime's. When the model calls a
`client` tool the runtime returns `{"ok": true}` without running anything; the
call is delivered live to the consumer through the streaming `tool_call` /
`tool_result` events (see *Streaming contract* below) and recorded afterwards in
the run's `steps` (`action` / `action_input`). With nobody listening a `client`
tool is a silent no-op — that is correct, not a bug.

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `type` | string | REQUIRED | -- | Must be `"client"`. |
| `name` | string | REQUIRED | -- | Tool name as exposed to the LLM. |
| `description` | string | REQUIRED | -- | What the call means to the consumer. |
| `parameters` | object | optional | -- | Input parameters. Shorthand `{param: {type, description}}` is normalized into valid JSON Schema before it reaches the model — required, because a strict provider rejects the whole request on an invalid schema. A `parameters` that is not a mapping skips just this tool with a warning. |
| `rate_limit` | object | optional | -- | `{max_calls, window_seconds}`. |

### Type: integration (astromesh core v0.37.0+)

Exposes an allowlist of actions from an integration in the node's catalog
(`astromesh/integrations/catalog/`: `praxis*`, `gmail`, `google_calendar`,
`google_drive`, `google_sheets`, `http`, `whatsapp`, `facebook`, `instagram`,
`tiktok`, `conocimiento` in 0.68). Each action registers as a tool named
`<slug>_<action>`. Credentials come from the run (Nexus connection bundle) or
`config/connections.yaml`, never from the manifest.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | REQUIRED | `"integration"`. |
| `name` | string | REQUIRED | The catalog slug (e.g. `praxis`). Unknown slug → warning, tool skipped. |
| `connection` | string | REQUIRED | Connection name. Missing → warning, tool skipped. |
| `actions` | string array | REQUIRED | Allowlist of the integration's actions. Missing → warning, tool skipped (exposing every action inflates the prompt). An unknown action skips just that action. |
| `confirm` | string array | optional | Subset of `actions` gated by the confirmation gate (core v0.42.0+). |
| `rate_limit` | object | optional | Overrides the action's / integration's default. |

**Confirmation gate.** An action in `confirm` does not execute when the model
calls it: the runtime registers a pending proposal and replies with a sentence
**it** writes ("Para confirmar respondé SI. Voy a ejecutar: …"); only a human
"sí" in the next turn, for that exact tool **and** arguments, runs it. A
`confirm` naming something not in `actions`, or a `confirm` without `actions`,
is a `ValueError` — **the agent does not load** (`engine.py:1067-1089`). On a
node older than v0.42.0 `confirm` is ignored and the write runs ungated, so
verify the version before relying on it.

### Type: api (astromesh core v0.57.0+)

The tenant's own HTTP API, carried inline. Each operation becomes one tool named
`<slug with _>_<operation>`. Unlike `integration`, **any invalid declaration
raises and the agent does not load** (`integrations/api.py`).

```yaml
tools:
  - name: erp-cliente          # kebab slug
    type: api
    connection: erp_main       # supplies base_url + credential per run
    auth: { scheme: bearer }
    operations:
      - name: buscar_cliente
        description: Find a customer by tax id.
        writes: false
        parameters:
          cuit: { type: string, description: The customer's tax id., required: true }
        request:
          method: GET
          path: /clientes
          query: { cuit: "{cuit}" }
```

| Field | Required | Description |
|-------|----------|-------------|
| `name` | Yes | Kebab-case slug, `^[a-z][a-z0-9]*(-[a-z0-9]+)*$`. |
| `connection` | Yes | Connection that supplies `base_url` and the credential (there is no `base_url` in the manifest). |
| `auth.scheme` | Yes | `header` (+ `auth.header`), `bearer`, `basic`, or `query` (+ `auth.param`). |
| `operations` | Yes | Non-empty list. Each: `name` (snake_case), `description`, `parameters`, `request`, `response`, `writes`, optional `mode`. |
| `description` | No | Description of the API. |
| `rate_limit` | No | Applies to every operation. |

Per operation: `writes: false` (exactly) to read, or `writes: true` **with**
`mode: propose` (core v0.59.0+) — a proposing write is never executed, it is
returned to the caller in `propuestas`. Any other combination raises.
`request.method` is `GET` or `POST`; `request.path` starts with `/`;
`request.headers` is rejected (auth goes through `auth`); the final tool name
must be ≤ 64 characters. Calls go only to **public** hosts.

### Type: mcp (astromesh core v0.58.0+)

The tenant's MCP server with a **snapshot** of its tools inline — the runtime
does not discover. Each tool is registered as `<slug with _>_<tool name
normalized to [a-z0-9_]>`. Invalid declarations raise; unknown keys are rejected
(`integrations/mcp.py`). The node needs the `mcp` extra or the agent does not load.

```yaml
tools:
  - name: soporte
    type: mcp
    connection: soporte_mcp
    path: /mcp
    auth: { scheme: header, header: X-Api-Key }
    tools:
      - name: buscar_ticket
        description: Find a support ticket by number.
        writes: false
        input_schema:
          type: object
          properties: { numero: { type: string } }
          required: [numero]
```

| Field | Required | Description |
|-------|----------|-------------|
| `name` | Yes | Kebab-case slug, ≤ 30 characters. |
| `connection` | Yes | Supplies `base_url` and credential; the call goes to `base_url + path`. |
| `path` | Yes | Starts with `/`, no query or fragment. |
| `auth.scheme` | Yes | `bearer`, or `header` with `auth.header` (reserved headers such as `Host`, `Content-Type`, `Mcp-Session-Id` are rejected). |
| `tools` | Yes | At least one. Each: `name`, `description`, `input_schema` (`type: object`, ≤ 16 KB, ≤ 8 levels), `writes` (**no default** — must be declared), optional `mode: propose` (required when `writes: true`). |
| `rate_limit` | No | Applies to every tool of the server. |
| `max_result_tokens` | No | Accepted since core v0.66.0 (before, it rejected the server and left the agent in draft). |

**Name collisions (`api` and `mcp`).** If the final name of an `api` or `mcp`
tool matches any other tool of the agent — builtin, integration, another `api`
or `mcp`, or the bundle's `kb_*` tools — in whichever order, the agent does not
load (`engine.py:1124-1130, 1166-1174, 1210-1215`). The same holds for an
`a2a` tool (core v0.74.0+).

### Type: a2a (astromesh core v0.74.0+)

A remote agent that speaks [A2A 1.0](https://a2a-protocol.org), used as one
tool. It runs on `astromesh-a2a` (extra `astromesh[a2a]`, included in `all` and
in the `fulfarodev/astromesh` image). Without the extra the agent stays in
`draft` with "instalá astromesh[a2a]".

```yaml
tools:
  - name: agenda_externa
    type: a2a
    card_url: https://agents.example.com/.well-known/agent-card.json
    auth: { api_key_env: AGENDA_KEY }      # or { bearer_env: AGENDA_TOKEN }; optional
    policy: { timeout: 30, max_bytes: 262144, accept_parts: [text, data] }  # optional
    description: "Books and cancels appointments in the clinic's calendar."  # optional
```

| Field | Required | Description |
|-------|----------|-------------|
| `card_url` | Yes | URL of the remote Agent Card. **https only**; `http` only with `policy.allow_insecure_http: true` (local development). No credentials in the URL. Its host is the only host the tool may call. |
| `auth` | No | Exactly one of `api_key_env` (sent as `X-API-Key`) or `bearer_env` (`Authorization: Bearer`): the **name** of an environment variable on the node, never the secret. The library reads it on every request. |
| `policy.timeout` | No | Seconds per call, `(0, 600]`, default 60. Also the tool's timeout. |
| `policy.max_bytes` | No | Cap on every response body and part, 1 B – 16 MiB, default 1 MiB. |
| `policy.accept_parts` | No | Subset of `text`, `data`, `file`; default `[text, data]`. Other parts are dropped. |
| `policy.allow_insecure_http` | No | `false` by default. |
| `description` | No | What the model reads. Missing → built from the remote card (quoted as untrusted); write one when the card is vague. |
| `rate_limit` | No | `{max_calls, window_seconds}`. |

An unknown key in `policy` or `auth`, an http `card_url` without the opt-in, or a
secret written in the YAML raise at load: **the agent does not load**. Nothing
touches the network at load time.

What the model sees and what the runtime guarantees:

- The tool takes `{query}` and returns `{answer, status, task_id?, pending?}`.
  The remote's output arrives **wrapped as untrusted** text; the tool's
  description tells the model never to treat it as instructions and never to
  answer a confirmation on the user's behalf.
- Every call goes out with origin **agent**: the remote never sees it as a person.
- If the remote asks for more input (`input_required`), the task stays pending
  per `(session, tool)`, and the next call in the **same session** answers it.
- A remote that is down, or a failed task, comes back as an error observation
  (`{error, status: "failed"}`); the run goes on.
- The usage the remote reports (a `data` part `{"usage": {...}}`, which Nexus
  0.28.2+ sends) goes to the tool's trace span (`a2a.usage_remoto`), **not** to
  `usage.by_model`: it is not our model, and its `credits` are not money.

**On Nexus** (pool on core 0.74.1, Nexus 0.28.2) an `a2a` tool loads, but `auth`
reads an environment variable of the **shared runtime pool**, which carries only
the LLM provider keys. A tenant cannot set one. So from Nexus only a remote that
needs no credential works, unless the operator adds the variable to
`astromesh-runtime-secrets` (and then every tenant's agents can name it). For a
remote that needs a key, use a standalone node. Say this when you emit `auth`
for a Nexus deployment.

### Tool result budget (core v0.66.0+)

Every tool result is serialized (dicts and lists as compact JSON) and capped
when it enters the conversation, so one large response cannot fill the window:
a list keeps its first elements plus `{"_omitidos": N, …}`, a dict trims its
largest (possibly nested) list, anything else is cut at the head. The cap is
`tools[].max_result_tokens` → `orchestration.max_tool_result_tokens` → `8000`.
It applies to `react`, `plan_and_execute`, `pipeline`, `parallel_fan_out`,
`supervisor` and `swarm` (0.71.0+), and Glyph programs receive full data.

**Example tools array:**
```yaml
tools:
  - type: builtin
    name: web_search
    rate_limit:
      max_calls: 10
      window_seconds: 60
  - type: agent
    name: lookup_inventory
    agent: inventory-agent
    description: "Check product availability and pricing"
  - type: client
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
  - type: integration          # core v0.37.0+
    name: praxis
    connection: praxis_main
    actions: [buscar_records, crear_record]
    confirm: [crear_record]     # core v0.42.0+ — incompatible with spec.chain
    max_result_tokens: 4000     # core v0.66.0+
```

---

## spec.prefetch (optional) — astromesh core v0.51.0+

Read-only lookups that run **before the model**, so a turn that always starts by
looking up the same thing does not spend an LLM round trip deciding to do it.
Results land in the template variable `prefetch.<name>` as
`{success, data, metadata, error}`, and each call emits a `tool.prefetch` span.

```yaml
spec:
  tools:
    - name: praxis
      type: integration
      connection: praxis_main
      actions: [buscar_records]
  prefetch:
    - name: cliente
      tool: praxis_buscar_records          # the REGISTERED name, <slug>_<action>
      arguments:
        entidad: cliente
        filter: "telefono:eq:{{ sender_phone }}"
      when: "sender_phone is defined"      # an expression, not a template
  prompts:
    context: |
      {% if prefetch is defined and prefetch.cliente.success %}
      Customer: {{ prefetch.cliente.data }}
      {% endif %}
```

| Field | Required | Description |
|-------|----------|-------------|
| `name` | Yes | Key under `prefetch`. Unique within the block. |
| `tool` | Yes | A registered integration action — a catalog `integration` action or an `api` read operation — whose method is `GET` and that is **not** in `confirm`. |
| `arguments` | No | Object. String values are Jinja2 templates rendered with the run's caller context and earlier entries (`prefetch.<name>`). |
| `when` | No | Jinja2 **expression**, evaluated (not rendered). False skips the entry. |

- **An invalid declaration does not load the agent** (`runtime/prefetch.py`): unregistered tool, not an integration action, non-`GET`, in `confirm`, repeated `name`, and (v0.51.1+) a `when` or `arguments` template that does not compile.
- **A lookup that fails at run time does not fail the turn** — `success: false`, and the model keeps its tools.
- Prefetch results change every query: render them in `prompts.context`, not `prompts.system`.
- `prefetch` is **not declared in the 0.68 editor schema** (`spec` has `additionalProperties: false`), so a JSON-schema validator flags it although the runtime reads it (`engine.py:1247-1251`).

---

## spec.memory (optional)

Configures agent memory backends.

### spec.memory.conversational (optional)

> **Structure matters:** the runtime reads conversational settings **nested under `conversational`** (i.e. `spec.memory.conversational.strategy`, `spec.memory.conversational.max_turns`). A flat block like `memory: {type: conversational, backend: ...}` is silently ignored and defaults are used. Always nest.

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `backend` | string | optional | -- | Storage backend. **Only `redis` is built by the runtime** (`astromesh/memory/factory.py`). The editor schema also accepts `in_memory`, `sqlite`, `postgres`, but declaring one of those logs a warning and the agent runs **without conversational memory** — it re-introduces itself on every message. Same outcome if the node lacks the `redis` extra. |
| `connection.url` | string | REQUIRED with `redis` | -- | Redis DSN, e.g. `redis://redis:6379/0`. Read with no default: omit it and the agent runs **without memory** (warning at load, core v0.44.1+). |
| `strategy` | string | optional | `"sliding_window"` | One of: `sliding_window`, `summary`, `token_budget`. `summary` folds turns that leave the `max_turns` window into a running summary — one extra model call per exchange, on the `summarizer` role (falls back to `default`). |
| `max_turns` | integer | optional | `50` | Turns read back per run. This is the key the runtime reads; `max_messages` is accepted by the editor schema but **not read** by the runtime (`core/memory.py`). |
| `ttl` | integer | optional | `259200` | Time-to-live in seconds (72 h). |

Whatever the strategy, history is sent to the model **as messages** and trimmed
to a budget derived from the model's `context_window` (core v0.64.0+; see
`spec.model`). Before core v0.44.0 no agent had conversational memory with any
backend — the backend was never constructed.

### spec.memory.semantic / spec.memory.episodic (built from the manifest, core 0.71.0+)

Since core 0.71.0 both blocks are built from the manifest (`build_semantic_memory`,
`build_episodic_backend` in `astromesh/memory/factory.py`); before, they were accepted and did nothing.
Each backend needs its extra (`astromesh[chromadb|faiss|qdrant|postgres|embeddings]`). **A declared
block that cannot be built (missing `embeddings`, unknown backend, missing extra, missing
`dimension`/`connection.url`) is an `AgentConfigError`: the agent stays in `draft` and does not
load.** A backend that fails at run time (initialize, search, recall, store) is logged and the run
continues without that memory. On a node older than 0.71.0 the blocks are ignored silently.

#### spec.memory.semantic

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `backend` | string | **required** | -- | `chromadb`, `faiss`, `pgvector` or `qdrant`. Anything else (including `in_memory`) is an `AgentConfigError`. |
| `embeddings` | object | **required** | -- | Embedder, same shape as RAG's. `provider` is **required** (no default) and is `ollama`, `hf` or `sentence_transformer` (the last needs `astromesh[embeddings]`), plus `model`, etc. There is no implicit default. |
| `dimension` | integer | required except `chromadb` | -- | Vector size; must match the embedding model. |
| `connection.url` | string | **required** for `pgvector`; optional for `qdrant` | -- | DSN of the store (pgvector) / server URL (qdrant). |
| `path` | string | optional | -- | `chromadb` only: storage directory. |
| `similarity_threshold` | number | optional | `0.75` | Minimum cosine similarity for retrieval, 0.0-1.0. |
| `max_results` | integer | optional | `10` | Maximum number of results to return per query. |

In the prompt, `memory.semantic` arrives as a list of `{content, similarity, source}` (no vector).
Semantic writes no longer depend on having conversational memory. The search is scoped to the
**same session**: one person's stored answers never reach another person's prompt.

#### spec.memory.episodic

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `backend` | string | **required** | -- | `postgres` (the only one; needs `astromesh[postgres]`). |
| `connection.url` | string | **required** | -- | Postgres DSN. |

Each run records one episode (`ok`, `blocked`, `error` or `timeout`, the tools called, tokens and
the guardrail rule; if the input was blocked, or the run failed before the input guardrails, the
query is not stored). In the prompt, `memory.episodic` carries the 5 most recent episodes of the
**same session** (filtered in the backend query).

A sub-agent (a `type: agent` tool or a swarm target) keeps its memory in its own session,
`<session_id>::<agent name>`, not in the person's.

---

## spec.knowledge (optional)

Knowledge the agent can draw on. At least one of `pipeline` or `bundle` is
required by the editor schema (`anyOf`); they are independent and can be combined.

| Field | Type | Since | Description |
|-------|------|-------|-------------|
| `pipeline` | string | core v0.30.0 | `metadata.name` of a `RAGPipeline` resource. Chunks are retrieved for every query and exposed to the templates as `knowledge`. Unknown or unbuildable pipeline → warning, the agent runs without it (`engine.py:827-845`). |
| `top_k` | integer | core v0.30.0 | Chunks per run, `pipeline` only. Default: the pipeline's `retrieval.top_k` (else 5). |
| `bundle` | string | **core v0.68.0** | Directory of an [Open Knowledge Format](https://github.com/GoogleCloudPlatform/open-knowledge-format) bundle (v0.2, also reads v0.1), relative to the node's config directory or absolute. Not a directory → warning, the agent runs without it (`engine.py:808-825`). |

### Knowledge bundles (core v0.68.0+)

An OKF bundle is a directory of markdown concepts with YAML frontmatter,
cross-linked like a wiki. No embeddings, no vector store: the agent
**navigates** it. With `bundle` set the runtime registers three read-only tools
(do **not** declare them in `spec.tools`):

| Tool | Arguments | Returns |
|------|-----------|---------|
| `kb_index` | `path` (default `/`) | The directory's `index.md`, or one synthesized from frontmatter. |
| `kb_read` | `concept_id` | `type`, `title`, `trust`, `stale`, `warnings`, body, and links resolved to concept ids. |
| `kb_find` | `text`, `type`, `tags`, `include_deprecated`, `limit` (≤ 100, default 20) | Matching concepts without body. Deprecated ones excluded unless asked. |

and fills the template variable `bundle_index` with the root index.
`bundle_index` is stable between queries (it changes only when the bundle does,
or a root concept passes its `stale_after`), so put it in **`prompts.system`** —
it keeps the cache warm and saves the agent its first `kb_index` call. Per-query
data (`knowledge`, `prefetch`) goes in **`prompts.context`**.

Trust signals are reported, never used to hide a concept: `trust`
(`human-reviewed` / `machine-confirmed` / `unverified`), `stale` (past
`stale_after`), and `warnings` such as `DEPRECATED: …`. **Tell the agent in its
prompt what to do with them** (e.g. refuse deprecated sources, flag stale or
draft ones).

Rules:

- A declared tool named `kb_index`, `kb_read` or `kb_find` collides with the
  bundle's — the agent does not load.
- The bundle lives on the **node's filesystem** (relative to its config
  directory). Remote bundles (git, tarball) are not supported; nor is ingesting a
  bundle into a `RAGPipeline`, or writing to it.
- Read-only and confined to the bundle root (symlinks included); nothing in the
  bundle is executed. A concept body reaches the model like any tool result —
  mount bundles only from trusted sources.
- `kb_find` scans the whole bundle each call: fine for hundreds to a few
  thousand concepts.
- On a node older than v0.68.0 `knowledge.bundle` is ignored: no `kb_*` tools,
  empty `bundle_index`.

**pipeline vs bundle.** Use `pipeline` for a large or unstructured corpus where
"find the most similar chunks" is the right question (manuals, tickets, FAQs
scraped from a site). Use a `bundle` for curated, structured knowledge whose
**trust and freshness matter** — policies, product references, procedures — and
that is small enough to navigate. Both can coexist.

```yaml
spec:
  knowledge:
    bundle: knowledge/acme_retail       # core v0.68.0+
    pipeline: product-knowledge         # optional, independent
    top_k: 4
  prompts:
    system: |
      You answer Acme support questions from the knowledge bundle only.
      Never cite a concept whose warnings say DEPRECATED; say when a source is stale.
      {{ bundle_index }}
    context: |
      {% if knowledge %}RELATED DOCUMENTS:
      {{ knowledge }}{% endif %}
```

---

## spec.guardrails (optional)

Input and output rules, **enforced since core 0.70.0** (`astromesh/core/guardrails.py`; on 0.68/0.69 the section is stored but never run). Both `input` and `output` are arrays of rule objects. `Agent.run` checks input before memory and model, output on the final answer (and on each streamed `token`), and the budget before every model call. Unknown types, wrong side, bad keys or invalid values are dropped with a warning at load; the agent still loads.

### Rule types

| Rule Type | Applies to | Keys (default) | Actions (default first) | Behaviour |
|-----------|-----------|----------------|--------------------------|-----------|
| `pii_detection` | input, output | - | `redact`, `block`, `warn` | Redacts emails, US-format phones (10 digits), SSNs and 16-digit card numbers with `[REDACTED_*]` tags. |
| `topic_filter` | **input only** | `blocked_topics` (alias `forbidden_topics`) | `warn`, `block` | Case-insensitive **substring** match; `warn` only counts, `block` rejects. |
| `max_length` | **input only** | `max_chars` (10000; alias `max_characters`) | `truncate`, `block` | Cuts or rejects an over-long incoming message. It never limits the reply. |
| `content_filter` | **output only** | `blocked_patterns` (regex, case-insensitive), `forbidden_keywords` (literal) | `redact`, `block` | `redact` replaces each match with `[FILTERED]`. |
| `cost_limit` | either side | `max_tokens_per_turn` (**required**, positive int), `count: output\|total` (`output`) | `block` | Output-token budget (or in+out with `count: total`) for the WHOLE run, checked before each model call; with several, the smallest wins. |

Every rule may carry `message` (text returned on a block; default `I can't help with that request.`). `blocked_categories` and `allowed_topics` need a classifier and are **unsupported** (warned at load) - don't emit them; use `blocked_patterns` or the system prompt. `prompt_injection`, `log` actions and `limit` are not implemented.

### Blocked runs

A block is not an error: the run returns a normal response with `blocked: true`, the rule's `message` as the answer, and `guardrail: {direction, rule, reason}` (`direction` is `input`, `output` or `run`; `reason` never repeats the blocked content).

### Caveats an author must know

- **Input `pii_detection: redact` hides data from the model AND the tools.** Emails, phones, 10-digit and 16-digit numbers are replaced before anything runs, so an agent whose tools take a phone, customer ID or order number breaks. Redact on `output` instead.
- PII patterns are US-format: an Argentine phone number is not detected.
- `cost_limit` counts the whole run (every iteration and tool round), not one call; about 4000 suits tool-using agents.
- `topic_filter` is substring matching, not semantic.
- Manifests that carried `guardrails` on 0.68 start being enforced on 0.70; review them.

**Example:**
```yaml
guardrails:
  input:
    - type: max_length
      max_chars: 4000
    - type: topic_filter
      blocked_topics: [politics, religion]
      action: block
      message: "I can only help with orders and support."
  output:
    - type: pii_detection
      action: redact
    - type: content_filter
      blocked_patterns:
        - "(?i)\\b(kill yourself|kys)\\b"
      action: block
    - type: cost_limit
      max_tokens_per_turn: 4000
```

---

## spec.permissions (optional)

**Enforced since core 0.71.0** (`astromesh/runtime/permisos.py`), and only these two keys exist. Any other key (`filesystem`, `network`, `execution`, ...) is ignored with a WARNING at load. On a node older than 0.71.0 the whole section is stored and does nothing.

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `allowed_actions` | string array | optional | -- | Whitelist of **registered tool names** with `fnmatch` wildcards (`*`, `?`, `[]`). Tool names are what the runtime registered: `<slug>_<action>` for `integration`/`api`/`mcp`, `kb_index`/`kb_read`/`kb_find` for knowledge bundles, the declared `name` for `builtin`/`agent`/`client`. A tool that matches nothing is left out. It is *not* a list of `respond`/`use_tools`/`delegate`/`escalate` verbs. |
| `denied_actions` | string array | optional | -- | Blacklist with the same names and wildcards. **`denied` wins** over `allowed`. |

Applied both when offering tools to the model and when executing them. A pattern that matches no
registered tool, or a result that leaves the agent with no tools, logs a warning/error; a
`prefetch` that names a tool the agent is not allowed to use is an `AgentConfigError`. A value that
is not a list of strings is an `AgentConfigError`. Declaring only `denied_actions` allows everything
else.

```yaml
permissions:
  allowed_actions: ["crm_*", "kb_*"]
  denied_actions: ["crm_delete_*"]
```

This is a tool-visibility control, not a sandbox. Real write safety still comes from `confirm`
(integrations) and `mode: propose` (`api`/`mcp`); network safety for `api`/`mcp` comes from the
runtime's public-host guard.

---

## spec.skills (optional) — astromesh core v0.73.0+

What the agent can do, as a catalog other agents read. They become the `skills` of the
agent's **A2A Agent Card** (Nexus, `astromesh-a2a`'s server) and `GET /v1/agents/{name}`
returns them. They do not change how the agent runs.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | Yes | Stable identifier (`cotizar`, `book-appointment`). |
| `name` | string | Yes | Short human name. |
| `description` | string | Yes | One or two sentences a calling agent can decide on. |
| `tags` | string array | No | Keywords. |
| `examples` | string array | No | Sample requests. |
| `inputModes`, `outputModes` | string array | No | MIME types, when they differ from the card's defaults (`text/plain`, `application/json`). |

An invalid entry (missing or empty `id`/`name`/`description`, a list that is not a list
of strings) is **skipped with a warning** and the agent loads anyway. On a node older than
0.73.0 the whole section is ignored silently.

```yaml
skills:
  - id: reservar-turno
    name: Reservar turno
    description: Books a service appointment for a car, given plate, service and preferred day.
    tags: [turnos, taller]
    examples: ["Turno para cambio de aceite el jueves, patente AB123CD"]
```

Write skills from what the agent's tools and prompt really do; a skill the agent cannot
perform is a promise another agent will act on.

---

## spec.a2a (optional) — read by Nexus 0.27.0+, ignored by the core

Publishes the agent over A2A 1.0 in Nexus. The core runtime does not read it.

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `publicar` | boolean | `false` | `true` → Nexus serves the Agent Card at `/a2a/v1/agents/<name>/.well-known/agent-card.json` and the JSON-RPC endpoint at `/a2a/v1/agents/<name>`. |

```yaml
spec:
  a2a:
    publicar: true
  identity:
    description: "Books and cancels service appointments for the workshop."  # the card's description
  skills: [ ... ]                                                          # the card's skills
```

- It is **opt-in** and read from the **live version**: publishing a new version without it
  unpublishes the agent. An unpublished agent answers `404 {"error":"agent not found"}` on every
  A2A route, the same as one that does not exist.
- The Nexus deployment also needs `NEXUS_PUBLIC_URL`; without it no `/a2a/v1` route exists.
- No anonymous access: the card and every call need the tenant's `X-API-Key` (or a JWT plus
  `X-Tenant-ID`). An A2A run is a normal Nexus invocation (`adapter = "a2a"`): same plan limits,
  credits and billing.
- A2A turns reach the node with `origen: "agente"`, so another agent can never confirm a
  `confirm`-gated action. On a pool older than core 0.73.0 that was not true: do not publish an
  agent with `confirm` tools to a Nexus whose pool runs core < 0.73.0.
- Give it a real `spec.identity.description` and `spec.skills`: they are all a calling agent knows.

---

## spec.observability (optional)

Accepted by the editor schema: `tracing` (bool), `metrics` (bool), `collector`
(`stdout` | `otlp` | `internal`), `sample_rate` (0.0–1.0). `_build_agent` does not
read it per agent in 0.68 — tracing is configured at the node. Omit it unless the
user asks.

---

## Channels (deployment-time, not in the agent spec)

There is **no `spec.channels` block read by the runtime**. An agent is bound to a channel like WhatsApp at deployment time, through:

1. **Environment variables** on the astromesh-node running the agent (e.g. `WHATSAPP_VERIFY_TOKEN`, `WHATSAPP_ACCESS_TOKEN`, `WHATSAPP_PHONE_NUMBER_ID`, `WHATSAPP_APP_SECRET`), typically injected as Kubernetes Secrets.
2. **A per-agent webhook endpoint** exposed by the node: `GET/POST /v1/agents/{name}/channels/whatsapp/webhook` (GET verifies the token, POST receives messages). Register that URL in the Meta App Dashboard.

Use `metadata.labels` (e.g. `channel: whatsapp`) to *tag* a channel agent, but do not put channel credentials in the manifest. See `schemas/whatsapp-config.md` for the full WhatsApp flow. Supported channels today: **WhatsApp** (production-ready). Telegram is recognized by tooling but not yet implemented in the runtime.

---

## Complete Example

A sales lead qualification agent deployed on WhatsApp, valid against the core
v0.68.0 editor schema and using only what the 0.68 runtime actually reads
(needs a node at core v0.68.0+ for `knowledge.bundle`, v0.64.0+ for
`prompts.context` / `context_window`, v0.38.1+ for `output_schema`):

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
      context_window: 128000          # core v0.64.0+: sizes the history budget
      parameters:
        temperature: 0.4
        top_p: 0.95
        max_tokens: 1024
    fallback:
      provider: ollama
      model: llama3.1:8b
      endpoint: http://ollama.sales-team.svc.cluster.local:11434
      timeout: 60
      context_window: 8192
      parameters:
        temperature: 0.4
        max_tokens: 1024
    routing:
      strategy: quality_first
      health_check_interval: 30

  prompts:
    # Stable between turns -> cacheable. No per-query variables here.
    system: |
      You are the sales qualification assistant for Acme Corp, talking to
      prospects on WhatsApp.

      Your goal is to understand the prospect's needs, company size, budget and
      timeline through natural conversation. When you have enough, score the lead
      with the score_lead tool. For pricing questions call lookup_pricing.

      Be professional but conversational. Never be pushy. If someone is not
      interested, thank them politely and end the conversation. Keep replies short.
      {% if contact_name %}Address the prospect as {{ contact_name }}.{% endif %}

      Product knowledge map (open concepts with kb_read; never quote a concept
      whose warnings say DEPRECATED):
      {{ bundle_index }}
    # Per-query data goes here, next to the user's message.
    context: |
      {% if knowledge %}RELEVANT CASE STUDIES:
      {{ knowledge }}{% endif %}

  orchestration:
    pattern: react
    max_iterations: 10
    max_tool_result_tokens: 4000      # core v0.66.0+

  tools:
    - type: client
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
        required: [company_size, timeline]
    - type: agent
      name: lookup_pricing
      agent: pricing-agent
      description: "Look up product pricing for a given tier and feature set"

  knowledge:
    bundle: knowledge/acme-products   # core v0.68.0+: registers kb_index/kb_read/kb_find
    pipeline: case-studies            # RAGPipeline, retrieved per query as `knowledge`
    top_k: 3

  memory:
    conversational:
      backend: redis                  # the only backend the runtime builds
      connection:
        url: redis://redis.sales-team.svc.cluster.local:6379/0
      strategy: sliding_window
      max_turns: 20
      ttl: 259200

  output_schema:                      # core v0.38.1+: adds `data` to the run response
    score: {type: integer}
    qualified: {type: boolean}
```

Deliberately absent: `memory.semantic` / `episodic` (they need an embedder and a vector store or
Postgres the example does not have), `permissions` (this agent has no restriction to express),
`prompts.templates` (no pattern reads them), and a `spec.channels` block (does not exist —
WhatsApp is wired at deploy time). `orchestration.timeout_seconds` is omitted too: if you set it,
use >= 120 (core 0.71.0 applies it and answers 504 on expiry).

---

## Streaming contract & usage (astromesh core v0.34.0 / v0.36.0)

A run is observable while it happens. `AgentRuntime.run()` accepts an `on_event`
callback and `/v1/ws/agent/{name}` streams the same events over WebSocket:

```
status → (token | tool_call | tool_result)* → done | error
```

- `token` — one whole model completion (per-iteration reasoning under ReAct; NOT the final answer).
- `tool_call` — `{id, name, arguments}`, emitted **before** the tool runs. For a `client` tool this event *is* the delivery.
- `tool_result` — `{id, ok}`, after the tool returns. A `client` tool always reports `ok: true`.
- `done` — `{answer, session_id, usage}`.

### usage.by_model (v0.36.0)

`done.usage` and the `/run` response carry per-model attribution — a single run
routinely touches several models (multi-model patterns, per-role routing,
provider fallback), so the flat `usage.model` has no correct value there.

```
usage = {
  tokens_in, tokens_out,            # totals (flat, kept for compatibility)
  model,                            # first model seen — legacy
  by_model: [ { provider, model, role, calls, tokens_in, tokens_out, cost } ]
}
```

`by_model` is the authoritative breakdown; sort is by descending consumption.
There is no cost/credits economics here beyond the provider `cost` estimate —
tenant billing lives in the nexus hub, not the core run response. Since core
0.73.0 it includes the usage of sub-agents (`type: agent` tools); a remote A2A
agent's reported usage does **not** enter it (see *Type: a2a*).

### Who is speaking: `origen` (core v0.73.0+)

`POST /v1/agents/{name}/run` and the WebSocket messages accept
`origen: "humano" | "agente"` (default `"humano"`). A turn with `origen: "agente"`
**never confirms** an action gated by `confirm`: only a person can say "sí". An
invalid value is a `422` (REST) or an `error` event (WS); it never falls back to
`humano`. Nexus sends `agente` for A2A runs, and the ADK's `bind()`/`connect()`/
`remote()` send it too. When you test a confirmation flow by hand, send `humano`
(or nothing).
