# Knowledge Bundle (OKF) Authoring

Sources: `astromesh-okf` (OKF v0.2 reader, origin/main) and core 0.70.0
(`astromesh/runtime/engine.py` `_resolve_bundle`, `astromesh/knowledge/`). The agent side is
`spec.knowledge.bundle` (core v0.68.0+); this file covers the bundle itself.

## Layout

A bundle is a directory of Markdown files. A concept's id is its path without `.md`, with forward slashes.

```
support/
  index.md                 # optional root index; may carry okf_version: "0.2"
  policies/
    refunds.md             # concept "policies/refunds"
    refunds-2024.md
  products/
    plans.md
```

`index.md` and `log.md` are reserved names, never concepts. A directory's `index.md` is what `kb_index`
returns for it; without one the library synthesizes a listing. `log.md` is not read by any tool.

## Concept file

```markdown
---
type: Policy                    # the ONLY required field
title: Refund policy
description: Customers can ask for a full refund within 30 days of purchase.
tags: [support, refunds]
status: stable                  # stable (default) | draft | deprecated
verified: { by: human:support-lead, at: 2026-09-01T12:00:00Z }
stale_after: 2030-09-01T00:00:00Z
---

A full refund is available within 30 days of purchase. See [plans](/products/plans.md).
```

- `type`, `title` (default: file name), `description`, `tags`, `status`, `verified`, `stale_after` are
  what the tools use. `generated` and your own keys pass through in `extra`.
- `verified` is one `{by, at}` event or a list. `trust` is `human-reviewed` if any `by` starts with
  `human:`, `machine-confirmed` for any other `by`, else `unverified`.
- `stale_after` (date, or datetime; no timezone means UTC): from then on the concept is `stale`.
- A file with no parseable frontmatter mapping, or no `type`, is invalid; indexes and search skip it.
- **YAML anchors and aliases (`&x`, `*x`, `<<:`) are rejected**: the concept is invalid.
- Links are ordinary Markdown links; `/x.md` is bundle-absolute, others are relative to the concept.
  Links that leave the bundle are dropped.

## What the model sees

`kb_index` (directory listing), `kb_read` (one concept, with `trust`, `stale`, `status`, `links` and
`warnings` such as `DEPRECATED: ...`, `STALE since ...`, `DRAFT: ...`), `kb_find` (search; leaves
deprecated concepts out by default). Signals annotate; they never hide a concept from `kb_read`/`kb_index`.
Retire a policy by setting `status: deprecated` and linking its replacement, not by deleting it.

## Wiring it to an agent

```yaml
spec:
  knowledge:
    bundle: knowledge/support      # relative to the node's config dir, or absolute
  prompts:
    system: |
      Answer only from the knowledge bundle. Never rely on a DEPRECATED concept;
      mention when a concept is STALE or DRAFT.
      {{ bundle_index }}
```

If the path is not a directory the agent still starts, without the bundle, and logs a warning.

## Where it works

- **Standalone node** (`astromeshd`): put the bundle under the config dir (e.g. `/etc/astromesh/knowledge/support/`)
  and reference it. The bundle is resolved when the agent is built, so put it in place before deploying or reloading the agent.
- **Nexus runtime pool: not today.** The pool deployment mounts only `runtime.yaml`; Nexus 0.26.9-0.26.11
  changelogs state the pool mounts no bundles, so an agent that declares one starts without it. Do not
  promise bundles on Nexus; offer `knowledge.pipeline` (RAG) or a standalone node instead.

Inspect a bundle outside the runtime with the `astromesh-okf` CLI: `astromesh-okf index|read|find <bundle>`
(and `mcp` to serve it over MCP).
