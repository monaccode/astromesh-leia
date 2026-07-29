# Orchestration Patterns Reference

This document describes the six orchestration patterns available for astromesh agents. The architect subagent uses this reference to select the right pattern for each use case.

## Pattern Overview

| Pattern | Best For | How It Works | Example Use Case |
|---------|----------|--------------|------------------|
| `react` | General-purpose conversational agents | Reason-Act-Observe loop: the LLM reasons about what to do, calls a tool, observes the result, and repeats. | Customer support bot answering questions and looking up orders. |
| `plan_and_execute` | Complex multi-step tasks requiring upfront planning | LLM generates a step-by-step plan first, then executes each step sequentially, revising the plan if needed. | Research assistant that gathers data from multiple sources and synthesizes a report. |
| `parallel_fan_out` | Tasks where multiple independent sub-tasks can run simultaneously | Splits the task into independent sub-tasks, executes them in parallel, then aggregates results. | Competitive analysis agent querying 5 competitor APIs simultaneously. |
| `pipeline` | Sequential data transformation workflows | Passes output from one stage as input to the next in a fixed sequence. Each stage has a defined role. | Content pipeline: draft -> fact-check -> edit -> format -> publish. |
| `supervisor` | Coordinating multiple specialized sub-agents | A supervisor agent delegates tasks to specialized child agents, reviews their output, and synthesizes the final response. | Project management agent delegating to design, engineering, and QA agents. |
| `swarm` | Dynamic, peer-to-peer agent collaboration | Agents hand off conversations to each other based on context. No central coordinator; each agent decides when to transfer. | Sales team: greeter -> qualifier -> technical-demo -> closer. |

### A pattern is not a chain (astromesh core v0.38.1+)

A pattern decides how **one** agent reasons internally. `spec.chain` decides which
**other** agents fire when it finishes. They are orthogonal and compose freely: a
`react` agent can declare a chain, and so can a `supervisor` one.

`chain` is not a valid value for `spec.orchestration.pattern` — the six above are
the only ones. When a user describes "and then it should…", that is `spec.chain`
(see `schemas/astromesh-v1-agent.md`), not a pattern choice.

Two neighbouring patterns are easy to reach for by mistake:

- **`pipeline`** moves a value through stages *inside a single agent*, with one
  model call per stage. Use it when the stages are steps of one job.
- **`swarm`** hands the *conversation* from one agent to another, and the target
  is chosen by the model at runtime. Use it when a human is being passed along.
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

A two-phase pattern:
1. **Planning phase:** The LLM generates a structured plan with numbered steps.
2. **Execution phase:** Each step is executed sequentially. After each step, the LLM can revise the remaining plan based on results.

### When to Use

- Complex tasks requiring 5+ steps.
- Research or analysis tasks.
- When you need an audit trail of the reasoning process.
- When tasks may need dynamic re-planning.

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

- **Latency per turn:** 3-20+ LLM calls (plan + N execution steps).
- **Token usage:** High; full plan is carried in context.
- **Best max_iterations:** 15-30 for research tasks.

---

## parallel_fan_out

### Description

Splits a task into independent sub-tasks, executes them in parallel across multiple LLM calls or tool invocations, then aggregates the results into a single response.

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
- **Best max_iterations:** 3-5 (fan-out + aggregation).

---

## pipeline

### Description

A fixed sequence of stages where each stage's output becomes the next stage's input. Each stage can use different tools or prompts. The pipeline proceeds linearly from start to finish.

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
    max_iterations: 5       # One iteration per stage
    timeout_seconds: 180
  prompts:
    templates:
      stage_draft: "Draft a blog post about {{topic}}. Write 500-800 words."
      stage_factcheck: "Fact-check the following article. Flag any claims that need citations:\n\n{{draft}}"
      stage_edit: "Edit the following article for clarity, grammar, and tone:\n\n{{factchecked}}"
      stage_format: "Format the following article in Markdown with proper headings and a TL;DR section:\n\n{{edited}}"
  tools:
    - type: builtin
      name: web_search
```

### Performance Characteristics

- **Latency per turn:** N LLM calls where N = number of stages.
- **Token usage:** High; each stage processes the full document.
- **Best max_iterations:** Equal to the number of pipeline stages.

---

## supervisor

### Description

A central supervisor agent receives the task, delegates sub-tasks to specialized child agents, reviews their output, and synthesizes the final response. The supervisor controls the workflow and can re-delegate or request revisions.

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

### Description

A decentralized pattern where agents hand off conversations to each other based on context. There is no central coordinator. Each agent decides autonomously whether to respond or transfer the conversation to a more appropriate peer.

### When to Use

- Sales funnels (greeter -> qualifier -> demo -> closer).
- Multi-department routing (support -> billing -> technical).
- Scenarios where conversation context determines which agent should respond.
- When you want agents to specialize without centralized orchestration overhead.

### When NOT to Use

- When you need a single agent to maintain full context of the interaction.
- When strict workflow ordering is required (use `pipeline`).
- When a supervisor needs to review all sub-agent output before responding.
- Small teams (2 agents) where supervisor is simpler.

### YAML Example

```yaml
# Agent 1: Greeter (entry point)
spec:
  orchestration:
    pattern: swarm
    max_iterations: 5
    timeout_seconds: 30
  prompts:
    system: |
      You are the greeter for Acme Corp. Welcome users and understand their intent.
      If they want to buy, transfer to sales-qualifier.
      If they need help, transfer to support-agent.
  tools:
    - type: agent
      name: transfer_to_sales
      agent: sales-qualifier
      description: "Transfer to sales qualification"
    - type: agent
      name: transfer_to_support
      agent: support-agent
      description: "Transfer to customer support"

# Agent 2: Sales Qualifier
spec:
  orchestration:
    pattern: swarm
    max_iterations: 10
    timeout_seconds: 30
  prompts:
    system: |
      You qualify sales leads. Ask about company size, budget, and timeline.
      If qualified, transfer to demo-agent.
      If not qualified, politely redirect to self-serve resources.
  tools:
    - type: agent
      name: transfer_to_demo
      agent: demo-agent
      description: "Transfer to product demo agent"
    - type: client
      name: score_lead
      description: "Score lead quality"
      parameters:
        type: object
        properties:
          score:
            type: integer
        required: [score]
```

### Performance Characteristics

- **Latency per turn:** 1-3 LLM calls per agent in the chain.
- **Token usage:** Moderate per agent; context resets on handoff (only transfer context is passed).
- **Best max_iterations:** 3-10 per agent (each agent has its own iteration budget).

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
  |-- Does it involve multiple specialized agents?
  |     YES --> Do agents need a central coordinator for quality control?
  |               YES --> supervisor
  |               NO  --> Is it a conversation routing / handoff scenario?
  |                         YES --> swarm
  |                         NO  --> supervisor
  |
  |-- Default --> react
```

**Quick rules of thumb:**
- When in doubt, use `react`. It handles most cases well.
- Use `pipeline` only when you can define the exact stages upfront.
- Use `parallel_fan_out` when latency matters and sub-tasks are independent.
- Use `supervisor` when you have 2-5 specialized agents that need coordination.
- Use `swarm` when agents should hand off conversations dynamically.
- Use `plan_and_execute` for complex research or analysis tasks.
