# Astromesh ecosystem — what exists and when to point at it

Leia designs `astromesh/v1` manifests and operates them on Nexus. Users also ask about the rest
of the ecosystem: "can my LangGraph agent call this one?", "should I use the RAG library?",
"which Nexus do I need?". This file answers those questions. **State as of 2026-10-26.** The
latest released versions are below. Never recommend a capability that is not listed here or in
the component's own docs (`https://monaccode.github.io/astromesh/`).

## When the user asks for… → point at

Leia hears requests in Spanish and English. Match the request to the row, answer from the linked
section, and say plainly when the answer is "not Leia's job, use X".

| The user says (examples) | Answer with | Leia does |
|---|---|---|
| "creá / armá un agente", "build an agent" | A manifest | `/leia create` (architect) |
| "que otro agente lo llame", "exponelo por A2A", "agent card" | `spec.a2a.publicar` + `spec.skills` | architect step 5a, then the operator's A2A check |
| "que llame a un agente de afuera", "call this A2A agent" | `type: a2a` tool | architect |
| "mi agente de LangGraph / CrewAI / ADK / OpenAI SDK… tiene que hablar con este" | `astromesh-a2a` (`RemoteAgent`, `as_tool()`) | explain, no YAML |
| "lo quiero en código / Python, no YAML" | Astromesh ADK | explain, no YAML |
| "lo quiero editar visualmente / con formularios" | Forge (on the node, `/forge`) or Cortex (desktop) | explain |
| "¿por qué no aparece la card?", "¿qué ve el que llama?" | Cortex 0.29 A2A panel, or the operator's A2A check | operator |
| "mejorá la recuperación en mi propio código", "RAG en Python" | `astromesh-rag` | explain; on a node it is `knowledge.pipeline` |
| "qué entra en la ventana", "presupuesto de tokens", "contexto" | `astromesh-context` | explain; not a manifest field |
| "revisá / serví este bundle OKF", "conocimiento para Claude o Cursor" | `astromesh-okf` | `uvx astromesh-okf index <bundle>` |
| "WhatsApp / Instagram / Messenger / Telegram / web chat / email" | Herald + Nexus (WhatsApp also native on a node) | plain manifest; channel is wired outside `spec` |
| "un programa en vez de muchas tool calls", "Glyph" | `pattern: glyph` with a fixed `spec.program` | architect, with the cost warning |
| "correlo en mi servidor / en mi máquina" | Node (`astromeshd`) | `/leia deploy --node` |
| "un appliance / imagen inmutable" | Astromesh OS | explain |
| "en GCP con mi propia infraestructura" | Orbit | explain |
| "que sea el agente nativo de la nube / Agent Engine" | Prisma — **in development, provisions nothing yet** | explain, do not promise |
| "un modelo propio / en español / sentiment financiero" | Nebula and Centinela (`source: centinela`) — **preview** | explain the status first |
| "multi-tenant, créditos, facturación" | Nexus | `/leia deploy` |
| "specs antes de codear", "SDD" | Astromesh SDD | explain (about coding agents, not runtime agents) |

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
| **Astromesh Forge** | 0.29.1 | Visual agent editor, served by the node at `/forge`. Creates and edits `type: a2a` tools with validation that mirrors the core (0.29.1: a bracketed `card_url` host must be IPv6, as in the core). | The user prefers a form to YAML. |
| **Astromesh Cortex** | 0.29.0 | Desktop IDE and console that reaches local, GCP and Nexus runtimes. 0.28: schema at parity with core 0.74.1 — `type: a2a` tool form, an A2A block (`spec.a2a.publicar`, card description, `spec.skills`) and snippets that validate. 0.29: an **A2A panel** on each Nexus agent shows the Agent Card a caller sees, or why there is none (not published, or no A2A on that Nexus). | Authoring by hand, publishing YAML, or checking a card visually. |
| **Astromesh SDD** | 0.1.0 | Spec-Driven Development skills for any coding agent. | — (not about agents at runtime). |
| **Astromesh Leia** | 0.11.1 | This plugin. | — |

## Open-source libraries (standalone, any framework)

None of these needs the Astromesh runtime. The core depends on `astromesh-okf` and, through the
`a2a` extra, on `astromesh-a2a`. It does **not** use `astromesh-rag` or `astromesh-context`:
those are separate libraries, not manifest fields.

| Library | Version | What it is | Point at it when |
|---|---|---|---|
| **astromesh-a2a** | 0.1.1 (PyPI) | A2A 1.x client and server built to distrust the remote: host allowlist, size and time caps, untrusted output, never auto-confirm, `Origin.AGENT`. `RemoteAgent` + task API (`submit`, `get_task`, `subscribe`, `list_tasks`, `reply`). `as_tool()` for LangChain/LangGraph, CrewAI, Google ADK, OpenAI Agents SDK, PydanticAI, LlamaIndex, smolagents, Microsoft Agent Framework and MCP. `serve()` publishes your own agent (auth mandatory). | An agent in another framework must call an Astromesh/Nexus agent (point it at the Nexus card with the tenant API key), or a non-Astromesh agent must be callable from a `type: a2a` tool (serve it with `astromesh-a2a[server]`). |
| **astromesh-okf** | 0.1.0 (PyPI) | Reader for Open Knowledge Format bundles: `index`, `read`, `find`, and an MCP server (`astromesh-okf mcp <bundle>`). Read-only and confined to the bundle root. | Checking a bundle the architect wrote (`uvx astromesh-okf index <bundle>`), or serving a bundle to Claude/Cursor over MCP without Astromesh. Format: `schemas/okf-bundle.md`. |
| **astromesh-rag** | 0.1.0 (PyPI) | RAG patterns (`naive`, `hybrid`, `multi_query`, `hyde`) from the same parts, with a trace per stage. | The user builds retrieval in their own Python code. On a node, retrieval is still a `RAGPipeline` (`schemas/astromesh-v1-rag.md`) and `knowledge.pipeline`. |
| **astromesh-context** | 0.1.0 (PyPI) | Context engine: one token budget for RAG hits, OKF concepts, memory and history, with an included/excluded trace per item. Includes a published eval: level 1 (retrieval, no model calls) is run and published; **level 2 (answer quality, judged by a model) has not been run** — never claim it improves answers. CLI `astromesh-context explain`, and `astromesh-context mcp`. | The user asks how to decide what goes into the window in their own stack. It is not a manifest field. |
| **Glyph** | 0.1.3 | Action language: the model writes one program instead of one tool call per turn. Consumed by the core as `pattern: glyph` (extra `glyph`). Measured **+164% to +2839%** the cost of ReAct when the model writes the program each run. | Only a fixed `spec.program` (see `schemas/astromesh-v1-agent.md`). Never pitch it as a saving. |

## Deploy targets that are not Nexus

| Component | Version | Status | What it is |
|---|---|---|---|
| **Astromesh Orbit** | 0.4.4 | Released | Runs the Astromesh runtime on your own GCP infrastructure; Terraform you can eject to. |
| **Astromesh Prisma** | 0.2.1 | **In development** (private, not on PyPI) | Turns a manifest into the cloud's own native agent (on GCP an ADK config on Agent Engine with Sessions, Memory Bank, RAG Engine and Model Armor), reporting every gap. Today it reconciles against an in-memory gateway and **provisions nothing**. Routing/fallback, episodic memory, `mcp`/`api`/`agent` tools and the fan-out, supervisor, swarm and glyph patterns do not translate. |

Orbit or Prisma: Orbit keeps the whole runtime; Prisma hands the agent to the cloud's managed
service with a smaller surface. Only Orbit can be used today.

## Models

| Component | Version | Status | What it is |
|---|---|---|---|
| **Astromesh Nebula** | 0.1.0 | Pipeline complete; first Hub release **not out** | The model foundry: trains, gates and publishes the ecosystem's own models. It does not serve inference. |
| **Centinela-Qwen3-4B** | v0.1 | **Preview** (catalog has placeholder revisions) | Spanish financial sentiment classifier (`positivo`/`neutral`/`negativo`), reached with `source: centinela`. |

Do not design an agent that depends on Centinela without telling the user it is a preview.

## Channels

| Component | Version | What it is |
|---|---|---|
| **astromesh-herald** | 0.13.0 | Communications gateway: a person's message reaches a Nexus agent and the agent answers through the same outbox. Channels: WhatsApp, Instagram, Messenger (0.12.0; `pages_messaging` still in Meta App Review, so it is exercised through the emulator), Telegram, Web Chat, Email. 0.13.0 adds `GET /admin/stats/conversaciones` (admin token): new conversations per local day, hour and channel in an IANA `tz`, up to 62 days — the numbers behind an owner's dashboard. |

An agent that answers on WhatsApp through Herald is still a plain manifest: the channel is wired in
Herald and Nexus, never in `spec` (see `schemas/whatsapp-config.md`).

## A2A at a glance (what shipped 2026-10-19 → 10-22)

| Need | Use | Gate |
|---|---|---|
| Publish a Nexus agent to other agents | `spec.a2a.publicar: true` + `spec.identity.description` + `spec.skills` | Nexus 0.27.0+, skills core 0.73.0+ |
| Background runs, polling, `SubscribeToTask`, `ListTasks`, cross-pod `CancelTask` | Nexus A2A server | Nexus 0.28.0+ |
| Push notifications | Operator flag `NEXUS_A2A_PUSH=true` + secret. Off in dev and mvp. | Nexus 0.28.0+ |
| Run usage in the A2A result (`{"usage": {...}}` data part, Nexus credits) | Automatic | Nexus 0.28.2+ |
| A manifest agent calling a remote A2A agent | `type: a2a` tool | core 0.74.0+ (Nexus pool 0.28.1+) |
| Python/ADK code calling a remote A2A agent | `remote_a2a()` | ADK 0.7.0+ |
| Any other framework calling or serving A2A | `astromesh-a2a` | 0.1.0+ |
| Edit `type: a2a` tools and the A2A block in a form | Forge, Cortex | Forge 0.29.0+, Cortex 0.28.0+ |
| See the published card from a desktop app | Cortex A2A panel | Cortex 0.29.0+ |
| Another agent's turn must never confirm a write | `origen: "agente"` (sent by Nexus A2A and the ADK) | core 0.73.0+ |

Not available yet: agent identity across organizations (PAP/Poppy) is in development in a
separate product and is not part of Nexus or the core. Do not promise it.
