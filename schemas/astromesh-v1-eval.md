# Eval Manifest (`kind: Eval`)

Source: astromesh core 0.70.0 (`astromesh/evals/formato.py`, `__main__.py`;
`vscode-extension/schemas/eval.schema.json`). Needs **core v0.67.0+**.

An eval is a YAML file next to the agent's config that replays conversations against the agent and
asserts on the result. It is an author/CI tool: it runs **locally against a config directory**
(it boots an `AgentRuntime`), not against a remote Nexus, and the Nexus pool does not run it.

## Where it lives and how to run it

```
config/
  agents/support.agent.yaml
  evals/support.eval.yaml        # default location: <config_dir>/evals/*.eval.yaml
```

```bash
astromesh-eval <config_dir> [file.eval.yaml ...] [--out report.json]
```

Exit codes: `0` every eval meets its thresholds, `1` some eval is under threshold, `2` load error
(nothing was run). A case that raises counts as failed; it never aborts the eval.

## Format

```yaml
apiVersion: astromesh/v1
kind: Eval
metadata:
  name: support-smoke
spec:
  agent: support                  # required: metadata.name of an agent in the config dir
  tools_default: block            # real (default) | block
  judge:                          # optional; required by any case that has a rubric
    model: {provider: ollama, model: llama3.1:8b, endpoint: http://localhost:11434}
    pass_score: 0.7               # 0..1, default 0.7
  thresholds:
    pass_rate: 0.8                # required, 0..1
    max_avg_tokens: 3000          # optional, integer >= 1
  cases:
    - id: refund-window           # [a-z0-9_-], up to 64, unique
      turns: ["How long do I have to ask for a refund?"]   # non-empty list of user messages
      context: {}                 # optional map passed as run context
      tools:                      # optional fixtures: tool name -> canned result
        lookup_order: {status: shipped}
      expect:                     # each item is a map with ONE assert key
        - contains: "30 days"
        - not_contains: "I cannot"
        - tool_not_called: issue_refund
      rubric: "Answers in a friendly tone and cites the 30-day window."
  cases_file: cases.jsonl         # optional: more cases, one JSON object per line, same shape
```

Asserts (value is always a non-empty string): `contains`, `not_contains`, `regex` (must compile),
`equals`, `tool_called`, `tool_not_called`.

## Rules the loader enforces

- `apiVersion: astromesh/v1`, `kind: Eval`, `metadata.name`, `spec.agent`, `spec.thresholds.pass_rate` are required.
- Unknown keys at any level are an error (exit 2).
- A case needs `expect` or `rubric` (otherwise it cannot fail); a `rubric` needs `spec.judge`.
- At least one case, from `cases` and/or `cases_file`; case ids are unique.
- `tools_default: block` makes a tool without a fixture return an error instead of running for real,
  so evals do not write to real systems. Prefer `block` for agents with write tools.
- Conversational memory is isolated per eval; the confirmation gate and tool-result trimming still apply.

## Authoring guidance

Generate one eval per agent with 4-6 cases: a happy path, a tool-use case (with fixtures), an
out-of-scope case, and a guardrail probe when the agent has `spec.guardrails`. For a guardrail probe,
assert on the rejection text with `contains` (the rule's `message`, default `I can't help with that
request.`), because the eval sees the blocked reply as the answer.
