# fuzz_http

Synchronously fuzz an HTTP request with `HTTPMessage`-typed positions. The schema mirrors [`resend_http`](resend-http.md) plus a `positions[]` list -- each position is a typed path into the `HTTPMessage` shape. The cartesian product of all positions yields the variant sequence.

This tool is **synchronous**: it runs every variant in-process and returns the full per-variant result list when finished. There is no concurrency or rate limit -- pace from the client side. For larger asynchronous fuzz campaigns the legacy runner is being phased out in N9; this tool replaces it for typed HTTP fuzzing.

## Limits

- Maximum variants per call: **1000** (cartesian product across all positions)
- Maximum positions per call: **32**
- Maximum decoded payload size per position: **1 MiB**

Each variant is executed sequentially with a fresh dial. Per-variant `SafetyFilter` input gating runs after position substitution, before the upstream dial -- a blocked variant is recorded with `error` and the run continues.

## Parameters

`fuzz_http` inherits every base field from [`resend_http`](resend-http.md) (`flow_id`, `method`, `scheme`, `authority`, `path`, `raw_query`, `headers`, `body`, `body_encoding`, `body_set`, `body_patches`, `override_host`, `tls_fingerprint`, `timeout_ms`, `tag`).

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| (base fields) | -- | -- | -- | See [`resend_http`](resend-http.md). When `flow_id` is empty, `method`, `scheme`, `authority`, `path` are required |
| `positions` | array | Yes | | Ordered position list (see below). At least one entry |
| `stop_on_5xx` | boolean | No | `false` | Abort remaining variants once any variant returns a 5xx response |

### positions

Each position substitutes one typed path into the `HTTPMessage` shape:

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `path` | string | Yes | | Typed path: `method`, `scheme`, `authority`, `path`, `raw_query`, `body`, `headers[N].name`, `headers[N].value` |
| `payloads` | string[] | Yes | | List of values to substitute at this path. At least one element |
| `encoding` | string | No | `"text"` | `"text"` or `"base64"`. Applies to every payload |

Position payloads are written verbatim, including CR/LF. This is intentional: `fuzz_http` is the surface for request smuggling, header injection, and CRLF-injection fuzzing. The base-headers path still rejects CR/LF by design; per-position payloads bypass that guard.

## Response

| Field | Type | Description |
|-------|------|-------------|
| `total_variants` | integer | Total variant count from the cartesian product |
| `completed_variants` | integer | Variants actually executed before completion or stop |
| `stopped_reason` | string | Empty when all variants ran; `"stop_on_5xx: ..."` or `"ctx cancelled: ..."` otherwise |
| `variants` | array | Per-variant result rows (see below) |
| `duration_ms` | integer | Total run duration in milliseconds |
| `tag` | string | Echo of the supplied tag (when set) |

Each variant row:

| Field | Type | Description |
|-------|------|-------------|
| `index` | integer | Variant index (zero-based, in execution order) |
| `stream_id` | string | New stream record id under which `RecordStep` persisted the variant's flows |
| `status_code` | integer | HTTP response status code (`0` if the variant errored) |
| `body_size` | integer | Response body byte length |
| `payloads` | object | Map of position path -> decoded payload string for this variant |
| `error` | string | Error message when the variant failed (network, timeout, SafetyFilter) |
| `duration_ms` | integer | Per-variant duration |

Full response payloads are not stored on the row. Retrieve them via the [`query`](query.md) tool keyed by `stream_id`.

## Pipeline placement

Each variant traverses the same self-contained `PluginStepPost -> RecordStep` pipeline as `resend_http` (`PluginStepPre` is bypassed per RFC-001 §9.3).

## Examples

### Fuzz a path segment

```json
// fuzz_http
{
  "flow_id": "abc-123",
  "positions": [
    {
      "path": "path",
      "payloads": ["/admin", "/admin/", "/Admin", "/.git/config"]
    }
  ]
}
```

### Two-position cartesian product

```json
// fuzz_http
{
  "flow_id": "abc-123",
  "positions": [
    {"path": "headers[0].value", "payloads": ["Bearer A", "Bearer B"]},
    {"path": "body", "payloads": ["{\"role\":\"user\"}", "{\"role\":\"admin\"}"]}
  ],
  "stop_on_5xx": true
}
```

### CRLF smuggling fuzz

```json
// fuzz_http
{
  "flow_id": "abc-123",
  "positions": [
    {
      "path": "raw_query",
      "payloads": [
        "x=1\r\nX-Forwarded-For: 127.0.0.1",
        "x=1\r\nTransfer-Encoding: chunked"
      ]
    }
  ]
}
```

## Related pages

- [resend_http](resend-http.md) -- HTTP resend (single shot)
- [fuzz_ws](fuzz-ws.md) -- WebSocket fuzz
- [fuzz_grpc](fuzz-grpc.md) -- gRPC fuzz
- [fuzz_raw](fuzz-raw.md) -- Raw byte fuzz
- [Fuzzer](../features/fuzzer.md) -- Fuzzer feature guide
- [query](query.md) -- Inspect variant streams by `stream_id`
