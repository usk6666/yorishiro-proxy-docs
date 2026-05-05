# Resender

The resender lets you replay recorded proxy requests with optional mutations. You can override the method, URL, headers, and body, apply JSON patches or regex replacements, and resend raw bytes for protocol-level testing. Each protocol has its own typed sibling tool.

## Tools overview

| Tool | Description |
|--------|-------------|
| [`resend_http`](../tools/resend-http.md) | Resend an HTTP/1.x or HTTP/2 request with typed `HTTPMessage` fields |
| [`resend_ws`](../tools/resend-ws.md) | Resend a single WebSocket frame |
| [`resend_grpc`](../tools/resend-grpc.md) | Resend a gRPC unary RPC |
| [`resend_raw`](../tools/resend-raw.md) | Resend raw bytes over TCP/TLS with byte-level patches |

## Resending with overrides

The `resend_http` tool replays a recorded HTTP flow and lets you override any part of the request. The original flow is not modified; a new stream is created with the result.

### Method override

Change the HTTP method of the request:

```json
// resend_http
{
  "flow_id": "abc-123",
  "method": "PUT"
}
```

### URL override

Redirect the request to a different path or authority. Set the components individually:

```json
// resend_http
{
  "flow_id": "abc-123",
  "scheme": "https",
  "authority": "staging.target.com",
  "path": "/api/v2/users"
}
```

### Header mutations

Supplying `headers` replaces the recorded send's full ordered header list. To change a single header, recover the recorded list (e.g. via [`query`](../tools/query.md)), edit the entry, and pass the full list back:

```json
// resend_http
{
  "flow_id": "abc-123",
  "headers": [
    {"name": "Content-Type", "value": "application/json"},
    {"name": "X-Custom", "value": "value1"},
    {"name": "X-Custom", "value": "value2"}
  ]
}
```

The list preserves wire case, order, and duplicates. To remove a header, simply omit it from the supplied list. Header names and values containing CR/LF are rejected (CWE-113 guard).

### Body override

Replace the entire request body with text or Base64-encoded binary data:

```json
// resend_http
{
  "flow_id": "abc-123",
  "body": "{\"username\": \"admin\", \"role\": \"superuser\"}"
}
```

For binary data, set `body_encoding` to `"base64"`:

```json
// resend_http
{
  "flow_id": "abc-123",
  "body": "SGVsbG8gV29ybGQ=",
  "body_encoding": "base64"
}
```

To send an explicitly empty body, set `"body_set": true` (omitting `body` alone inherits the recorded body). When `body` is set it replaces the recorded body before `body_patches` are applied.

## JSON patches

For surgical modifications to JSON bodies, use `body_patches` with `json_path`. The path uses simplified dot notation (`$.key1.key2`):

```json
// resend_http
{
  "flow_id": "abc-123",
  "body_patches": [
    {"json_path": "$.user.role", "value": "admin"},
    {"json_path": "$.user.active", "value": true},
    {"json_path": "$.settings.limit", "value": 9999}
  ]
}
```

!!! info "JSON path limitations"
    The JSON path parser supports dot notation only (`$.key1.key2.key3`). Array index notation is not supported. The `$` prefix is optional.

## Regex body patches

For text-based modifications, use `body_patches` with `regex` and `replace`. Capture group references (`$1`, `$2`) are supported:

```json
// resend_http
{
  "flow_id": "abc-123",
  "body_patches": [
    {"regex": "csrf_token=[^&]+", "replace": "csrf_token=injected_value"},
    {"regex": "(role=)(user)", "replace": "${1}admin"}
  ]
}
```

### Encoding chains

Both JSON and regex patches support an `encoding` array that applies codec transformations to the patch value before applying. Codecs are applied in pipeline order:

```json
// resend_http
{
  "flow_id": "abc-123",
  "body_patches": [
    {
      "json_path": "$.token",
      "value": "<script>alert(1)</script>",
      "encoding": ["url_encode_query", "base64"]
    }
  ]
}
```

Available codecs: `base64`, `base64url`, `url_encode_query`, `url_encode_path`, `url_encode_full`, `double_url_encode`, `hex`, `html_entity`, `html_escape`, `unicode_escape`, `md5`, `sha256`, `lower`, `upper`.

Maximum chain length is 10 codecs.

## Body mutation priority

The body is resolved in this priority order:

1. `body` (full replacement; or `body_set: true` to force empty)
2. `body_patches` (partial modifications)
3. Original body from the recorded flow

## Raw bytes resend

The `resend_raw` tool sends the raw bytes from a recorded flow over a freshly dialled TCP/TLS connection. This is useful for testing HTTP request smuggling, malformed requests, or protocol-level vulnerabilities. `target_addr` (host:port) is required -- raw is protocol-agnostic, so there is no defaulting.

```json
// resend_raw
{
  "flow_id": "abc-123",
  "target_addr": "target.com:443",
  "use_tls": true
}
```

### Raw byte patches

You can modify raw bytes with offset-based patches:

```json
// resend_raw
{
  "flow_id": "abc-123",
  "target_addr": "target.com:443",
  "use_tls": true,
  "patches": [
    {"offset": 0, "data": "R0VU", "data_encoding": "base64"}
  ]
}
```

Each patch overwrites bytes at the given zero-based offset. `data_encoding` defaults to `"text"` for ASCII edits:

```json
// resend_raw
{
  "flow_id": "abc-123",
  "target_addr": "target.com:80",
  "patches": [
    {"offset": 0, "data": "POST"}
  ]
}
```

Find/replace patch modes from the legacy tool are no longer supported -- compute offsets client-side from the recovered bytes (available via [`query`](../tools/query.md)).

You can also fully replace the bytes with `override_bytes` (paired with `override_bytes_encoding`):

```json
// resend_raw
{
  "flow_id": "abc-123",
  "target_addr": "target.com:80",
  "override_bytes": "R0VUIC8gSFRUUC8xLjENCkhvc3Q6IGV4YW1wbGUuY29tDQoNCg==",
  "override_bytes_encoding": "base64"
}
```

## Raw TCP replay

To replay a raw TCP flow as-is, use `resend_raw` without overrides. Each call records a new stream containing the resend's send and received bytechunk flows:

```json
// resend_raw
{
  "flow_id": "tcp-flow-456",
  "target_addr": "db.target.com:3306",
  "use_tls": false,
  "tag": "tcp-replay-test"
}
```

## Multi-protocol support

Each protocol has its own typed resend tool:

| Protocol | Tool |
|----------|----------|
| HTTP/1.x and HTTP/2 | [`resend_http`](../tools/resend-http.md) |
| WebSocket | [`resend_ws`](../tools/resend-ws.md) |
| gRPC | [`resend_grpc`](../tools/resend-grpc.md) |
| Raw TCP / TLS | [`resend_raw`](../tools/resend-raw.md) |

### WebSocket resend

For WebSocket flows, use `resend_ws`. Each call dials a fresh upstream, performs the upgrade dance, sends one frame, and records the response:

```json
// resend_ws
{
  "flow_id": "ws-flow-123",
  "opcode": "text",
  "payload": "{\"action\":\"ping\"}"
}
```

To send from scratch (no recorded flow), supply `target_addr`, `scheme`, and `path`:

```json
// resend_ws
{
  "target_addr": "ws.target.com:443",
  "scheme": "wss",
  "path": "/socket",
  "opcode": "binary",
  "payload": "AAECAw==",
  "body_encoding": "base64"
}
```

## Additional options

| Parameter | Type | Description |
|-----------|------|-------------|
| `override_host` | string | (HTTP) Redirect the dial target while preserving the request's `Host` / `:authority` (must be `host:port`) |
| `target_addr` | string | (WS / raw) Upstream `host:port` for the dial |
| `timeout_ms` | integer | Per-call timeout in milliseconds (default: `30000`) |
| `tag` | string | Tag stored on the new stream's `Tags` map |

`follow_redirects` is no longer auto-followed -- chase redirects from the client side by inspecting the response and issuing a new resend.

## Hooks integration

Pre/post macro hooks for individual resend calls have been removed in favor of the streamlined per-protocol pipeline (`PluginStepPost -> RecordStep`). To run a macro before or after a resend, invoke the macro tool independently and stitch the calls from the client side. See [Macros](macros.md) for the standalone macro execution model.

## Practical use cases

### Authentication testing

Resend a login request with different credentials to test for weak password policies:

```json
// resend_http
{
  "flow_id": "login-flow-id",
  "body_patches": [
    {"json_path": "$.password", "value": "admin123"}
  ]
}
```

### Parameter tampering

Modify a recorded API request to test for authorization bypass:

```json
// resend_http
{
  "flow_id": "api-flow-id",
  "body_patches": [
    {"json_path": "$.user_id", "value": 1},
    {"json_path": "$.role", "value": "admin"}
  ],
  "tag": "idor-test"
}
```

### HTTP request smuggling

Use `resend_raw` with offset patches to test for request smuggling. Compute offsets from the recovered bytes (available via [`query`](../tools/query.md)) and patch the relevant header bytes:

```json
// resend_raw
{
  "flow_id": "abc-123",
  "target_addr": "target.com:80",
  "patches": [
    {"offset": 64, "data": "Transfer-Encoding: chunked\r\n\r\n"}
  ]
}
```

## Related pages

- [resend_http](../tools/resend-http.md), [resend_ws](../tools/resend-ws.md), [resend_grpc](../tools/resend-grpc.md), [resend_raw](../tools/resend-raw.md) -- typed per-protocol resend MCP tools
- [Fuzzer](fuzzer.md) -- Automated fuzzing with payload injection
- [Macros](macros.md) -- Hook integration for multi-step workflows
