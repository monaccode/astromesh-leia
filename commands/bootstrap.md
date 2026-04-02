---
description: "Bootstrap a nexus cluster — create Kind local cluster or connect to existing remote cluster"
argument-hint: "[local|remote]"
---

You are handling the `/leia bootstrap` command for setting up a nexus cluster.

## Modes

There are two bootstrap modes:

| Mode | Description |
|---|---|
| `local` | Create a local Kind cluster and deploy nexus |
| `remote` | Connect to an existing remote nexus cluster |

## Parsing $ARGUMENTS

- No arguments: ask the user whether they want `local` or `remote`.
- `local`: proceed with Kind bootstrap flow.
- `remote`: proceed with remote connection flow.

## Local (Kind) Bootstrap Flow

### Step 1 — Check for existing cluster

Run `kubectl cluster-info --context kind-nexus-local` to see if a Kind cluster is already running.

- If it is running, ask the user if they want to tear it down and recreate, or just reconnect.
- If it is not running, proceed to creation.

### Step 2 — Locate nexus repo

Check the current leia config (`~/.astromesh-leia/config.yaml`) for a `nexus-repo` value on any context. Fall back to the default path `D:\monaccode\astromesh-nexus`. Verify the directory exists and contains `hack/bootstrap.sh`.

### Step 3 — Run bootstrap

Execute `hack/bootstrap.sh` from the nexus repo directory. This script creates the Kind cluster and deploys all nexus components. Stream output so the user can follow progress.

### Step 4 — Wait for healthy

Poll the nexus healthz endpoint every 5 seconds with a timeout of 5 minutes:

```bash
curl -sf http://localhost:8080/healthz
```

If the timeout is reached, report failure and suggest checking pod logs with `kubectl get pods -n nexus-system`.

### Step 5 — Generate API key

Once healthy, generate an API key:

```bash
kubectl exec -n nexus-system deploy/nexus-server -- nexus-admin create-key --name leia-cli --scope admin
```

Capture the key from the output.

### Step 6 — Save context

Write or update the `local` context in `~/.astromesh-leia/config.yaml`:

```yaml
current-context: local
contexts:
  local:
    nexus-url: http://localhost:8080
    api-key: <generated-key>
    cluster-type: kind
    cluster-name: nexus-local
    nexus-repo: <path-used>
```

Set `current-context` to `local`.

## Remote Bootstrap Flow

### Step 1 — Ask for URL

Ask the user for the nexus server URL (e.g., `https://nexus.example.com`).

### Step 2 — Test healthz

```bash
curl -sf <url>/healthz
```

If it fails, report the error and ask the user to verify the URL and network access.

### Step 3 — Ask for API key

Ask the user for their API key (should start with `nxk_`).

### Step 4 — Test authentication

```bash
curl -sf -H "Authorization: Bearer <api-key>" <url>/api/v1/tenants
```

If it fails, report the auth error and ask the user to verify the key.

### Step 5 — Save context

Ask the user for a context name (default: hostname from URL). Write the context to config and set it as `current-context`.

## Error Handling

- If `kind` is not installed for local mode, tell the user to install it and provide the link: https://kind.sigs.k8s.io/docs/user/quick-start/#installation
- If `kubectl` is not available, tell the user to install it.
- If the bootstrap script fails, show the last 20 lines of output and suggest checking prerequisites.
- On any failure, do NOT leave a half-written config — only save the context after all steps succeed.
