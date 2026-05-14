# resend_http

Resend or construct an HTTP/1.x or HTTP/2 request with `HTTPMessage`-typed schema fields. Each call records a fresh stream containing the resend's send and receive flows.

## Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `flow_id` | string | No | | Recorded stream id. When set, omitted fields are inherited from the original send |
| `method` | string | Conditional | | HTTP method (e.g. `"GET"`, `"POST"`). Required when `flow_id` is empty |
| `scheme` | string | Conditional | | `"http"` or `"https"`. Required when `flow_id` is empty |
| `authority` | string | Conditional | | Host header / `:authority` value. Required when `flow_id` is empty |
| `path` | string | Conditional | | Request path including the leading slash. Required when `flow_id` is empty. A literal `?` in `path` is auto-split into `path` (before `?`) plus `raw_query` (after `?`); supplying both `raw_query` AND a `?` in `path` returns an error |
| `raw_query` | string | No | | Raw query string without the leading `?` |
| `headers` | array | No | | Ordered header list as `[{"name": "...", "value": "..."}]`. Preserves wire case, order, and duplicates |
| `body` | string | No | | Request body interpreted per `body_encoding` |
| `body_encoding` | string | No | `"text"` | `"text"` or `"base64"` |
| `body_set` | boolean | No | `false` | Set `true` to override the body to empty; otherwise omitting `body` inherits the original |
| `body_patches` | array | No | | Patches applied on top of any body replacement (see below) |
| `override_host` | string | No | | Redirect the dial target while preserving the request's `Host`/`:authority` (must be `host:port`) |
| `follow_redirects` | boolean | No | `false` | Unsupported. Setting `true` returns an error |
| `timeout_ms` | integer | No | `30000` | Per-request timeout in milliseconds |
| `tls_fingerprint` | string | No | | Informational v1; per-call selection is deferred. The server uses its configured fingerprint |
| `tag` | string | No | | Tag stored on the new flow's `Tags` map |

### body_patches

Each patch is one of:

- **JSON path patch**: `{"json_path": "$.key", "value": "new"}`
- **Regex patch**: `{"regex": "pattern", "replace": "replacement"}`

An optional `"encoding"` array applies codec transformations as a pipeline.

### Header validation

Header names and values are rejected if they contain CR/LF characters (CWE-113 guard). The list is replaced wholesale when supplied; otherwise the recorded send's headers are inherited.

## Response

| Field | Type | Description |
|-------|------|-------------|
| `stream_id` | string | New stream record id holding the resend's send + receive flows |
| `status_code` | integer | HTTP response status code |
| `headers` | array | Ordered response header list `[{"name": "...", "value": "..."}]` |
| `body` | string | Response body (subject to SafetyFilter output masking) |
| `body_encoding` | string | `"text"` or `"base64"` |
| `duration_ms` | integer | Round-trip duration in milliseconds |
| `tag` | string | Echo of the supplied tag (when set) |

## Pipeline placement

The resend traverses `PluginStepPost -> RecordStep`. `PluginStepPre` and `InterceptStep` are bypassed (RFC-001 §9.3) so post-mutation hooks fire exactly once on the resent envelope while pre-pipeline annotation hooks stay quiet.

The resent stream is finalised on completion: `Stream.State` transitions to `"complete"` on success or `"error"` on failure (USK-789). The new flow records `origin = "resend"` so `query` callers can filter live capture out of analyses via `filter.origin = "proxy"`.

## Examples

### Resend with method override

```json
// resend_http
{
  "flow_id": "abc-123",
  "method": "PUT",
  "headers": [
    {"name": "Content-Type", "value": "application/json"}
  ]
}
```

### Construct from scratch

```json
// resend_http
{
  "method": "POST",
  "scheme": "https",
  "authority": "api.example.com",
  "path": "/v1/login",
  "headers": [
    {"name": "Content-Type", "value": "application/json"}
  ],
  "body": "{\"user\":\"alice\"}"
}
```

### Redirect to a different upstream

```json
// resend_http
{
  "flow_id": "abc-123",
  "override_host": "10.0.0.5:8443"
}
```

## Related pages

- [resend_ws](resend-ws.md) -- WebSocket frame resend
- [resend_grpc](resend-grpc.md) -- gRPC RPC resend
- [resend_raw](resend-raw.md) -- Raw bytes resend
- [fuzz_http](fuzz-http.md) -- HTTP fuzz with the same schema
- [Resender](../features/resender.md) -- Resender feature guide
- [query](query.md) -- Inspect the new stream by `stream_id`
