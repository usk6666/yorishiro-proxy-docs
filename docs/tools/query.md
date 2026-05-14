# query

Unified information query tool. Retrieve flows, flow details, messages, proxy status, configuration, CA certificate, intercept queue, macros, fuzz jobs, and fuzz results.

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
| `include_bodies` | boolean | No | `true` | Include message bodies in responses. When `false`, body fields are suppressed; metadata, headers, and `body_truncated` remain. Mirrors `manage.export_flows.include_bodies` |
| `body_max_bytes` | integer | No | `0` | Truncate per-message body to at most this many bytes (`0` = no cap). When applied, `body_truncated_by_query=true` and `body_original_size` reports the pre-truncation length |
| `decode_bodies` | boolean | No | `true` | Decode HTTP `Content-Encoding` (gzip / deflate / br / zstd) into `*_body_decoded` fields. The original wire-form body is always returned in `*_body`. Set `false` to skip decoding for performance |

### filter

| Field | Type | Applies to | Description |
|-------|------|------------|-------------|
| `protocol` | string | flows | Canonical Message-type family. One of `"http"`, `"ws"`, `"grpc"`, `"grpc-web"`, `"sse"`, `"raw"`, `"tls-handshake"`. See [Protocol family filter](#protocol-family-filter) |
| `scheme` | string | flows | Wire-observed handshake transport: `"http"`, `"https"`, `"tcp"`. Use `scheme=https` to match all TLS flows regardless of HTTP version. Application-level schemes (`ws`, `wss`) are rejected -- combine `protocol="ws"` with `scheme="https"` for WS-over-TLS |
| `http_version` | string | flows | HTTP version filter: `"http/1.0"`, `"http/1.1"`, `"h2"`, `"h2c"`. Empty-string explicit value matches pre-USK-788 rows that lack a recorded version |
| `origin` | string | flows | Stream origin filter: `"proxy"` (live capture), `"resend"` (resend_* tools), `"fuzz"` (fuzz campaigns) |
| `method` | string | flows | HTTP method filter (e.g. `"GET"`, `"POST"`) |
| `url_pattern` | string | flows | URL substring match |
| `status_code` | integer | flows, fuzz_results | HTTP status code filter |
| `blocked_by` | string | flows | Blocked flow filter (e.g. `"target_scope"`, `"intercept_drop"`, `"safety_filter"`) |
| `state` | string | flows | Flow lifecycle state (`"active"`, `"complete"`, `"error"`) |
| `conn_id` | string | flows | Connection ID filter (exact match) |
| `host` | string | flows | Host filter (matches server_addr or request URL host) |
| `direction` | string | messages | Message direction (`"send"` or `"receive"`) |
| `body_contains` | string | fuzz_results | Response body substring filter |
| `outliers_only` | boolean | fuzz_results | Return only outlier results |
| `status` | string | fuzz_jobs | Job status filter (e.g. `"running"`, `"completed"`) |
| `tag` | string | fuzz_jobs | Job tag filter (exact match) |

### Protocol family filter

The `filter.protocol` value matches a canonical `Envelope.Protocol` family. Each canonical value expands across every wire spelling the proxy may have recorded for that family:

| Value | Matches |
|-------|---------|
| `http` | HTTP/1.x and HTTP/2 traffic, including TLS variants and SOCKS5-tunnelled spellings |
| `ws` | Native WebSocket and SOCKS5-tunnelled WebSocket |
| `grpc` | Native gRPC over HTTP/2 |
| `grpc-web` | gRPC-Web (HTTP/1.1 and HTTP/2 transports) |
| `sse` | Server-Sent Events |
| `raw` | TCP-passthrough and `Raw` flows |
| `tls-handshake` | TLS handshake records observed without a higher-level decode |

Only these canonical values are accepted -- the legacy literals (`HTTP/1.x`, `HTTPS`, `HTTP/2`, `WebSocket`, `gRPC`, `gRPC-Web`, `TCP`, `SOCKS5+...`) were retired in N9. Use `filter.scheme` to find all TLS flows (e.g. `scheme=https` returns HTTP/1.x + HTTP/2 + gRPC over TLS).

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

!!! note "Modified intercept variants surface as separate flows"
    When an intercept rule modifies a request or response, both the original and the modified envelope are recorded as separate flows. The modified flow receives a fresh UUID `id` (no longer the legacy `-modified` suffix) and carries `metadata.variant = "modified"` so import round-trips stay consistent. Filter on `variant=original|modified` (or read the metadata field) to distinguish them.

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
  "filter": {"protocol": "ws"}
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

### Filter flows by origin (exclude resend / fuzz traffic)

```json
// query
{
  "resource": "flows",
  "filter": {"origin": "proxy"}
}
```

### Page through metadata-only flow listings

```json
// query
{
  "resource": "flows",
  "include_bodies": false,
  "limit": 200
}
```

### Cap response body size per message

```json
// query
{
  "resource": "messages",
  "id": "abc-123",
  "body_max_bytes": 4096
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
