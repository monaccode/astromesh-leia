# Nexus REST API Reference

This document describes the astromesh-nexus control plane REST API. All agent lifecycle operations go through this API.

> **Version:** This reference reflects **astromesh-nexus v0.26.11** (pool on astromesh core 0.70.0; `Blocked`/`Guardrail` persistence and the `timeout` status are Nexus 0.26.13+, core 0.71.0), re-derived from the router (`internal/server/server.go`) and handlers. The sections up to *Agent Endpoints* date from v0.3.0 and still hold; see *Execution, Platform and Operator Endpoints (Nexus 0.26)* for what was added.

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

Nexus v0.3.0 protects routes with a **dual-auth** model (`DualAuthMiddleware`): a protected
endpoint accepts **either** of two credentials. The `/auth/*` routes and the `/healthz` /
`/readyz` probes require **no** authentication.

| Method | Header | Format | Use case |
|--------|--------|--------|----------|
| **API key** | `X-API-Key` | `nxk_<key>` | Programmatic / machine access. Tenant-scoped: the key resolves directly to one tenant. Created via the API-key endpoints and shown **once** at creation. |
| **JWT Bearer** | `Authorization` | `Bearer <accessToken>` | Interactive user sessions. Issued by `/auth/login` (and `/auth/refresh`). |

When authenticating with a JWT, you may add an optional `X-Tenant-ID: <tenantId>` header to scope
the request to one of your tenants (required for tenant-scoped operations such as the agent
endpoints). Nexus verifies that the authenticated user owns that tenant; if not it returns
`403 Forbidden`.

The middleware tries the `Authorization: Bearer` token first (validated **only as a JWT**), then
falls back to `X-API-Key`. **An `nxk_` API key sent as `Authorization: Bearer nxk_...` is not a
JWT and is rejected with `401`** — API keys travel only in `X-API-Key`. A protected request with
neither a valid JWT nor a valid API key receives `401 Unauthorized`.

Leia always sends its configured key as `X-API-Key: ${API_KEY}`.

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

## Authentication Endpoints

These endpoints bootstrap and maintain user sessions. `/auth/*` routes require no auth; `/api/v1/me`
accepts either credential.

Access tokens are **short-lived**. When one expires, call `/auth/refresh` with the refresh token to
obtain a new access token rather than re-prompting for the password.

| Method | Path | Auth | Purpose |
|--------|------|------|---------|
| `POST` | `/auth/register` | none | Register a user. |
| `POST` | `/auth/login` | none | Login and obtain tokens. |
| `POST` | `/auth/refresh` | none | Exchange a refresh token for a new access token. |
| `GET` | `/api/v1/me` | JWT or API key | Current authenticated user / tenant context. |

### POST /auth/register

Register a new user account.

| Property | Value |
|----------|-------|
| **Method** | `POST` |
| **Path** | `/auth/register` |
| **Auth** | Not required |
| **Content-Type** | `application/json` |

**Request body:**
```json
{
  "email": "alice@example.com",
  "password": "correct-horse-battery",
  "displayName": "Alice"
}
```

**Response codes:**

| Code | Description |
|------|-------------|
| `201 Created` | User created. |
| `400 Bad Request` | Invalid body (e.g. missing fields, password too short). |
| `409 Conflict` | Email already registered. |

**Response (201):** Registration also logs the user in — it returns the same token bundle as `/auth/login`, so you can use the `accessToken` immediately without a separate login call.
```json
{
  "accessToken": "eyJhbGciOi...",
  "refreshToken": "eyJhbGciOi...",
  "user": {
    "id": "usr_1f2e...",
    "email": "alice@example.com",
    "displayName": "Alice"
  }
}
```

**curl example:**
```bash
curl -X POST http://localhost:8080/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email":"alice@example.com","password":"correct-horse-battery","displayName":"Alice"}'
```

---

### POST /auth/login

Authenticate with email and password and receive tokens.

| Property | Value |
|----------|-------|
| **Method** | `POST` |
| **Path** | `/auth/login` |
| **Auth** | Not required |
| **Content-Type** | `application/json` |

**Request body:**
```json
{
  "email": "alice@example.com",
  "password": "correct-horse-battery"
}
```

**Response codes:**

| Code | Description |
|------|-------------|
| `200 OK` | Tokens issued. |
| `400 Bad Request` | Invalid body. |
| `401 Unauthorized` | Invalid credentials. |

**Response (200):**
```json
{
  "accessToken": "eyJhbGciOi...",
  "refreshToken": "rt_9c8b...",
  "user": {
    "id": "usr_1f2e...",
    "email": "alice@example.com",
    "displayName": "Alice"
  }
}
```

**curl example:**
```bash
curl -X POST http://localhost:8080/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"alice@example.com","password":"correct-horse-battery"}'
```

---

### POST /auth/refresh

Exchange a valid refresh token for a fresh access token. Use this when an access token expires.

| Property | Value |
|----------|-------|
| **Method** | `POST` |
| **Path** | `/auth/refresh` |
| **Auth** | Not required (the refresh token is the credential) |
| **Content-Type** | `application/json` |

**Request body:**
```json
{
  "refreshToken": "rt_9c8b..."
}
```

**Response codes:**

| Code | Description |
|------|-------------|
| `200 OK` | New access token issued. |
| `400 Bad Request` | Invalid body. |
| `401 Unauthorized` | Invalid or expired refresh token. |

**Response (200):**
```json
{
  "accessToken": "eyJhbGciOi..."
}
```

**curl example:**
```bash
curl -X POST http://localhost:8080/auth/refresh \
  -H "Content-Type: application/json" \
  -d '{"refreshToken":"rt_9c8b..."}'
```

---

### GET /api/v1/me

Return the current authenticated user / tenant context. Works with either a JWT or an API key.

| Property | Value |
|----------|-------|
| **Method** | `GET` |
| **Path** | `/api/v1/me` |
| **Auth** | JWT or API key |

**Response codes:**

| Code | Description |
|------|-------------|
| `200 OK` | Caller context. |
| `401 Unauthorized` | Missing or invalid credentials. |

**Response (200):**
```json
{
  "id": "usr_1f2e...",
  "email": "alice@example.com",
  "displayName": "Alice"
}
```

**curl example:**
```bash
curl http://localhost:8080/api/v1/me \
  -H "Authorization: Bearer eyJhbGciOi..."
```

---

## Tenant Management

A **tenant** is the unit of isolation in Nexus. Creating one provisions a dedicated Kubernetes
namespace and an astromesh-node, and all of the tenant's agents live in that namespace. A tenant's
CR name / namespace has the form `tenant-<uuid>`. These endpoints require **JWT** auth (they act on
behalf of the logged-in user, who becomes the tenant `owner`).

| Method | Path | Auth | Purpose |
|--------|------|------|---------|
| `POST` | `/api/v1/tenants` | JWT | Create a tenant (also creates its namespace + astromesh-node). |
| `GET` | `/api/v1/tenants` | JWT | List the caller's tenants. |
| `DELETE` | `/api/v1/tenants/:id` | JWT | Delete a tenant and its namespace. |

### POST /api/v1/tenants

| Property | Value |
|----------|-------|
| **Method** | `POST` |
| **Path** | `/api/v1/tenants` |
| **Auth** | JWT |
| **Content-Type** | `application/json` |

**Request body:**
```json
{
  "displayName": "Acme Sales",
  "nodeProfile": "standard"
}
```

**Response codes:**

| Code | Description |
|------|-------------|
| `201 Created` | Tenant created; namespace `tenant-<uuid>` provisioned. |
| `400 Bad Request` | Invalid body. |
| `401 Unauthorized` | Missing or invalid JWT. |

**Response (201):**
```json
{
  "id": "t_3a4b...",
  "crName": "tenant-3a4b...",
  "displayName": "Acme Sales"
}
```

**curl example:**
```bash
curl -X POST http://localhost:8080/api/v1/tenants \
  -H "Authorization: Bearer eyJhbGciOi..." \
  -H "Content-Type: application/json" \
  -d '{"displayName":"Acme Sales","nodeProfile":"standard"}'
```

---

### GET /api/v1/tenants

List the tenants the authenticated user belongs to.

| Property | Value |
|----------|-------|
| **Method** | `GET` |
| **Path** | `/api/v1/tenants` |
| **Auth** | JWT |

**Response codes:**

| Code | Description |
|------|-------------|
| `200 OK` | Array of the caller's tenants. |
| `401 Unauthorized` | Missing or invalid JWT. |

**Response (200):**
```json
[
  {
    "id": "t_3a4b...",
    "crName": "tenant-3a4b...",
    "displayName": "Acme Sales"
  }
]
```

**curl example:**
```bash
curl http://localhost:8080/api/v1/tenants \
  -H "Authorization: Bearer eyJhbGciOi..."
```

---

### DELETE /api/v1/tenants/:id

Delete a tenant and its Kubernetes namespace. The caller must own the tenant.

| Property | Value |
|----------|-------|
| **Method** | `DELETE` |
| **Path** | `/api/v1/tenants/:id` |
| **Auth** | JWT |

**Response codes:**

| Code | Description |
|------|-------------|
| `200 OK` | Tenant deleted. |
| `401 Unauthorized` | Missing or invalid JWT. |
| `403 Forbidden` | Caller does not own this tenant. |

**Response (200):**
```json
{
  "deleted": "t_3a4b..."
}
```

**curl example:**
```bash
curl -X DELETE http://localhost:8080/api/v1/tenants/t_3a4b... \
  -H "Authorization: Bearer eyJhbGciOi..."
```

---

## API Key Management

API keys are **tenant-scoped** credentials for programmatic access (the `nxk_` keys used on the
agent endpoints). They are managed by a tenant owner over **JWT** auth. The raw key is returned
**only once**, at creation; afterwards only metadata is retrievable, and keys are revoked by their
hash.

**Typical bootstrap flow:**

1. `POST /auth/register` — create a user account.
2. `POST /auth/login` — obtain an access token (JWT).
3. `POST /api/v1/tenants` — create a tenant (provisions its namespace + node).
4. `POST /api/v1/tenants/:id/keys` — mint an API key; **save the `rawKey`** (shown once).
5. Use the `nxk_` key via `X-API-Key` for all agent operations.

| Method | Path | Auth | Purpose |
|--------|------|------|---------|
| `POST` | `/api/v1/tenants/:id/keys` | JWT | Create an API key for a tenant. |
| `GET` | `/api/v1/tenants/:id/keys` | JWT | List a tenant's API keys (metadata only). |
| `DELETE` | `/api/v1/tenants/:id/keys/:hash` | JWT | Revoke an API key by its hash. |

### POST /api/v1/tenants/:id/keys

| Property | Value |
|----------|-------|
| **Method** | `POST` |
| **Path** | `/api/v1/tenants/:id/keys` |
| **Auth** | JWT |
| **Content-Type** | `application/json` |

**Request body:**
```json
{
  "label": "ci-pipeline"
}
```

**Response codes:**

| Code | Description |
|------|-------------|
| `201 Created` | Key created. `rawKey` is shown **once** — store it now. |
| `400 Bad Request` | Invalid body. |
| `401 Unauthorized` | Missing or invalid JWT. |
| `403 Forbidden` | Caller does not own this tenant. |

**Response (201):**
```json
{
  "rawKey": "nxk_abc123...",
  "label": "ci-pipeline"
}
```

**curl example:**
```bash
curl -X POST http://localhost:8080/api/v1/tenants/t_3a4b.../keys \
  -H "Authorization: Bearer eyJhbGciOi..." \
  -H "Content-Type: application/json" \
  -d '{"label":"ci-pipeline"}'
```

---

### GET /api/v1/tenants/:id/keys

List a tenant's API keys. Returns metadata only — the raw key is never returned again.

| Property | Value |
|----------|-------|
| **Method** | `GET` |
| **Path** | `/api/v1/tenants/:id/keys` |
| **Auth** | JWT |

**Response codes:**

| Code | Description |
|------|-------------|
| `200 OK` | Array of key metadata. |
| `401 Unauthorized` | Missing or invalid JWT. |
| `403 Forbidden` | Caller does not own this tenant. |

**Response (200):**
```json
[
  {
    "hash": "9f86d081...",
    "label": "ci-pipeline",
    "createdAt": "2026-06-08T10:00:00Z"
  }
]
```

**curl example:**
```bash
curl http://localhost:8080/api/v1/tenants/t_3a4b.../keys \
  -H "Authorization: Bearer eyJhbGciOi..."
```

---

### DELETE /api/v1/tenants/:id/keys/:hash

Revoke an API key, identified by its hash (as returned by the list endpoint).

| Property | Value |
|----------|-------|
| **Method** | `DELETE` |
| **Path** | `/api/v1/tenants/:id/keys/:hash` |
| **Auth** | JWT |

**Response codes:**

| Code | Description |
|------|-------------|
| `200 OK` | Key revoked. |
| `401 Unauthorized` | Missing or invalid JWT. |
| `403 Forbidden` | Caller does not own this tenant. |

**Response (200):**
```json
{
  "revoked": "9f86d081..."
}
```

**curl example:**
```bash
curl -X DELETE http://localhost:8080/api/v1/tenants/t_3a4b.../keys/9f86d081... \
  -H "Authorization: Bearer eyJhbGciOi..."
```

---

## Agent Endpoints

The agent endpoints are **tenant-scoped**: the target namespace is the caller's tenant namespace
(`tenant-<uuid>`). Authenticate with either an API key (resolves to its tenant) or a JWT plus an
`X-Tenant-ID` header.

> **Agent body:** `POST` and `PUT` accept the **raw agent YAML** as the request body — set
> `Content-Type: application/x-yaml` and send it with `--data-binary @agent.yaml`. The agent name is
> taken from `metadata.name` in the manifest, not from the URL on create.
>
> **Observability endpoints:** in 0.26 `/status` and `/logs` return the agent's recent invocations
> (20 and 100 rows) read from the database, not a live proxy to the node. `/metrics` and `/usage`
> take `?period=YYYY-MM` (default: current month) and return `current_month` and `all_time`.
> The response examples under those endpoints below are v0.3.0-era; trust the field names the
> server actually returns.
>
> **Blocked runs.** Each invocation row (in `/logs` and in the invocation detail,
> `GET /agents/:name/invocations/:id`) carries `Blocked` (bool) and `Guardrail` (the rule object)
> **only when a guardrail rejected the run** (`omitempty`; Nexus 0.26.13+ persists
> them). A core 504 `{"error":"timeout"}` (`orchestration.timeout_seconds`) closes the
> invocation with `Status: "timeout"`.

### POST /api/v1/agents

Create a new agent from a YAML/JSON manifest.

| Property | Value |
|----------|-------|
| **Method** | `POST` |
| **Path** | `/api/v1/agents` |
| **Auth** | JWT or API key |
| **Content-Type** | `application/json` or `application/x-yaml` |

**Request body:** Full agent manifest (see `schemas/astromesh-v1-agent.md`).

**Response codes:**

| Code | Description |
|------|-------------|
| `201 Created` | Agent created successfully. |
| `400 Bad Request` | Validation errors in the manifest. |
| `401 Unauthorized` | Missing or invalid credentials (no valid API key or JWT). |
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
| **Auth** | JWT or API key |

**Query parameters:**

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `namespace` | string | all | Filter by namespace. |
| `label` | string | -- | Label selector (e.g., `team=sales`). |

**Response codes:**

| Code | Description |
|------|-------------|
| `200 OK` | Array of agent summaries. |
| `401 Unauthorized` | Missing or invalid credentials (no valid API key or JWT). |

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
| **Auth** | JWT or API key |

**Query parameters:**

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `namespace` | string | `default` | Namespace of the agent. |

**Response codes:**

| Code | Description |
|------|-------------|
| `200 OK` | Full agent manifest and runtime status. |
| `401 Unauthorized` | Missing or invalid credentials (no valid API key or JWT). |
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
| **Auth** | JWT or API key |
| **Content-Type** | `application/json` or `application/x-yaml` |

**Request body:** Full updated agent manifest.

**Response codes:**

| Code | Description |
|------|-------------|
| `200 OK` | Agent updated. |
| `400 Bad Request` | Validation errors. |
| `401 Unauthorized` | Missing or invalid credentials (no valid API key or JWT). |
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
| **Auth** | JWT or API key |

**Query parameters:**

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `namespace` | string | `default` | Namespace of the agent. |

**Response codes:**

| Code | Description |
|------|-------------|
| `200 OK` | Agent deleted. |
| `401 Unauthorized` | Missing or invalid credentials (no valid API key or JWT). |
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
| **Auth** | JWT or API key |

**Response codes:**

| Code | Description |
|------|-------------|
| `200 OK` | Status from node. |
| `401 Unauthorized` | Missing or invalid credentials (no valid API key or JWT). |
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
| **Auth** | JWT or API key |

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
| `401 Unauthorized` | Missing or invalid credentials (no valid API key or JWT). |
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
| **Auth** | JWT or API key |

**Query parameters:**

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `namespace` | string | `default` | Namespace. |
| `period` | string | `1h` | Time window: `5m`, `15m`, `1h`, `24h`, `7d`. |

**Response codes:**

| Code | Description |
|------|-------------|
| `200 OK` | Metrics object. |
| `401 Unauthorized` | Missing or invalid credentials (no valid API key or JWT). |
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

## Execution, Platform and Operator Endpoints (Nexus 0.26)

Source: `internal/server/server.go`, `handlers/invocations.go`, `invocations_stream.go`,
`middleware.go`. All `/api/v1/*` routes except `/api/v1/runs/*` and `/api/v1/admin/*` use
`DualAuthMiddleware` (API key in `X-API-Key`, or JWT Bearer + `X-Tenant-ID`). An operator JWT
(`isOperator` claim) may send any `X-Tenant-ID` without owning the tenant.

Some routes exist only when the deployment wires the dependency: tenant and key routes need the
Kubernetes client, `/catalog` and `/run*` need the runtime client, `/messages/send` and `/runs/*`
need Herald (the messaging gateway). A missing route is a deployment fact, not a Leia bug.

### POST /api/v1/agents/:name/run

Synchronous execution. Body: `{"query": "...", "session_id": "...", "context": {...}}`
(`session_id` defaults to `default`). Optional `Idempotency-Key` header (reuse by another request: `409`).

**200:**
```json
{
  "invocation_id": "inv_...", "status": "ok", "answer": "...",
  "usage": {"tokens_in": 120, "tokens_out": 40,
            "by_model": [{"provider": "...", "model": "...", "role": "default", "calls": 1,
                          "tokens_in": 120, "tokens_out": 40, "tokens_cached": 0, "cost": 0.0}]},
  "credits_charged_micros": 0, "pricing_status": "..."
}
```
`data`, `chain` and `propuestas` appear only when the runtime produced them. **`blocked` and
`guardrail` (core 0.70) are not forwarded by Nexus 0.26.11**: a guardrail block returns `200` with
the rejection text in `answer`. Nexus 0.26.12+ forwards them in the response (absent when nothing was blocked). The header `X-Nexus-Credit-Usage: <pct>` is set at 80% or more of the credit budget.

| Code | Meaning |
|------|---------|
| `400` | Invalid body. |
| `402` | Credit budget exhausted (`credit budget exhausted for this period`, or the subscription code `subscription_credit_budget_exhausted`). |
| `404` | Agent not found. |
| `409` | Idempotency key belongs to another request. |
| `422` | Agent has no live version (draft), or the subscription is inactive (`subscription_inactive`; retrying does not help). |
| `429` | Rate limited (`too many requests in this minute`) or too many concurrent invocations. |
| `502` / `504` | Runtime could not load the agent / load timed out. |
| `503` | Tenant or subscription limits, or the invocation record, could not be resolved; the run is not executed. |

A run that fails in the runtime returns an error body with `invocation_id` and `status` (`error` or `timeout`).

### GET /api/v1/agents/:name/run/stream (WebSocket)

Same admission checks as `/run`, done **before** the upgrade, so those failures are plain HTTP
statuses. Parameters travel in the URL: `?query=...&session_id=...` (`query` required, `400` otherwise).
After the upgrade the runtime's events are proxied as they come; the terminal event is
`{"type": "done", "invocation_id", "status": "ok", "answer", "usage", "credits_charged_micros", "pricing_status"[, "propuestas"]}`
or `{"type": "error", "invocation_id", "status", "error_code", "message"}`. Closing the socket cancels the run.

### Subscription-bound publish

`POST /api/v1/agents` and `PUT /api/v1/agents/:name` accept `?subscription_id=<id>` to attach the agent
to a subscription (the body is YAML, so it travels in the query). A missing subscription or one owned by
another tenant is `404`; a cancelled one is `422`. A `PUT` that omits it does not detach the agent.
Runs of that agent are then admitted against the subscription's limits (`402`/`422`/`429` above).

### Invocations, metrics and usage

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/agents/:name/invocations/:id` | `{invocation, models[], children[]}` for one run (trace, per-model usage, child runs). |
| GET | `/api/v1/agents/:name/status` | Recent invocations (20). |
| GET | `/api/v1/agents/:name/logs` | Recent invocations (100). |
| GET | `/api/v1/agents/:name/metrics?period=YYYY-MM` | `{agent, current_month, all_time}`. |
| GET | `/api/v1/usage?period=YYYY-MM` | `{tenant_id, current_month, all_time, agents[]}`. |
| GET | `/api/v1/agents/:name/versions`, `/versions/:number` | Immutable version history (each edit appends a version). |
| GET | `/api/v1/agents/by-id/:id/versions`, `/versions/:number` | Same by agent id (ids are never reused, names are). |

Agents are versioned in the database; `PUT` appends a version and moves the current pointer.
Create/update bodies are YAML or JSON (the raw body is converted), up to 1 MiB; invalid specs return
`400 {"error": "invalid agent spec", "problems": [...]}`.

### Other tenant routes

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/catalog` | Builtin tools and integrations the runtime offers (not tenant-scoped). |
| PUT / GET / DELETE | `/api/v1/connections/:name`, `/api/v1/connections` | Tenant connections (`base_url` + encrypted `material`); the tenant comes from the credential, not the path. |
| GET | `/api/v1/subscriptions` | The tenant's active subscription. |
| POST | `/api/v1/usage/events` | Report business-unit usage. Creating or changing a subscription is operator-only. |
| POST | `/api/v1/messages/send` | Proactive send: `{channel, recipient, text, conversation_ref}`; `202 {message_id, status}`, `400` missing fields, `502` gateway error. |
| POST | `/api/v1/runs/messages/send` | Same handler for the runtime, authenticated with a per-run token (not a tenant credential). |

### Operator plane: /api/v1/admin/*

Guarded by `OperatorMiddleware`: an operator JWT (`is_operator`) **or** the static `X-Operator-Token`
header. Not reachable with a tenant API key. Routes: `POST|GET /tariffs`, `POST|GET /plans`,
`PUT /tenants/:id/plan`, `POST|GET /subscriptions`, `PUT /subscriptions/:id/plan`,
`POST /subscriptions/:id/cancel`, `GET /usage`, `GET /tenants`, `GET /tenants/:id`, `GET /agents`,
`GET /invocations`, `GET /invocations/:id`, `GET /tenants/:id/statement` (and `.csv`),
`POST /billing-periods/:id/facturado`, `GET /billing-periods/overdue`. Leia does not automate these;
it only explains a `402/422/429` by pointing the user to their operator.

### Knowledge bundles on the pool

The runtime pool (`deploy/components/runtime-pool/deployment.yaml`) mounts only `runtime.yaml`,
no knowledge bundle directory (Nexus 0.26.9 and 0.26.10 changelogs say so). An agent that declares
`spec.knowledge.bundle` starts without it and logs a warning. Bundles work on standalone nodes.

---

## RAG Pipeline and Workflow Resources

### RAGPipeline resources (astromesh core v0.31.0+)

CRUD over RAGPipeline resources, the RAG twin of `/v1/agents`. An external tool
can author knowledge bases declaratively.

| Method | Path | Description |
|--------|------|-------------|
| GET | `/v1/rag/pipelines` | List pipelines (store seeded from `config/rag/*.rag.yaml`). |
| GET | `/v1/rag/pipelines/{name}` | Get one. |
| POST | `/v1/rag/pipelines` | Create; body validated via `spec_from_raw` (astromesh/rag/loader.py) (422 on malformed). |
| PUT | `/v1/rag/pipelines/{name}` | Update; `metadata.name` must equal the path (400 otherwise). |
| DELETE | `/v1/rag/pipelines/{name}` | Delete. |

Distinct from the operation endpoints `/v1/rag/ingest` and `/v1/rag/query`.

### Dynamic blueprint registration (astromesh core v0.32.0+)

| Method | Path | Description |
|--------|------|-------------|
| POST | `/v1/workflows/register` | Register a full blueprint `{workflow, agents, rag_pipelines}` at runtime (RAG → agents → workflow order, so an agent's KB resolves at build). Idempotent upsert; 422 on invalid spec, 503 without engine/runtime. Launch it afterwards with the existing `POST /v1/workflows/{name}/run`. |

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
