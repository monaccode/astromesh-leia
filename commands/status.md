---
description: "Show cluster status dashboard — tenants, agents, health"
argument-hint: "[agent-name] [--tenant <name>]"
---

# /leia status — Cluster Status Dashboard

You are the **status** command handler for the astromesh-leia CLI plugin. You display a CLI dashboard showing cluster health, tenants, and agents.

## Argument Parsing

Parse `$ARGUMENTS` for:
- **agent-name** — optional positional argument for a specific agent
- **--tenant <name>** — optional filter to show only agents in a specific tenant

## Flow

Dispatch **leia-operator** to fetch all required data from the cluster.

### Data Collection

Gather the following via leia-operator:

1. **Cluster health** — `GET /healthz` and `GET /readyz`
2. **Agents list** — `GET /api/v1/agents` (with tenant filter if `--tenant` specified)
3. **Tenants** — `kubectl get nexustenant -n nexus-system -o wide`

### Case 1: No Agent Name (Full Dashboard)

Display a complete status dashboard with three sections:

#### Cluster Line

```
Cluster: astromesh-nexus | Type: kind | Health: OK
```

If health check fails, show `Health: DEGRADED` or `Health: UNREACHABLE` and suggest `/leia diagnose`.

#### Tenants Table

```
TENANT          PHASE    AGENTS   NODE ENDPOINT
default         Active   3        ws://node-default:9090
production      Active   5        ws://node-prod:9090
staging         Active   1        ws://node-staging:9090
```

#### Agents Table

```
AGENT                  TENANT      PHASE    CHANNEL    LAST SYNCED
marios-pizza           default     Ready    whatsapp   2m ago
support-bot            default     Ready    web        5m ago
lead-qualifier         production  Running  whatsapp   30s ago
```

### Case 2: Specific Agent (Detailed View)

When an agent name is provided in `$ARGUMENTS`, show a detailed single-agent view:

```
Agent: marios-pizza
  Tenant:     default
  Phase:      Ready
  Channel:    whatsapp
  Created:    2025-01-15T10:30:00Z
  Last Synced: 2025-01-15T10:32:00Z (2m ago)
  Node Ack:   true

  Conditions:
    TYPE           STATUS   REASON              MESSAGE
    Validated      True     ValidationPassed    Schema validation passed
    Deployed       True     DeploymentReady     Agent deployed to node
    ChannelReady   True     WebhookRegistered   WhatsApp webhook active
```

If `--tenant` is also provided, use it to disambiguate agents with the same name across tenants.

## Empty States

- **No tenants**: "No tenants found. The cluster may need bootstrapping. Try `/leia bootstrap`."
- **No agents**: "No agents deployed yet. Create one with `/leia create`."
- **No agents matching filter**: "No agents found in tenant `<name>`. Check the tenant name or list all with `/leia status`."

## Error Handling

- **Cluster unreachable**: "Cannot connect to the nexus cluster. Check that it is running and your config is correct in `~/.astromesh-leia/config.yaml`."
- **Partial failure**: If health check fails but agent list succeeds (or vice versa), show what you can and note the failures.
