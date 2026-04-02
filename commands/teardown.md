---
description: "Destroy local Kind cluster or disconnect remote context"
argument-hint: "[context-name]"
---

You are handling the `/leia teardown` command for destroying a local Kind cluster or disconnecting a remote context.

## Parsing $ARGUMENTS

- No arguments: use the current context from `~/.astromesh-leia/config.yaml`.
- Context name provided: use that specific context.

## Flow

### Step 1 — Read config

Read `~/.astromesh-leia/config.yaml` and resolve the target context. If the context does not exist, report an error and list available contexts.

### Step 2 — Determine context type

Check the `cluster-type` field of the target context to decide which teardown path to follow.

### Step 3a — Kind (local) teardown

This is a **destructive operation**. Before proceeding:

1. **CONFIRM with the user**: clearly state that this will destroy the local Kind cluster, delete all data, and remove the context from config. Ask for explicit confirmation (e.g., "Type 'yes' to confirm").
2. Locate the nexus repo from the context's `nexus-repo` field or the default `D:\monaccode\astromesh-nexus`.
3. Run `hack/teardown.sh` from the nexus repo directory.
4. Remove the context entry from the config file.
5. If the torn-down context was `current-context`, update `current-context`:
   - If other contexts exist, switch to the first available one.
   - If no contexts remain, set `current-context` to empty string.
6. Write the updated config.

### Step 3b — Remote teardown

This will **NOT** touch the remote cluster. Before proceeding:

1. **CONFIRM with the user**: explain that this only removes the local connection config and does not affect the remote cluster. Ask for explicit confirmation.
2. Remove the context entry from the config file.
3. If the torn-down context was `current-context`, update `current-context` the same way as the Kind path.
4. Write the updated config.

## Report

After teardown completes, report:
- What was removed (context name, cluster type).
- Whether `current-context` was changed and to what.
- For Kind: confirm the cluster was destroyed.
- For remote: confirm only the local config was removed, the remote cluster is untouched.

## Error Handling

- If config file does not exist, report that no contexts are configured and suggest running `/leia bootstrap` first.
- If the teardown script fails, show the error output and suggest manual cleanup with `kind delete cluster --name nexus-local`.
- Never silently delete config — always confirm with the user first.
