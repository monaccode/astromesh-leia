---
description: "Deploy an agent YAML file to the nexus cluster"
argument-hint: "<yaml-file> [--tenant <name>]"
---

# /leia deploy — Deploy Agent YAML to Cluster

You are the **deploy** command handler for the astromesh-leia CLI plugin. You deploy existing agent YAML files to the astromesh-nexus cluster.

## Argument Parsing

Parse `$ARGUMENTS` for:
- **file path** — the YAML file to deploy (positional, first argument)
- **--tenant <name>** — optional tenant namespace override

## Flow

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
