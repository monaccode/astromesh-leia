# Orchestration Patterns Reference

This document describes the seven orchestration patterns available for astromesh agents (synced to **astromesh core v0.68.0**). The architect subagent uses this reference to select the right pattern for each use case.

Since core **v0.65.0** an unknown `spec.orchestration.pattern` is an `AgentConfigError` — the agent does not load (it used to fall back to `react` silently). An explicit `pattern: null` is `react`.

## Pattern Overview

| Pattern | Best For | How It Works | Example Use Case |
|---------|----------|--------------|------------------|
| `react` | General-purpose conversational agents | Reason-Act-Observe loop: the LLM reasons about what to do, calls a tool, observes the result, and repeats. | Customer support bot answering questions and looking up orders. |
| `plan_and_execute` | Complex multi-step tasks requiring upfront planning | LLM generates a step-by-step plan first, then executes each step sequentially, revising the plan if needed. | Research assistant that gathers data from multiple sources and synthesizes a report. |
| `parallel_fan_out` | Tasks where multiple independent sub-tasks can run simultaneously | Splits the task into independent sub-tasks, executes them in parallel, then aggregates results. | Competitive analysis agent querying 5 competitor APIs simultaneously. |
| `pipeline` | Sequential data transformation workflows | Passes output from one stage as input to the next in a fixed sequence of named stages (`orchestration.stages`, core v0.65.0+). | Content pipeline: draft -> fact-check -> edit -> format. |
| `supervisor` | Coordinating multiple specialized sub-agents | A coordinator that **only** delegates to its workers — the agent's `type: agent` tools — and answers with what they returned. | Project management agent delegating to design, engineering, and QA agents. |
| `swarm` | *(avoid in 0.68 — see below)* | A single-model loop that may emit a JSON "handoff"; in the 0.68 runtime the handoff invokes no agent. | — |
| `glyph` | Deterministic multi-tool procedures (core v0.39.0+) | The plan is a Glyph **program**: the model writes it (expensive) or `spec.program` fixes it ahead of time (zero planning calls). | Fixed lookup chain: find customer -> list devices -> check warranty for each. |

### Conversation awareness (core v0.64.0 / v0.65.0)

Which patterns see the conversation history and the per-turn `prompts.context`:

| Pattern | History (as messages) | `prompts.context` separated for caching | Tool-result budget (v0.66.0) |
|---|---|---|---|
| `react` | yes | yes | yes |
| `glyph` | yes | yes | no — programs get full data |
| `plan_and_execute` | planner and synthesizer (v0.65.0+); each step runs its own tool loop **without** history | yes | yes |
| `parallel_fan_out` | the decomposer and synthesizer (v0.65.0+); subtasks without history | yes | yes |
| `pipeline` | **first stage only** (v0.65.0+) | yes | yes |
| `supervisor` | yes (rewritten on native tools in v0.65.0) | yes | yes |
| `swarm` | **no** — sees only the current query | no — appended to the system prompt, with a load-time warning | no |

Before v0.65.0, `plan_and_execute`, `pipeline` and `parallel_fan_out` did not see
the conversation at all; before v0.64.0 history only reached the model when the
system prompt rendered `memory.conversation`. For a conversational agent (WhatsApp,
chat) on a v0.65.0+ node any pattern except `swarm` keeps context; on older nodes,
use `react`.

### A pattern is not a chain (astromesh core v0.38.1+)

A pattern decides how **one** agent reasons internally. `spec.chain` decides which
**other** agents fire when it finishes. They are orthogonal and compose freely: a
`react` agent can declare a chain, and so can a `supervisor` one.

`chain` is not a valid value for `spec.orchestration.pattern` — the seven above are
the only ones. When a user describes "and then it should…", that is `spec.chain`
(see `schemas/astromesh-v1-agent.md`), not a pattern choice.

Two neighbouring patterns are easy to reach for by mistake:

- **`pipeline`** moves a value through stages *inside a single agent*, with one
  model call per stage. Use it when the stages are steps of one job.
- **`swarm`** is *meant* to hand the conversation from one agent to another at
  runtime, but in core 0.68 the handoff invokes nobody (see the `swarm` section).
  To pass a human along, use `react` with `type: agent` tools.
- **`spec.chain`** fires other agents after this one is done, on conditions you
  declare, and the routing is deterministic and inspectable before it runs.

### Per-role models (astromesh v0.29.0+)

Each pattern requests one or more **named roles** at its decision points, and an agent can bind a different model — and source (local `ollama` or cloud `litellm`) — to each role via `spec.model.default` + `roles`. See `schemas/astromesh-v1-agent.md` for the full schema. Roles per pattern:

| Pattern | Roles requested |
|---------|------------------|
| `react` | `reasoner` |
| `plan_and_execute` | `planner`, `worker`, `synthesizer` |
| `parallel_fan_out` | `planner`, `worker`, `synthesizer` |
| `pipeline` | `stage:<name>` per stage (default: `analyze`, `process`, `synthesize`) |
| `supervisor` | `supervisor` |
| `swarm` | `reasoner` |
| `glyph` | `reasoner` (writes the program), `default` (narration, `ask`) |

Any role an agent doesn't define falls back to `default`. This is what lets a `plan_and_execute` agent plan with a frontier cloud model and execute with a cheap local one.

---

## react

### Description

The default and most common pattern. Implements a Reason-Act-Observe (ReAct) loop where the LLM:
1. **Reasons** about the current state and what action to take.
2. **Acts** by calling a tool or generating a response.
3. **Observes** the tool result.
4. Repeats until the task is complete or `max_iterations` is reached.

### When to Use

- General-purpose conversational agents.
- Agents with a small to moderate number of tools (1-10).
- When the task flow is unpredictable and depends on user input.
- WhatsApp bots and customer-facing agents.

### When NOT to Use

- Tasks that always follow the same sequence of steps (use `pipeline`).
- Tasks requiring 10+ parallel tool calls (use `parallel_fan_out`).
- Multi-agent coordination scenarios (use `supervisor` or `swarm`).

### YAML Example

```yaml
spec:
  orchestration:
    pattern: react
    max_iterations: 10
    timeout_seconds: 30
  tools:
    - type: builtin
      name: web_search
    - type: client
      name: lookup_order
      description: "Look up an order by ID"
      parameters:
        type: object
        properties:
          order_id:
            type: string
        required: [order_id]
```

### Performance Characteristics

- **Latency per turn:** 1-5 LLM calls (varies by task complexity).
- **Token usage:** Moderate; context grows with each iteration.
- **Best max_iterations:** 5-15 for most use cases.

---

## plan_and_execute

### Description

A three-role pattern:
1. **Planning (`planner`):** the LLM writes a plan — capped at **6 steps** (core 0.68).
2. **Execution (`worker`):** each step runs the same tool loop as `react` (tool results go back to the model), without the conversation history.
3. **Synthesis (`synthesizer`):** the final answer from the step results.

Since core v0.65.0 the planner and the synthesizer receive the conversation history and `prompts.context`, so it works for chat agents. There is no re-planning mid-run.

### When to Use

- Complex tasks requiring 5+ steps.
- Research or analysis tasks.
- When you need an audit trail of the reasoning process.

### When NOT to Use

- Simple Q&A or single-tool-call scenarios (overhead of planning is wasteful).
- Real-time conversational agents where latency matters (planning adds a full LLM call upfront).
- Highly parallel workloads (use `parallel_fan_out`).

### YAML Example

```yaml
spec:
  orchestration:
    pattern: plan_and_execute
    max_iterations: 20
    timeout_seconds: 120
  tools:
    - type: builtin
      name: web_search
    - type: builtin
      name: http_request
    - type: client
      name: save_report
      description: "Save the final research report"
      parameters:
        type: object
        properties:
          title:
            type: string
          content:
            type: string
        required: [title, content]
```

### Performance Characteristics

- **Latency per turn:** plan + up to 6 steps (each a tool loop) + synthesis.
- **Token usage:** High; full plan is carried in context.
- **max_iterations:** bounds each step's tool loop, not the whole run — 5-10 is plenty.

---

## parallel_fan_out

### Description

Splits a task into independent sub-tasks (`planner`, capped at **4 subtasks**), runs them concurrently — each one a full tool loop (`worker`) — then aggregates the results (`synthesizer`). If one subtask fails, the others are cancelled (core v0.65.0+). The decomposer and synthesizer see the conversation history since v0.65.0; before that, `fan_out` also discarded tool calls.

### When to Use

- Querying multiple data sources simultaneously.
- Comparative analysis (e.g., comparing products, prices, competitors).
- Batch processing where items are independent.
- When latency reduction from parallelism outweighs the cost of extra tokens.

### When NOT to Use

- Tasks where steps depend on each other (use `pipeline` or `plan_and_execute`).
- Simple single-tool tasks (unnecessary overhead).
- When you have strict cost constraints (parallel calls multiply token usage).

### YAML Example

```yaml
spec:
  orchestration:
    pattern: parallel_fan_out
    max_iterations: 5
    timeout_seconds: 60
  tools:
    - type: agent
      name: search_amazon
      agent: amazon-scraper
      description: "Search Amazon for product pricing"
    - type: agent
      name: search_walmart
      agent: walmart-scraper
      description: "Search Walmart for product pricing"
    - type: agent
      name: search_target
      agent: target-scraper
      description: "Search Target for product pricing"
```

### Performance Characteristics

- **Latency per turn:** Wall-clock time of the slowest sub-task (not the sum).
- **Token usage:** High; N parallel calls each consume tokens.
- **max_iterations:** bounds each subtask's tool loop; 3-5.

---

## pipeline

### Description

A fixed sequence of **named stages** where each stage's answer becomes the next stage's input. Stages are declared in `spec.orchestration.stages` (core v0.65.0+): 2 to 6 names of 1–40 characters; without it the stages are `analyze`, `process`, `synthesize`. An out-of-range `stages` stops the agent from loading.

What a stage gets is its **name** and the previous stage's output — there is no per-stage prompt (`prompts.templates` are not read by the pattern). So the stage names must be self-explanatory (`draft`, `fact_check`, `edit`), and the system prompt should describe what each stage does. Each stage runs the `react` tool loop with all the agent's tools, on the model role `stage:<name>` — bind a different model per stage with `spec.model.roles`. Only the first stage sees the conversation history.

### When to Use

- Content creation workflows (draft, review, edit, publish).
- Data processing chains (extract, transform, validate, load).
- Any workflow that always follows the same steps in order.

### When NOT to Use

- Conversational agents where the flow depends on user input.
- Tasks requiring dynamic re-planning.
- Workflows where stages need to run in parallel.

### YAML Example

```yaml
spec:
  orchestration:
    pattern: pipeline
    stages: [draft, fact_check, edit, format]   # core v0.65.0+
    max_iterations: 4       # per stage: bounds that stage's tool loop
    timeout_seconds: 180
  prompts:
    system: |
      You write blog posts in four stages. In "draft" write 500-800 words on the
      requested topic. In "fact_check" flag claims that need citations (use
      web_search). In "edit" improve clarity, grammar and tone. In "format" return
      Markdown with headings and a TL;DR.
  tools:
    - type: builtin
      name: web_search
```

### Performance Characteristics

- **Latency per turn:** at least one LLM call per stage (more if a stage uses tools).
- **Token usage:** High; each stage processes the full document.
- **max_iterations:** per stage, not total.

---

## supervisor

### Description

A coordinator (model role `supervisor`) that **only delegates**. Since core v0.65.0 its workers are exactly the agent's `type: agent` tools: those are the only tools offered to the model, and a call to anything else is not executed. With no `type: agent` tool the agent does not load (`AgentConfigError`). It sees the conversation history. Any `builtin`/`client`/`integration` tool on a supervisor is dead weight — put it on a worker.

### When to Use

- Coordinating 2-5 specialized agents with different capabilities.
- When quality control over sub-agent output is important.
- Complex workflows where a central coordinator adds value.
- When agents have overlapping capabilities and need routing.

### When NOT to Use

- Simple tasks that a single agent can handle.
- Peer-to-peer handoff scenarios where no central control is needed (use `swarm`).
- When latency is critical (supervisor adds overhead to every interaction).

### YAML Example

```yaml
spec:
  orchestration:
    pattern: supervisor
    max_iterations: 15
    timeout_seconds: 120
  prompts:
    system: |
      You are a project manager agent. You coordinate work between specialized agents.
      Delegate tasks to the most appropriate agent and review their output before
      presenting it to the user.
  tools:
    - type: agent
      name: design_review
      agent: design-agent
      description: "Reviews UI/UX designs and provides feedback"
    - type: agent
      name: code_review
      agent: code-agent
      description: "Reviews code changes and suggests improvements"
    - type: agent
      name: qa_check
      agent: qa-agent
      description: "Runs test scenarios and reports issues"
```

### Performance Characteristics

- **Latency per turn:** 2-10+ LLM calls (supervisor + N child calls + review).
- **Token usage:** Very high; supervisor context includes all child responses.
- **Best max_iterations:** 10-20 depending on number of child agents.

---

## swarm

### Status in core 0.68 — do not emit for new agents

The pattern exists and the name is valid, but read the runtime before choosing it:

- The engine builds `SwarmPattern()` with **no peer agents** (`runtime/engine.py`, `_build_pattern`), so when the model answers with a handoff JSON (`{"handoff": "<agent>", "context": "…"}`) the runtime only relabels the loop — **no other agent is invoked**. The same model keeps answering.
- It sends the model **only the current query** — no conversation history — so a WhatsApp/chat agent on `swarm` forgets everything between messages.
- `prompts.context` is not separated (appended to the system prompt, with a load-time warning), and the v0.66.0 tool-result budget does not apply.
- Core v0.65.0 rewrote `plan_and_execute`, `pipeline`, `parallel_fan_out` and `supervisor`; `swarm` was explicitly left unchanged.

### What to use instead

| The user wants… | Emit |
|---|---|
| A conversation passed from a greeter to a specialist | `react` (or `supervisor`) with one `type: agent` tool per specialist |
| Something to happen *after* this agent finishes | `spec.chain` (core v0.38.1+) |
| A coordinator that only routes | `supervisor` |

If a user explicitly asks for `swarm`, emit it but tell them the three limits above.

---

## glyph

### Description (core v0.39.0+)

The agent's plan is a **program** in the Glyph action language, executed by the runtime instead of a tool-calling loop. Two modes:

1. **Model-written** (`pattern: glyph`, no `spec.program`): the model writes a program each run (role `reasoner`); `orchestration.max_repairs` (default 2) caps recompiles. **Measured at +164% to +2839% the cost of ReAct** — output tokens dominate. Do not use this mode for production.
2. **Fixed** (`spec.program`): written once (typically by running mode 1 once and parameterizing the result with `context.<field>`), reviewed by a human, compiled against the agent's tools when it loads, and run on every request with **zero planning calls**. This is where Glyph pays off.

`orchestration.narrate` (default `true`) adds a model call that turns the program's result into prose; set `narrate: false` when the agent feeds another agent (`spec.chain`, an API consumer) rather than a person.

### When to Use

- A deterministic procedure of 5+ tool calls where each result feeds the next, with independent branches or a "do this for every item" step.
- The agent feeds another system, not a human (`narrate: false`).
- The program can be fixed in `spec.program`.

### When NOT to Use

- One or two tools — the grammar overhead dominates.
- The agent must see the data before deciding what to do next.
- Conversational agents — the fixed cost is paid every turn.
- Reasoning models (huge output writing code).
- The node lacks the `glyph` extra (`astromesh-glyph` is not on PyPI yet). Without it, no-program agents fall back to `react` with a warning; agents **with** `spec.program` do not load.

### YAML Example

```yaml
spec:
  orchestration:
    pattern: glyph
    narrate: false
  program: |
    customer = find_customer(email=context.email)
    devices  = list_devices(customer_id=customer.id)
    return {customer, devices}
  # tools: must register find_customer and list_devices under exactly those names
```

Calls in the program use the **registered** tool names — for `integration` and `api` tools that is `<slug>_<action>` (e.g. `praxis_buscar_records`), not the bare action. `spec.program` with any other pattern, or a program that does not compile against the declared tools, stops the agent from loading.

---

## Decision Flowchart

Use this flowchart to select the right orchestration pattern:

```
START: What kind of task is this?
  |
  |-- Is it a single conversational agent with tools?
  |     YES --> react
  |
  |-- Does the task always follow the same steps in order?
  |     YES --> pipeline
  |
  |-- Does the task require upfront planning with 5+ steps?
  |     YES --> plan_and_execute
  |
  |-- Can the task be split into independent sub-tasks?
  |     YES --> Are there 3+ sub-tasks that can run simultaneously?
  |               YES --> parallel_fan_out
  |               NO  --> react (with multiple tool calls)
  |
  |-- Is it a fixed, deterministic procedure of 5+ tool calls, feeding a system
  |   rather than a person, on a node with the glyph extra?
  |     YES --> glyph + spec.program (narrate: false)
  |
  |-- Does it involve multiple specialized agents?
  |     YES --> Should the coordinator ONLY route to them?
  |               YES --> supervisor
  |               NO  --> react with one `type: agent` tool per specialist
  |
  |-- Should other agents run AFTER this one finishes?
  |     YES --> spec.chain (not a pattern; combine with any of the above)
  |
  |-- Default --> react
```

**Quick rules of thumb:**
- When in doubt, use `react`. It handles most cases well.
- Use `pipeline` only when you can define the exact stages upfront.
- Use `parallel_fan_out` when latency matters and sub-tasks are independent.
- Use `supervisor` when you have 2-5 specialized agents and the coordinator only routes.
- Do not emit `swarm` for new agents in core 0.68 (no real handoff, no history).
- Use `glyph` only with a fixed `spec.program`, never as a model-written default.
- Use `plan_and_execute` for complex research or analysis tasks.
