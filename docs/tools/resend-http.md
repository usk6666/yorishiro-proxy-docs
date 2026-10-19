# resend_http

Resend or construct an HTTP/1.x or HTTP/2 request with `HTTPMessage`-typed schema fields. Each call records a fresh stream containing the resend's send and receive flows.

## Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `flow_id` | string | No | | Recorded stream id. When set, omitted fields are inherited from the original send |
| `method` | string | Conditional | | HTTP method (e.g. `"GET"`, `"POST"`). Required when `flow_id` is empty |
| `scheme` | string | Conditional | | `"http"` or `"https"`. Required when `flow_id` is empty |
| `authority` | string | Conditional | | Host / `:authority` value. Sent as `:authority` over HTTP/2; over HTTP/1.1 it is the default `Host` header when none is supplied. Required when `flow_id` is empty |
| `path` | string | Conditional | | Request path including the leading slash. Required when `flow_id` is empty. A literal `?` in `path` is auto-split into `path` (before `?`) plus `raw_query` (after `?`); supplying both `raw_query` AND a `?` in `path` returns an error. The path is sent and recorded exactly as given (wire form): percent-encoding is neither decoded nor added. When inherited from `flow_id`, the recorded wire-form path is replayed byte-for-byte |
| `raw_query` | string | No | | Raw query string without the leading `?` |
| `headers` | array | No | | Ordered header list as `[{"name": "...", "value": "..."}]`. Preserves wire case, order, and duplicates |
| `body` | string | No | | Request body interpreted per `body_encoding` |
| `body_encoding` | string | No | `"text"` | `"text"` or `"base64"` |
| `body_set` | boolean | No | `false` | Set `true` to override the body to empty; otherwise omitting `body` inherits the original |
| `body_patches` | array | No | | Patches applied on top of any body replacement (see below) |
| `override_host` | string | No | | Redirect the dial target while preserving the request's `Host`/`:authority` (must be `host:port`) |
| `follow_redirects` | boolean | No | `false` | Unsupported. Setting `true` returns an error |
| `http_version` | string | No | recorded flow's, else `"http/1.1"` | Wire protocol for the upstream send: `"http/1.1"`, `"h2"` (scheme `https` only, ALPN `h2`), or `"h2c"` (scheme `http` only, prior knowledge). See [Wire protocol](#wire-protocol-http11-vs-http2) |
| `upstream_proxy` | object | No | | Per-call upstream proxy override: `{"url_template": "..."}`. See [Upstream proxy override](#upstream-proxy-override) |
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

### Wire protocol (HTTP/1.1 vs HTTP/2)

The resend goes out over the protocol the original flow used -- it is never silently downgraded:

| Source of the decision | Result |
|------------------------|--------|
| `http_version` supplied | That protocol (`http/1.1`, `h2`, or `h2c`) |
| `flow_id` whose recorded `http_version` is `h2` / `h2c` | HTTP/2: `h2` (TLS, ALPN) when the resolved scheme is `https`, `h2c` (cleartext, prior knowledge) when it is `http` |
| Anything else (HTTP/1.x flow, unversioned flow, from-scratch request) | `http/1.1` |

- **`h2`** offers the configured ALPN list unchanged, so the ClientHello matches the live proxy path, and requires the upstream to select `h2`. If it selects anything else, the call fails with an error naming the negotiated ALPN -- pass `"http_version": "http/1.1"` to downgrade deliberately. HTTP/2 SETTINGS and pseudo-header order follow the configured `tls_fingerprint`.
- **HTTP/2 headers** -- the request target is sent as `:method` / `:scheme` / `:authority` / `:path`. No `Host` header is added. Header names are lowercased on the wire and connection-specific headers (`Connection`, `Keep-Alive`, `Transfer-Encoding`, `Upgrade`, `Proxy-Connection`, `TE` other than `trailers`) are stripped -- relevant when you force an HTTP/1.x flow to `h2`.
- **HTTP/1.1** pins ALPN to `http/1.1` and adds a `Host` header from `authority` when the header list has none.
- The new flows record the protocol actually used in `http_version`. 1xx interim responses are recorded as their own receive flows; the result fields describe the final response.
- WebSocket over HTTP/2 (RFC 8441) is not covered -- use [`resend_ws`](resend-ws.md).

### Upstream proxy override

`upstream_proxy.url_template` routes this one resend through an upstream proxy (`http://` via CONNECT, or `socks5://`), for example to switch residential proxy IPs. The template uses `§var§` syntax and is expanded once per call with the reserved variables `§__iteration§` (always `0` for resend) and `§__nonce§` (a fresh UUID). The expanded URL is checked for CRLF injection.

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

### Resend through a rotating residential proxy

```json
// resend_http
{
  "flow_id": "abc-123",
  "upstream_proxy": {
    "url_template": "http://user-session-§__nonce§:password@residential.example:8080"
  }
}
```

### Replay an HTTP/2 flow over HTTP/1.1 on purpose

```json
// resend_http
{
  "flow_id": "abc-123",
  "http_version": "http/1.1"
}
```

## Related pages

- [resend_ws](resend-ws.md) -- WebSocket frame resend
- [resend_grpc](resend-grpc.md) -- gRPC RPC resend
- [resend_raw](resend-raw.md) -- Raw bytes resend
- [fuzz_http](fuzz-http.md) -- HTTP fuzz with the same schema
- [Resender](../features/resender.md) -- Resender feature guide
- [query](query.md) -- Inspect the new stream by `stream_id`
