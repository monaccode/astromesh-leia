# Nexus REST API Reference

This document describes the astromesh-nexus control plane REST API. All agent lifecycle operations go through this API.

## Base URL

The Nexus API runs on the cluster. Default base URL:

```
http://nexus.astromesh-system.svc.cluster.local:8080
```

For local development with port-forwarding:

```
http://localhost:8080
```

## Authentication

All endpoints except `/healthz` and `/readyz` require authentication via API key.

| Header | Format | Description |
|--------|--------|-------------|
| `X-API-Key` | `nxk_<base64-string>` | Nexus API key. Generated during bootstrap or via `leia config`. |

Unauthenticated requests receive `401 Unauthorized`.

## Error Format

**Single error:**
```json
{
  "error": "agent 'my-agent' not found"
}
```

**Multiple validation errors:**
```json
{
  "errors": [
    "metadata.name is required",
    "spec.model.primary.provider must be one of: ollama, openai, openai_compat, azure_openai"
  ]
}
```

---

## Endpoints

### POST /api/v1/agents

Create a new agent from a YAML/JSON manifest.

| Property | Value |
|----------|-------|
| **Method** | `POST` |
| **Path** | `/api/v1/agents` |
| **Auth** | Required |
| **Content-Type** | `application/json` or `application/x-yaml` |

**Request body:** Full agent manifest (see `schemas/astromesh-v1-agent.md`).

**Response codes:**

| Code | Description |
|------|-------------|
| `201 Created` | Agent created successfully. |
| `400 Bad Request` | Validation errors in the manifest. |
| `401 Unauthorized` | Missing or invalid API key. |
| `409 Conflict` | Agent with this name already exists in the namespace. |

**Response (201):**
```json
{
  "name": "sales-qualifier",
  "namespace": "default",
  "version": "1.0.0",
  "status": "pending",
  "created_at": "2026-04-02T10:30:00Z"
}
```

**curl example:**
```bash
curl -X POST http://localhost:8080/api/v1/agents \
  -H "X-API-Key: nxk_abc123..." \
  -H "Content-Type: application/x-yaml" \
  --data-binary @agent.yaml
```

---

### GET /api/v1/agents

List all agents in the cluster (filtered by namespace if the API key is namespace-scoped).

| Property | Value |
|----------|-------|
| **Method** | `GET` |
| **Path** | `/api/v1/agents` |
| **Auth** | Required |

**Query parameters:**

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `namespace` | string | all | Filter by namespace. |
| `label` | string | -- | Label selector (e.g., `team=sales`). |

**Response codes:**

| Code | Description |
|------|-------------|
| `200 OK` | Array of agent summaries. |
| `401 Unauthorized` | Missing or invalid API key. |

**Response (200):**
```json
[
  {
    "name": "sales-qualifier",
    "namespace": "sales-team",
    "version": "1.0.0",
    "status": "running",
    "created_at": "2026-04-02T10:30:00Z",
    "updated_at": "2026-04-02T10:31:15Z"
  },
  {
    "name": "support-bot",
    "namespace": "default",
    "version": "0.2.0",
    "status": "running",
    "created_at": "2026-04-01T08:00:00Z",
    "updated_at": "2026-04-01T08:01:00Z"
  }
]
```

**curl example:**
```bash
curl http://localhost:8080/api/v1/agents \
  -H "X-API-Key: nxk_abc123..." \
  -G -d "namespace=sales-team"
```

---

### GET /api/v1/agents/:name

Get full details for a specific agent.

| Property | Value |
|----------|-------|
| **Method** | `GET` |
| **Path** | `/api/v1/agents/:name` |
| **Auth** | Required |

**Query parameters:**

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `namespace` | string | `default` | Namespace of the agent. |

**Response codes:**

| Code | Description |
|------|-------------|
| `200 OK` | Full agent manifest and runtime status. |
| `401 Unauthorized` | Missing or invalid API key. |
| `404 Not Found` | Agent does not exist. |

**Response (200):**
```json
{
  "name": "sales-qualifier",
  "namespace": "sales-team",
  "version": "1.0.0",
  "status": "running",
  "manifest": { "...full agent manifest..." },
  "node": "astromesh-node-01",
  "created_at": "2026-04-02T10:30:00Z",
  "updated_at": "2026-04-02T10:31:15Z"
}
```

**curl example:**
```bash
curl http://localhost:8080/api/v1/agents/sales-qualifier \
  -H "X-API-Key: nxk_abc123..." \
  -G -d "namespace=sales-team"
```

---

### PUT /api/v1/agents/:name

Update an existing agent manifest. Triggers a rolling update on the node.

| Property | Value |
|----------|-------|
| **Method** | `PUT` |
| **Path** | `/api/v1/agents/:name` |
| **Auth** | Required |
| **Content-Type** | `application/json` or `application/x-yaml` |

**Request body:** Full updated agent manifest.

**Response codes:**

| Code | Description |
|------|-------------|
| `200 OK` | Agent updated. |
| `400 Bad Request` | Validation errors. |
| `401 Unauthorized` | Missing or invalid API key. |
| `404 Not Found` | Agent does not exist. |

**Response (200):**
```json
{
  "name": "sales-qualifier",
  "namespace": "sales-team",
  "version": "1.1.0",
  "status": "updating",
  "updated_at": "2026-04-02T12:00:00Z"
}
```

**curl example:**
```bash
curl -X PUT http://localhost:8080/api/v1/agents/sales-qualifier \
  -H "X-API-Key: nxk_abc123..." \
  -H "Content-Type: application/x-yaml" \
  -G -d "namespace=sales-team" \
  --data-binary @agent-v2.yaml
```

---

### DELETE /api/v1/agents/:name

Delete an agent. Stops the agent on its node and removes the manifest.

| Property | Value |
|----------|-------|
| **Method** | `DELETE` |
| **Path** | `/api/v1/agents/:name` |
| **Auth** | Required |

**Query parameters:**

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `namespace` | string | `default` | Namespace of the agent. |

**Response codes:**

| Code | Description |
|------|-------------|
| `200 OK` | Agent deleted. |
| `401 Unauthorized` | Missing or invalid API key. |
| `404 Not Found` | Agent does not exist. |

**Response (200):**
```json
{
  "deleted": "sales-qualifier"
}
```

**curl example:**
```bash
curl -X DELETE http://localhost:8080/api/v1/agents/sales-qualifier \
  -H "X-API-Key: nxk_abc123..." \
  -G -d "namespace=sales-team"
```

---

### GET /api/v1/agents/:name/status

Get real-time status proxied from the astromesh-node running the agent.

| Property | Value |
|----------|-------|
| **Method** | `GET` |
| **Path** | `/api/v1/agents/:name/status` |
| **Auth** | Required |

**Response codes:**

| Code | Description |
|------|-------------|
| `200 OK` | Status from node. |
| `401 Unauthorized` | Missing or invalid API key. |
| `404 Not Found` | Agent does not exist. |
| `502 Bad Gateway` | Node is unreachable. |

**Response (200):**
```json
{
  "name": "sales-qualifier",
  "status": "running",
  "uptime_seconds": 3600,
  "active_conversations": 5,
  "model_health": {
    "primary": "healthy",
    "fallback": "healthy"
  },
  "memory_usage_mb": 256,
  "last_message_at": "2026-04-02T11:29:45Z"
}
```

**curl example:**
```bash
curl http://localhost:8080/api/v1/agents/sales-qualifier/status \
  -H "X-API-Key: nxk_abc123..."
```

---

### GET /api/v1/agents/:name/logs

Retrieve agent logs. Returns recent log lines from the node.

| Property | Value |
|----------|-------|
| **Method** | `GET` |
| **Path** | `/api/v1/agents/:name/logs` |
| **Auth** | Required |

**Query parameters:**

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `namespace` | string | `default` | Namespace. |
| `lines` | integer | `100` | Number of log lines to return. |
| `since` | string | -- | RFC 3339 timestamp; return logs after this time. |
| `level` | string | all | Filter by log level: `debug`, `info`, `warn`, `error`. |

**Response codes:**

| Code | Description |
|------|-------------|
| `200 OK` | Log lines. |
| `401 Unauthorized` | Missing or invalid API key. |
| `404 Not Found` | Agent does not exist. |

**Response (200):**
```json
{
  "agent": "sales-qualifier",
  "lines": [
    {"timestamp": "2026-04-02T11:29:00Z", "level": "info", "message": "Conversation started: conv_abc123"},
    {"timestamp": "2026-04-02T11:29:05Z", "level": "info", "message": "Tool called: score_lead"},
    {"timestamp": "2026-04-02T11:29:06Z", "level": "info", "message": "Lead scored: 78/100"}
  ]
}
```

**curl example:**
```bash
curl http://localhost:8080/api/v1/agents/sales-qualifier/logs \
  -H "X-API-Key: nxk_abc123..." \
  -G -d "lines=50" -d "level=error"
```

---

### GET /api/v1/agents/:name/metrics

Retrieve agent performance metrics from the node.

| Property | Value |
|----------|-------|
| **Method** | `GET` |
| **Path** | `/api/v1/agents/:name/metrics` |
| **Auth** | Required |

**Query parameters:**

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `namespace` | string | `default` | Namespace. |
| `period` | string | `1h` | Time window: `5m`, `15m`, `1h`, `24h`, `7d`. |

**Response codes:**

| Code | Description |
|------|-------------|
| `200 OK` | Metrics object. |
| `401 Unauthorized` | Missing or invalid API key. |
| `404 Not Found` | Agent does not exist. |

**Response (200):**
```json
{
  "agent": "sales-qualifier",
  "period": "1h",
  "conversations": {
    "total": 42,
    "active": 5,
    "completed": 37
  },
  "latency_ms": {
    "p50": 850,
    "p95": 2100,
    "p99": 4500
  },
  "tokens": {
    "input": 125000,
    "output": 45000,
    "total": 170000
  },
  "cost_usd": 1.23,
  "tool_calls": {
    "score_lead": 38,
    "http_request": 12,
    "lookup_pricing": 5
  },
  "errors": 2
}
```

**curl example:**
```bash
curl http://localhost:8080/api/v1/agents/sales-qualifier/metrics \
  -H "X-API-Key: nxk_abc123..." \
  -G -d "period=24h"
```

---

### GET /healthz

Liveness probe. Always returns 200 if the API process is running.

| Property | Value |
|----------|-------|
| **Method** | `GET` |
| **Path** | `/healthz` |
| **Auth** | Not required |

**Response codes:**

| Code | Description |
|------|-------------|
| `200 OK` | API is alive. |

**Response (200):**
```json
{
  "status": "ok"
}
```

**curl example:**
```bash
curl http://localhost:8080/healthz
```

---

### GET /readyz

Readiness probe. Returns 200 when the API can serve requests (database connected, nodes reachable). Returns 503 during startup or when dependencies are unhealthy.

| Property | Value |
|----------|-------|
| **Method** | `GET` |
| **Path** | `/readyz` |
| **Auth** | Not required |

**Response codes:**

| Code | Description |
|------|-------------|
| `200 OK` | API is ready to serve. |
| `503 Service Unavailable` | API is not ready (starting up or dependency failure). |

**Response (200):**
```json
{
  "status": "ready",
  "checks": {
    "database": "ok",
    "nodes": "ok"
  }
}
```

**Response (503):**
```json
{
  "status": "not_ready",
  "checks": {
    "database": "ok",
    "nodes": "failing"
  }
}
```

**curl example:**
```bash
curl http://localhost:8080/readyz
```
