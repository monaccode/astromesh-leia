# Astromesh ecosystem — what exists and when to point at it

Leia designs `astromesh/v1` manifests and operates them on Nexus. Users also ask about the rest
of the ecosystem: "can my LangGraph agent call this one?", "should I use the RAG library?",
"which Nexus do I need?". This file answers those questions. **State as of 2026-10-21.** The
latest released versions are below. Never recommend a capability that is not listed here or in
the component's own docs (`https://monaccode.github.io/astromesh/`).

## Runtime and control plane

| Component | Version | What it is | Leia cares because |
|---|---|---|---|
| **astromesh** (core) | 0.74.1 (PyPI, `fulfarodev/astromesh`) | The agent runtime that loads `*.agent.yaml`. | Every field the architect emits is gated on a core version (`schemas/astromesh-v1-agent.md`). |
| **astromesh-nexus** | 0.28.2 (dev and mvp) | The multi-tenant hub: agents as versions, invocations, credits, the `/admin` plane, and an **A2A 1.0 server** per published agent. Its runtime pool runs core 0.74.1. | It is where `/leia deploy` publishes (`schemas/nexus-api.md`). |
| **astromesh-node** | 0.1.10 | `astromeshd`, the standalone daemon (`/etc/astromesh`, `systemctl reload`). | `/leia deploy --node`. It is the only place a `knowledge.bundle` or an `a2a` tool with credentials works. |
| **astromesh-os** | 0.13.8 | Immutable, API-only Linux appliance that runs `astromeshd`. It pins a core release. | A node built from it runs the pinned core: check `GET /v1/health` before trusting a version gate. |
| **astromesh-cli** | 0.4.1 | Terminal client for a node. | — |
| **astromesh-orbit** | 0.4.4 | Deploys the stack to GCP. | — |

## Building agents

| Component | Version | What it is | Point at it when |
|---|---|---|---|
| **astromesh-adk** | 0.7.1 | Python SDK: agents as code, `bind()`/`connect()`/`remote()` to a node. 0.7.0 adds `remote_a2a(card_url, …)` (extra `astromesh-adk[a2a]`): a remote A2A agent as a runnable or a tool. Remote calls send `origen: "agente"`. | The user wants code, not YAML, or wants to call a remote A2A agent from Python. |
| **Astromesh Forge** | 0.29.0 | Visual agent editor. Creates and edits `type: a2a` tools with validation that mirrors the core. | The user prefers a form to YAML. |
| **Astromesh Cortex** | 0.27.0 | Desktop IDE and console that reaches local, GCP and Nexus runtimes. | Authoring fields the architect does not know, or publishing YAML by hand. |
| **Astromesh SDD** | 0.1.0 | Spec-Driven Development skills for any coding agent. | — (not about agents at runtime). |

## Open-source libraries (standalone, any framework)

None of these needs the Astromesh runtime. The core depends on `astromesh-okf` and, through the
`a2a` extra, on `astromesh-a2a`. It does **not** use `astromesh-rag` or `astromesh-context`:
those are separate libraries, not manifest fields.

| Library | Version | What it is | Point at it when |
|---|---|---|---|
| **astromesh-a2a** | 0.1.1 (PyPI) | A2A 1.x client and server built to distrust the remote: host allowlist, size and time caps, untrusted output, never auto-confirm, `Origin.AGENT`. `RemoteAgent` + task API (`submit`, `get_task`, `subscribe`, `list_tasks`, `reply`). `as_tool()` for LangChain/LangGraph, CrewAI, Google ADK, OpenAI Agents SDK, PydanticAI, LlamaIndex, smolagents, Microsoft Agent Framework and MCP. `serve()` publishes your own agent (auth mandatory). | An agent in another framework must call an Astromesh/Nexus agent (point it at the Nexus card with the tenant API key), or a non-Astromesh agent must be callable from a `type: a2a` tool (serve it with `astromesh-a2a[server]`). |
| **astromesh-okf** | 0.1.0 (PyPI) | Reader for Open Knowledge Format bundles: `index`, `read`, `find`, and an MCP server (`astromesh-okf mcp <bundle>`). Read-only and confined to the bundle root. | Checking a bundle the architect wrote (`uvx astromesh-okf index <bundle>`), or serving a bundle to Claude/Cursor over MCP without Astromesh. Format: `schemas/okf-bundle.md`. |
| **astromesh-rag** | 0.1.0 (PyPI) | RAG patterns (`naive`, `hybrid`, `multi_query`, `hyde`) from the same parts, with a trace per stage. | The user builds retrieval in their own Python code. On a node, retrieval is still a `RAGPipeline` (`schemas/astromesh-v1-rag.md`) and `knowledge.pipeline`. |
| **astromesh-context** | 0.1.0 (PyPI) | Context engine: one token budget for RAG hits, OKF concepts, memory and history, with an included/excluded trace per item. Includes a published eval. CLI `astromesh-context explain`, and `astromesh-context mcp`. | The user asks how to decide what goes into the window in their own stack. It is not a manifest field. |

## Channels

| Component | Version | What it is |
|---|---|---|
| **astromesh-herald** | 0.12.0 | Communications gateway: a person's message reaches a Nexus agent and the agent answers through the same outbox. Channels: WhatsApp, Instagram, Messenger (0.12.0; `pages_messaging` still in Meta App Review, so it is exercised through the emulator), Telegram, Web Chat, Email. |

An agent that answers on WhatsApp through Herald is still a plain manifest: the channel is wired in
Herald and Nexus, never in `spec` (see `schemas/whatsapp-config.md`).

## A2A at a glance (what shipped 2026-10-19 → 10-20)

| Need | Use | Gate |
|---|---|---|
| Publish a Nexus agent to other agents | `spec.a2a.publicar: true` + `spec.identity.description` + `spec.skills` | Nexus 0.27.0+, skills core 0.73.0+ |
| Background runs, polling, `SubscribeToTask`, `ListTasks`, cross-pod `CancelTask` | Nexus A2A server | Nexus 0.28.0+ |
| Push notifications | Operator flag `NEXUS_A2A_PUSH=true` + secret. Off in dev and mvp. | Nexus 0.28.0+ |
| Run usage in the A2A result (`{"usage": {...}}` data part, Nexus credits) | Automatic | Nexus 0.28.2+ |
| A manifest agent calling a remote A2A agent | `type: a2a` tool | core 0.74.0+ (Nexus pool 0.28.1+) |
| Python/ADK code calling a remote A2A agent | `remote_a2a()` | ADK 0.7.0+ |
| Any other framework calling or serving A2A | `astromesh-a2a` | 0.1.0+ |
| Another agent's turn must never confirm a write | `origen: "agente"` (sent by Nexus A2A and the ADK) | core 0.73.0+ |

Not available yet: agent identity across organizations (PAP/Poppy) is in development in a
separate product and is not part of Nexus or the core. Do not promise it.
