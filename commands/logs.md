---
description: "View logs for a deployed agent"
argument-hint: "<agent-name> [--follow] [--lines <n>]"
---

# /leia logs — View Agent Logs

You are the **logs** command handler for the astromesh-leia CLI plugin. You fetch and display logs for a deployed agent.

## Argument Parsing

Parse `$ARGUMENTS` for:
- **agent-name** — positional argument, the name of the agent
- **--follow** or **-f** — optional flag to continuously poll for new logs
- **--lines <n>** or **-n <n>** — optional, number of log lines to fetch (default: 50)

## Flow

### Case 1: Agent Name Provided

#### Step 1 — Fetch Logs

Dispatch **leia-operator** to retrieve logs:

```bash
curl -s -X GET "${NEXUS_URL}/api/v1/agents/<agent-name>/logs?lines=<n>" \
  -H "Authorization: Bearer ${API_KEY}"
```

Display the log output to the user. If the response is empty, inform the user: "No logs available yet for `<agent-name>`. The agent may not have received any requests."

#### Step 2 — Follow Mode

If `--follow` or `-f` is specified:
- After displaying initial logs, poll every 5 seconds for new log entries.
- Display only new lines on each poll (track the last seen timestamp or line count).
- Continue until the user interrupts or says "stop".
- Show a hint: "Following logs for `<agent-name>`... (say 'stop' to end)"

### Case 2: No Agent Name

When no agent name is provided:

1. Dispatch **leia-operator** to list all deployed agents.
2. Display the list and ask the user which agent's logs they want to see:
   ```
   Which agent's logs do you want to view?
     1. marios-pizza (Ready)
     2. support-bot (Ready)
     3. lead-qualifier (Running)
   ```
3. Once the user selects, fetch and display logs for that agent.

## Output Format

Display logs with timestamps and level indicators when available:

```
[2025-01-15 10:30:01] INFO  Agent started
[2025-01-15 10:30:05] INFO  WhatsApp webhook registered
[2025-01-15 10:31:12] INFO  Received message from +1234567890
[2025-01-15 10:31:13] INFO  Generated response (245ms)
[2025-01-15 10:31:13] INFO  Sent reply to +1234567890
```

## Error Handling

- **Agent not found**: "No agent named `<name>` found. Run `/leia status` to see deployed agents."
- **Cluster unreachable**: "Cannot reach the nexus cluster. Check your config in `~/.astromesh-leia/config.yaml`."
- **No logs endpoint**: "The logs endpoint is not available. The agent may still be initializing."
