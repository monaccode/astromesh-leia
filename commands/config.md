---
description: "Manage nexus cluster connection contexts (add, list, switch, show, set defaults)"
argument-hint: "[list|use|add|show|set] [context-name|key value]"
---

You are handling the `/leia config` command for managing nexus cluster connection contexts.

## Config File

Location: `~/.astromesh-leia/config.yaml`

Expected format:

```yaml
current-context: local
contexts:
  local:
    nexus-url: http://localhost:8080
    api-key: nxk_...
    cluster-type: kind
    cluster-name: nexus-local
    nexus-repo: D:\monaccode\astromesh-nexus
defaults:
  channel: whatsapp
  model-provider: auto
  tenant: default
```

## Parsing $ARGUMENTS

Parse the first word of `$ARGUMENTS` to determine the subcommand:

| Input | Action |
|---|---|
| (empty or `show`) | Show current config |
| `list` | List all contexts |
| `use <name>` | Switch current-context |
| `add <name>` | Add a new context interactively |
| `set <key> <value>` | Set a config value using dot notation |

## Implementation

Use the `Read` tool to parse `~/.astromesh-leia/config.yaml` and the `Write` tool to update it. If the file or directory does not exist, create them with sensible defaults.

### Subcommand: show (default)

Read the config file and display:
- Current context name
- The active context's details (nexus-url, cluster-type, cluster-name)
- Current defaults

Format the output as a readable summary, not raw YAML.

### Subcommand: list

Read the config and list all context names, marking the current one with `*`.

### Subcommand: use <name>

- Validate the named context exists in the config.
- Update `current-context` to the given name.
- Write the updated config back.
- Confirm the switch to the user.

### Subcommand: add <name>

Interactively ask the user for:
1. `nexus-url` (required)
2. `api-key` (required)
3. `cluster-type` — `kind` or `remote` (default: `remote`)
4. `cluster-name` (default: same as context name)
5. `nexus-repo` (optional, only relevant for kind clusters)

Add the new context to the config and ask if the user wants to switch to it.

### Subcommand: set <key> <value>

Support dot-notation keys to set any value in the config:
- `defaults.channel whatsapp` sets `defaults.channel`
- `contexts.local.nexus-url http://...` sets a context field

Read the config, apply the change, write it back, and confirm.

## Error Handling

- If the config file does not exist, create `~/.astromesh-leia/` directory and a default config with an empty contexts map and default defaults.
- If a referenced context does not exist, report the error and list available contexts.
- If `$ARGUMENTS` does not match a known subcommand, show usage help.
