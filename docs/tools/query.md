# query

Unified information query tool. Retrieve flows, flow details, messages, proxy status, configuration, CA certificate, intercept queue, macros, fuzz jobs, fuzz results, and detected technologies.

## Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `resource` | string | Yes | | Resource to query (see below) |
| `id` | string | Conditional | | Flow ID or macro name. Required for `flow`, `messages`, and `macro` |
| `fuzz_id` | string | Conditional | | Fuzz job ID. Required for `fuzz_results` |
| `filter` | object | No | | Filter options (see below) |
| `fields` | string[] | No | all | Fields to include in the response |
| `sort_by` | string | No | | Field to sort results by |
| `limit` | integer | No | `50` | Maximum items to return (max 1000) |
| `offset` | integer | No | `0` | Items to skip for pagination |

### filter

| Field | Type | Applies to | Description |
|-------|------|------------|-------------|
| `protocol` | string | flows | Protocol filter (e.g. `"HTTP/1.x"`, `"HTTPS"`, `"WebSocket"`, `"HTTP/2"`, `"gRPC"`, `"TCP"`) |
| `method` | string | flows | HTTP method filter (e.g. `"GET"`, `"POST"`) |
| `url_pattern` | string | flows | URL substring match |
| `status_code` | integer | flows, fuzz_results | HTTP status code filter |
| `blocked_by` | string | flows | Blocked flow filter (e.g. `"target_scope"`, `"intercept_drop"`) |
| `state` | string | flows | Flow lifecycle state (`"active"`, `"complete"`, `"error"`) |
| `technology` | string | flows | Technology name filter (case-insensitive substring) |
| `conn_id` | string | flows | Connection ID filter (exact match) |
| `host` | string | flows | Host filter (matches server_addr or request URL host) |
| `direction` | string | messages | Message direction (`"send"` or `"receive"`) |
| `body_contains` | string | fuzz_results | Response body substring filter |
| `outliers_only` | boolean | fuzz_results | Return only outlier results |
| `status` | string | fuzz_jobs | Job status filter (e.g. `"running"`, `"completed"`) |
| `tag` | string | fuzz_jobs | Job tag filter (exact match) |

## Resources

### flows

List recorded proxy flows with optional filtering and pagination.

Each flow entry includes a `protocol_summary` field with protocol-specific information:

- **WebSocket**: `message_count`, `last_frame_type`
- **HTTP/2**: `stream_count`, `scheme`
- **gRPC**: `service`, `method`, `grpc_status`, `grpc_status_name`
- **TCP**: `send_bytes`, `receive_bytes`

Returns: `flows[]`, `count`, `total`.

### flow

Get full details of a single flow including request/response headers, bodies, and connection info. Requires `id`.

Flow state values:

- `"active"` -- in progress (send recorded, awaiting receive)
- `"complete"` -- finished successfully
- `"error"` -- failed (e.g. 502 error, upstream connection failure)

For streaming flows (`flow_type` != `"unary"`), the response includes `message_preview` (first 10 messages) and `message_count`. Use the `messages` resource with `limit`/`offset` to page through all messages.

When intercept/transform modifies a request, the `original_request` field contains the pre-modification request data.

### messages

Get paginated messages within a flow. Requires `id`. Supports `limit`, `offset`, and `filter.direction`.

Returns: `messages[]`, `count`, `total`.

- **body_encoding**: `"text"` for UTF-8 safe bodies, `"base64"` for binary content.
- **metadata**: Protocol-specific fields (e.g. WebSocket `opcode`, gRPC `service`/`method`/`grpc_status`).

### status

Get current proxy status and health metrics. No additional parameters.

Returns: `running`, `listen_addr`, `active_connections`, `total_flows`, `db_size_bytes`, `uptime_seconds`, `ca_initialized`, `tls_fingerprint`.

### config

Get current configuration including capture scope, TLS passthrough, TCP forwards, and enabled protocols. No additional parameters.

Returns: `capture_scope`, `tls_passthrough`, `tcp_forwards`, `enabled_protocols`.

### ca_cert

Get the CA certificate PEM, metadata, and persistence state. No additional parameters.

Returns: `pem`, `fingerprint`, `subject`, `not_after`, `persisted`, `cert_path`, `install_hint`.

### intercept_queue

List intercepted requests/responses currently waiting in the intercept queue. Supports `limit`.

Each item includes a `phase` field: `"request"` (pre-send) or `"response"` (post-receive).

Returns: `items[]`, `count`.

### macros

List all stored macro definitions with summary information. No additional parameters.

Returns: `macros[]` (name, description, step_count, created_at, updated_at), `count`.

### macro

Get full details of a single macro definition. Requires `id` (macro name).

Returns: `name`, `description`, `steps[]`, `initial_vars`, `timeout_ms`, `created_at`, `updated_at`.

### fuzz_jobs

List fuzz jobs with optional filtering. Supports `filter.status`, `filter.tag`, `fields`, `limit`, `offset`.

Returns: `jobs[]`, `count`, `total`.

### fuzz_results

Get results for a specific fuzz job with filtering, sorting, and aggregate statistics. Requires `fuzz_id`.

The `summary` includes:

- **total_results**: Total matching results.
- **statistics**: Aggregate stats with `status_code_distribution`, `body_length` and `timing_ms` distributions (min, max, median, stddev).
- **outliers**: Result IDs that deviate from baseline (by_status_code, by_body_length, by_timing).

Supports `filter.status_code`, `filter.body_contains`, `filter.outliers_only`, `fields`, `sort_by`, `limit`, `offset`.

`sort_by` values: `index_num` (default), `status_code`, `duration_ms`, `response_length`.

Returns: `results[]`, `count`, `total`, `summary`.

### technologies

Aggregate detected technology stacks per host across all completed flows.

Returns: `hosts[]` (host, technologies[] with name, version, category, confidence), `count`.

Categories: `web_server`, `framework`, `language`, `cms`, `cdn`, `waf`, `js_framework`.

## Examples

### List all flows

```json
// query
{
  "resource": "flows"
}
```

### Filter flows by protocol

```json
// query
{
  "resource": "flows",
  "filter": {"protocol": "WebSocket"}
}
```

### Filter flows by method and URL

```json
// query
{
  "resource": "flows",
  "filter": {"method": "POST", "url_pattern": "/api/login"},
  "limit": 10
}
```

### Get flow details

```json
// query
{
  "resource": "flow",
  "id": "abc-123"
}
```

### Get flow messages with pagination

```json
// query
{
  "resource": "messages",
  "id": "abc-123",
  "limit": 20,
  "offset": 0
}
```

### Check proxy status

```json
// query
{
  "resource": "status"
}
```

### Get current config

```json
// query
{
  "resource": "config"
}
```

### Export CA certificate

```json
// query
{
  "resource": "ca_cert"
}
```

### List intercepted requests

```json
// query
{
  "resource": "intercept_queue"
}
```

### List macros

```json
// query
{
  "resource": "macros"
}
```

### Get macro details

```json
// query
{
  "resource": "macro",
  "id": "auth-flow"
}
```

### List detected technologies

```json
// query
{
  "resource": "technologies"
}
```

### Get fuzz results with filtering

```json
// query
{
  "resource": "fuzz_results",
  "fuzz_id": "fuzz-789",
  "filter": {"status_code": 200, "body_contains": "admin"},
  "sort_by": "status_code",
  "limit": 50
}
```

### Get only outlier fuzz results

```json
// query
{
  "resource": "fuzz_results",
  "fuzz_id": "fuzz-789",
  "filter": {"outliers_only": true}
}
```

## Related pages

- [Flows](../concepts/flows.md) -- Understanding flow data
- [Intercept](../features/intercept.md) -- Intercept feature guide
- [Fuzzer](../features/fuzzer.md) -- Fuzzer feature guide
- [Technology detection](../features/technology-detection.md) -- Technology detection feature guide
