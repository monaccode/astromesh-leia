# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.3.0] - 2026-07-10

Synced Leia's knowledge to astromesh **core v0.29.0**, teaching the plugin the transversal **per-role model** feature so the architect can design agents that bind a different model — and source — to each orchestration role.

### Added

- **`schemas/astromesh-v1-agent.md`: per-role model selection** (astromesh v0.29.0+). New section documenting the `spec.model.default` + `roles` shape: `candidates` lists with fields `source`/`model`/`endpoint`/`api_key_env`/`api_key`/`parameters`; the `litellm` cloud multi-provider source (100+ models via the runtime's optional `litellm` extra) alongside `ollama`/`openai_compat`; source inference from the model string; graceful skip when `litellm` isn't installed; the role vocabulary each pattern requests; `orchestration.role_map` remapping; and backward compatibility (legacy `primary`/`fallback`/`extra` normalize into the `default` role).
- **`schemas/orchestration-patterns.md`: per-role table** mapping each pattern to the roles it requests (`reasoner` / `planner` / `worker` / `synthesizer` / `supervisor` / `stage:<name>`).
- **`agents/leia-architect.md`: per-role guidance.** The architect now offers per-role models when the pattern has distinct reasoning/execution roles and a cloud key is available (strong `litellm` planner/supervisor + cheap local `ollama` worker/default), with guardrails: `default` is required, `litellm` candidates need `api_key_env`, and per-role and legacy shapes must not be mixed.

### Notes

- Per-role model selection requires the deployed runtime at astromesh core **v0.29.0+**. On older nodes the architect emits the legacy single-model shape. The 6 bundled templates remain single-model (all-Ollama) so they deploy as-is without cloud keys; per-role is opt-in through the architect.

## [0.2.1] - 2026-07-01

Synced Leia's knowledge to astromesh **core v0.28.9** (from v0.28.5), so the architect can design — and the operator can explain — agents backed by Moonshot's Kimi models.

### Added

- **`schemas/astromesh-v1-agent.md`: Moonshot / Kimi provider recipe** (astromesh v0.28.6+). Documented reaching `kimi-k2.5`/`kimi-k2.6` via `provider: openai_compat` + `endpoint: https://api.moonshot.ai/v1` + `api_key_env: MOONSHOT_API_KEY`, including the thinking-model behavior (`reasoning_content` is preserved automatically on tool-call turns; the `400 … reasoning_content is missing` error the runtime guards against) and cache-aware pricing (`cache_read_input_tokens`, derived provider labels) from v0.28.8–v0.28.9.
- **`leia-architect`**: provider-detection guidance now offers Kimi when a `MOONSHOT_API_KEY` is present or requested, while still defaulting to Ollama.

### Changed

- **`schemas/astromesh-v1-agent.md`: `cost_optimized` routing** noted as cache-aware for providers that expose cached tokens (e.g. Kimi's context cache).
- **Version compatibility** table + badge bumped to Leia 0.2.x ↔ astromesh 0.28.9.

## [0.2.0] - 2026-06-08

Synced Leia's knowledge with current astromesh (core v0.28.5, Nexus v0.3.0). This makes the architect generate correct, current manifests and the operator speak the new control-plane API.

### Fixed

- **Templates: memory config was silently ignored.** All 6 templates used a flat `memory:` block, but the runtime reads memory nested under `conversational` (`astromesh/core/memory.py`). Converted every template to `spec.memory.conversational.{backend,strategy,max_turns}` so the configured window actually applies.
- **Templates: `max_length` guardrail was silently ignored.** The runtime reads `max_chars` (`astromesh/core/guardrails.py`), not `limit`. Renamed the field in all 6 templates so the response cap is enforced.
- **`leia-architect` emitted invalid patterns and sections.** It told the architect to use `single`/`chain`/`router`/`parallel` (none of which exist) and to write `spec.system_prompt`/`spec.constraints`. Corrected to the real patterns (`react`, `plan_and_execute`, `pipeline`, `parallel_fan_out`, `supervisor`, `swarm`) and real sections (`spec.prompts.system`, `spec.guardrails`, nested `spec.memory`).

### Changed

- **`schemas/astromesh-v1-agent.md`** modernized to astromesh v0.28.x: added `spec.model.extra` (multi-provider ranking), more generation `parameters` (`top_k`, `frequency_penalty`, `presence_penalty`, `stop`), `round_robin` routing (default is `cost_optimized`), MCP/`webhook`/`rag` tool types, `postgres` conversational + `pgvector`/`qdrant` semantic memory backends, corrected guardrail fields (`pii_detection` action `redact`, `content_filter` `blocked_patterns`, `cost_limit` `max_tokens_per_turn`), the `{{contact_name}}` prompt variable, and a note that channels are wired via env vars + per-agent webhook (no `spec.channels`).
- **`schemas/nexus-api.md`** updated to Nexus v0.3.0: dual auth (JWT Bearer **or** `X-API-Key`), new `/auth/register|login|refresh`, `/api/v1/me`, tenant management, and per-tenant API-key management; agent endpoints now accept either credential.
- **`schemas/whatsapp-config.md`**: documented the per-agent webhook path (`/v1/agents/{name}/channels/whatsapp/webhook`), the `contact_name` context variable, and `system`-direction delivery receipts.
- **Provider guidance**: clarified that only `ollama`/`openai`/`openai_compat`/`azure_openai` are runtime-wired; reach vLLM/llama.cpp/TGI via `openai_compat`.

## [0.1.0] - 2026-04-02

### Added

- Initial release of astromesh-leia Claude Code plugin
- 10 slash commands: `/leia`, `create`, `deploy`, `status`, `logs`, `test`, `templates`, `config`, `bootstrap`, `teardown`
- 5 specialized subagents: `leia-interpreter` (sonnet), `leia-architect` (opus), `leia-operator` (sonnet), `leia-tester` (sonnet), `leia-doctor` (sonnet)
- 6 business-vertical templates: customer support, restaurant booking, ecommerce, appointment scheduling, lead qualification, employee onboarding
- 4 schema references: astromesh/v1 agent spec, nexus REST API, WhatsApp channel config, orchestration patterns
- Multi-context configuration with `~/.astromesh-leia/config.yaml`
- Full cluster lifecycle management (bootstrap Kind, connect remote, teardown)
- Smart model detection (Ollama auto-detect with cloud fallback)
- Interactive and automated agent testing with per-template test scenarios
- Diagnostic engine (`leia-doctor`) for troubleshooting agent and cluster issues
- Comprehensive documentation with Mermaid diagrams:
  - Architecture guide with decision log
  - Complete commands reference (10 commands)
  - Subagents guide (5 agents)
  - Templates guide (6 templates with customization examples)
  - Configuration reference
  - WhatsApp end-to-end setup guide
  - Troubleshooting guide with diagnostic flowcharts
  - 4 tutorials: first agent, business templates, multi-tenant, advanced patterns

[0.1.0]: https://github.com/monaccode/astromesh-leia/releases/tag/v0.1.0
