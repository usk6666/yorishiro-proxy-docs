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
| `blocked_by` | string | flows | Blocked flow filter: `"target_scope"`, `"intercept_drop"`, `"intercept_cancelled"`, `"rate_limit"`, `"safety_filter"`, `"budget"`, or `"capacity"`. See [Blocked flows](#blocked-flows) |
| `state` | string | flows | Flow lifecycle state (`"active"`, `"complete"`, `"error"`) |
| `conn_id` | string | flows | Connection ID filter (exact match) |
| `host` | string | flows | Host filter (matches server_addr or request URL host) |
| `wire_level` | string | flows, messages | Wire-fidelity overlay selector: `"semantic"` (default; the canonical application-level envelope), `"h1-chunk"`, `"h2-frame"`, `"grpc-lpm-frame"`, `"grpcweb-base64"`, or `"all"`. Overlay rows are hidden from `message_count` / `message_preview` and from default `messages` listings unless this filter explicitly includes them. See [Wire-level overlays](#wire-level-overlays) below |
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

### Wire-level overlays

A single application-level (semantic) flow may be persisted alongside one or more **wire-level overlay** rows that capture the exact wire bytes at a lower protocol layer. These overlays are forensic detail: they let you inspect the wire even when the semantic envelope has been re-encoded or reassembled.

| `wire_level` value | Records |
|--------------------|---------|
| `semantic` (default) | The canonical application-level envelope (HTTPMessage, GRPCDataMessage, WSMessage, SSEMessage, ...) |
| `h1-chunk` | One row per HTTP/1.x chunked-transfer chunk boundary |
| `h2-frame` | One row per HTTP/2 DATA frame (per direction, per stream), including WS/SSE-over-h2 paths |
| `grpc-lpm-frame` | One row per gRPC length-prefixed-message frame |
| `grpcweb-base64` | One row per gRPC-Web base64-encoded body chunk |
| `all` | Include every wire-level row alongside `semantic` |

Overlay rows are excluded by default from `flow.message_count`, `flow.message_preview`, and `messages` listings. Pass `filter.wire_level="h2-frame"` (etc.) to surface them, or `wire_level="all"` to get every layer.

HAR export ignores the wire_level filter -- HAR is a semantic format. Use the JSONL export (`manage.export_flows` with `filter.wire_level`) when forensic preservation of the wire matters.

### gRPC schema-aware decode

When [`grpc_schema`](grpc-schema.md) has a registered descriptor for the flow's `(grpc_service, grpc_method)` pair and `decode_bodies=true`, the body is decoded via `protojson` with real `.proto` field names and `body_decoded_encoding="proto-json"` is set on the response. On parse failure the proxy falls back to the schemaless path and surfaces `body_decode_anomaly { type: "proto_schema_mismatch" }`. Without a registered schema, gRPC bodies fall back to schemaless decode (`body_decoded_encoding="proto-schemaless-json"`).

### HTTPMessage anomalies

Parser-detected anomalies on HTTP messages (header smuggling indicators, malformed framing, etc.) are persisted on the flow row and surfaced by the `flow` and `messages` resources under `anomalies[]`. Filter for flows that carry an anomaly by combining `protocol=http` with a body-level filter, or fetch the full flow detail for inspection.

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

When intercept/transform modifies a request, the `original_request` field contains the pre-modification request data, including a `body_truncated` flag.

For TLS connections, `conn_info` carries the client's ClientHello fingerprints as `tls_client_ja3` and `tls_client_ja4`, alongside `tls_version`, `tls_cipher`, `tls_alpn`, and `tls_server_cert_subject`.

The `url` field (in `flow`, `flows[]`, `messages[]`, and `original_request`) carries the request path in **wire form** -- percent-encoding exactly as sent, not decoded. Because a JSON string cannot carry invalid UTF-8, any such path or query bytes are shown `%XX`-escaped with upper-case hex (the wire byte `0xFF` appears as `%FF`); every other byte is verbatim. `resend_http` with `flow_id` still replays the original wire bytes, not the escaped text.

#### Request heads closed by the proxy

HTTP/1.x request heads the proxy closed before the request was complete are recorded as `"error"` flows with `tags.closed_by = "proxy"` and one of these `failure_reason` values:

| `failure_reason` | Meaning |
|------------------|---------|
| `request_head_timeout` | The request line and headers did not arrive within `request_timeout_ms` (`tags.request_timeout_ms` holds the value in force). An idle keep-alive connection with zero bytes of a next request is not recorded |
| `request_line_too_long` | The request line exceeded the parser limit; the proxy answered `414 URI Too Long` |
| `request_header_too_large` | The header section exceeded the parser limit; the proxy answered `431 Request Header Fields Too Large` |

`tags.phase` is `request_line` or `headers`. `raw_request` holds the bytes received, capped at `max_raw_capture_size` (`metadata.raw_truncated = "true"` marks a capped capture, and `metadata.raw_captured_bytes` gives its length). For 414 / 431, the proxy-generated response is in `response_status_code` / `raw_response` (message `metadata.proxy_generated = "true"`); `tags.wrote_response = "false"` means the response could not be written. These flows share `conn_id` with earlier requests on the same connection. Because a client can create them by sending a partial or oversized request head, `capture_scope` and `max_raw_capture_size` bound what is stored.

#### Blocked flows

| `blocked_by` | Meaning |
|--------------|---------|
| `target_scope` | Blocked by the [target scope](../features/target-scope.md) policy |
| `intercept_drop` | Dropped by an `intercept` `drop` action |
| `intercept_cancelled` | An intercept hold that ended without an `intercept` action because the connection or proxy went away while it was held (`proxy_stop`, session teardown, the peer direction failing). State is `error`, and the stream `id` equals the held entry's `stream_id` |
| `rate_limit` | Rejected by a rate limit |
| `safety_filter` | Blocked by the Safety Filter |
| `budget` | Rejected because the diagnostic budget was exhausted |
| `capacity` | The proxy refused the connection because `max_connections` was reached. State is `error`, `server_addr` is the refused target (CONNECT authority, HTTP/1.x absolute-form authority, or TCP forward target), and `tags.target` is the request target as received. Written at most about 10 per second and only for targets inside `capture_scope`; the full count is in `status` -> `connection_rejections` |

Like other blocked flows, no flow rows carry the held or rejected content.

### messages

Get paginated messages within a flow. Requires `id`. Supports `limit`, `offset`, and `filter.direction`.

Returns: `messages[]`, `count`, `total`.

- **body_encoding**: `"text"` for UTF-8 safe bodies, `"base64"` for binary content.
- **metadata**: Protocol-specific fields (e.g. WebSocket `opcode`, gRPC `service`/`method`/`grpc_status`).

### status

Get current proxy status and health metrics. No additional parameters.

Returns: `running`, `listen_addr`, `listeners[]`, `listener_count`, `upstream_proxy`, `active_connections`, `max_connections`, `peek_timeout_ms`, `request_timeout_ms`, `total_flows`, `db_size_bytes`, `uptime_seconds`, `ca_initialized`, `socks5_enabled`, `tls_fingerprint`, `rate_limits`, `budget`, `h1_upstream`, `h2_upstream`, `connection_rejections`, `export_dir`.

| Field | Description |
|-------|-------------|
| `listeners[]` | Per-listener `name`, `listen_addr`, `active_connections`, `uptime_seconds`, `upstream_proxy`, and -- while upstream proxy rotation is active -- `upstream_proxy_template` (credentials redacted) and `upstream_proxy_rotation_policy` |
| `h1_upstream` | HTTP/1.x upstream redial / replay counters: `stale_detected_healthcheck`, `write_epipe`, `redial_healthcheck`, `redial_write_epipe`, `redial_failed_healthcheck`, `redial_failed_write_epipe`, `replay_success`, ... |
| `h2_upstream` | HTTP/2 upstream redial counters: `goaway_observed_prewarm`, `goaway_observed_reactive`, `refused`, `redial_prewarm`, `redial_reactive`, `redial_refused_stream`, `prewarm_suppressed`, and the matching `redial_failed_*` counters |
| `connection_rejections` | Connections refused because `max_connections` was reached, counted since server start: `total`, `last_at`, `by_kind` (`http1`, `connect`, `socks5`, `http2`, `tcp`, `unknown`), `recent[]` (newest first: `at`, `listener`, `forward_port`, `remote_addr`, `detected_kind`, `method`, `target`, `target_truncated`, `wrote_503`, `max_connections`, `recorded`), `unrecorded` (audit flows shed by the write-rate cap or a full recorder queue), and `out_of_scope` (not recorded because `capture_scope` excludes the target) |
| `export_dir` | Directory that [`manage`](manage.md) `export_flows` / `import_flows` file paths are confined to. Empty means file export / import is disabled |

!!! note "Modified intercept variants surface as separate flows"
    When an intercept rule modifies a request or response, both the original and the modified envelope are recorded as separate flows. The modified flow receives a fresh UUID `id` (no longer the legacy `-modified` suffix) and carries `metadata.variant = "modified"` so import round-trips stay consistent. Filter on `variant=original|modified` (or read the metadata field) to distinguish them.

### config

Get the current configuration. No additional parameters.

Returns: `upstream_proxy`, `tls_passthrough` (patterns, count), `tcp_forwards`, `socks5_enabled`, `client_cert`, `safety_filter` (`enabled`, `input_rules`, `output_rules`), `max_connections`, `peek_timeout_ms`, `request_timeout_ms`, `tls_fingerprint`, size limits (`max_raw_capture_size`, `max_body_size`, `body_spill_threshold`, ...), `capture_scope` (`includes`, `excludes`), `intercept_queue`, and `export_dir`.

`intercept_queue` echoes the current hold-queue settings in the same shape that [`configure`](configure.md) returns (`timeout_ms`, `timeout_behavior`, `queued_items`, per-protocol overrides). It is omitted until the proxy has started.

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
