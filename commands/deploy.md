---
description: "Deploy an agent YAML file to the nexus cluster"
argument-hint: "<yaml-file> [--tenant <name>] [--node <url>]"
---

# /leia deploy — Deploy Agent YAML to Cluster

You are the **deploy** command handler for the astromesh-leia CLI plugin. You deploy existing agent YAML files to the astromesh-nexus cluster.

## Argument Parsing

Parse `$ARGUMENTS` for:
- **file path** — the YAML file to deploy (positional, first argument)
- **--tenant <name>** — optional tenant namespace override

## Flow

### Standalone node target (`--node`)

For a standalone `astromeshd` node (astromesh-node 0.1.10, core 0.60.0+), not Nexus:

1. Copy the YAML to `<config_dir>/agents/<name>.agent.yaml` (system install: `/etc/astromesh`), plus any `rag/*.rag.yaml` and knowledge bundle it needs under the config dir.
2. Reload without restarting: `sudo systemctl reload astromeshd` (SIGHUP; also works with `--foreground`). It calls `AgentRuntime.reload()`, which computes `{added, updated, removed, failed}`; an agent that fails to build stays `draft` with its error and does not affect the others. `runtime.yaml` changes need a restart. Reload is refused when `ASTROMESH_PERSIST_AGENTS=0`.
3. Verify with `GET <node>/v1/agents` and `GET <node>/v1/agents/<name>`; smoke-test with `POST <node>/v1/agents/<name>/run` (the node returns `blocked`/`guardrail` directly).
4. Alternative without disk access: `POST <node>/v1/agents` with the agent as JSON (`201`, `409` if it exists), `PUT /v1/agents/<name>` to update.
5. `astromeshctl validate --path <config_dir>` (from astromesh-node) checks the YAML files before you reload.

### Case 1: File Path Provided

#### Step 1 — Read and Validate YAML

Read the specified YAML file using the Read tool. Validate the following:

| Check | Expected | On Failure |
|---|---|---|
| File exists | File is readable | "File not found: `<path>`. Check the path and try again." |
| `apiVersion` | `astromesh/v1` | "Invalid apiVersion: expected `astromesh/v1`, got `<value>`." |
| `kind` | `Agent` | "Invalid kind: expected `Agent`, got `<value>`." |
| `metadata.name` | Non-empty, RFC 1123 compliant | "Invalid or missing `metadata.name`." |

If the `--tenant` flag is provided, inject or override `metadata.namespace` in the YAML before deploying.

#### Step 2 — Deploy

Dispatch **leia-operator** to deploy the agent:
- POST the YAML to the nexus API endpoint
- Poll status every 5 seconds for up to 60 seconds
- Report progress as it happens

#### Step 3 — Report

Show deployment result:
- Agent name, tenant, current phase
- If `Ready`: display success message and suggest `/leia test <agent-name>`
- If timed out waiting: show current status and suggest `/leia status <agent-name>` to keep monitoring
- If error: show the error details and suggest `/leia logs <agent-name>` or `/leia diagnose`

### Case 2: No File Specified

When `$ARGUMENTS` is empty or contains only flags:

1. Search for `*.agent.yaml` files in the current working directory using Glob.
2. **If files found**: list them and ask the user which one to deploy.
   ```
   Found agent YAML files:
     1. restaurant-booking.agent.yaml
     2. customer-support.agent.yaml
   Which one would you like to deploy? (number or name)
   ```
3. **If no files found**: inform the user and suggest alternatives:
   ```
   No .agent.yaml files found in the current directory.
   - Use /leia create to design a new agent
   - Specify a path: /leia deploy path/to/agent.yaml
   - Browse templates: /leia templates
   ```

## Error Handling

- **Connection refused**: "Cannot reach the nexus cluster. Is it running? Try `/leia status` to check cluster health."
- **401 Unauthorized**: "Authentication failed. Check your API key in `~/.astromesh-leia/config.yaml`."
- **400 Bad Request**: Show the API error body so the user can fix the YAML.
- **409 Conflict**: "An agent with this name already exists. Use a different name or delete the existing agent first."
