# fuzz_grpc

Synchronously fuzz a gRPC unary RPC with `GRPCStartMessage` / `GRPCDataMessage`-typed positions. The schema mirrors [`resend_grpc`](resend-grpc.md) plus a `positions[]` list. Each variant becomes one independent gRPC stream -- fresh dial, fresh `ConnID`, fresh `StreamID`.

This tool is **synchronous**: every variant runs in-process and the full per-variant result list is returned when finished. There is no concurrency or rate limit -- pace from the client side.

## Limits

- Maximum variants per call: **1000** (cartesian product across all positions)

Per-variant `SafetyFilter` input gating runs before the upstream send -- blocked variants are recorded with `error` and the run continues.

## Parameters

`fuzz_grpc` inherits every base field from [`resend_grpc`](resend-grpc.md) (`flow_id`, `target_addr`, `scheme`, `service`, `method`, `metadata`, `encoding`, `accept_encoding`, `messages`, `trailer_metadata`, `tls_fingerprint`, `timeout_ms`, `tag`).

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| (base fields) | -- | -- | -- | See [`resend_grpc`](resend-grpc.md). When `flow_id` is empty, `target_addr` + `service` + `method` are required. `messages[]` must have at least one element |
| `positions` | array | Yes | | Ordered position list (see below). At least one entry |
| `stop_on_non_ok` | boolean | No | `false` | Abort remaining variants once any variant returns a non-OK gRPC status (or terminates without a trailer) |

### positions

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `path` | string | Yes | | Typed path: `service`, `method`, `metadata[N].name`, `metadata[N].value`, `messages[N].payload` |
| `payloads` | string[] | Yes | | List of values to substitute at this path. At least one element |
| `encoding` | string | No | `"text"` | `"text"` or `"base64"`. Applies to every payload |

`scheme`, `target_addr`, and `encoding` are intentionally **not** fuzz positions -- they affect connection setup. Issue separate `fuzz_grpc` calls when you need to vary those.

## Response

| Field | Type | Description |
|-------|------|-------------|
| `total_variants` | integer | Total variant count |
| `completed_variants` | integer | Variants actually executed before completion or stop |
| `stopped_reason` | string | Empty when all variants ran; otherwise the stop reason |
| `variants` | array | Per-variant result rows (see below) |
| `duration_ms` | integer | Total run duration in milliseconds |
| `tag` | string | Echo of the supplied tag (when set) |

Each variant row:

| Field | Type | Description |
|-------|------|-------------|
| `index` | integer | Variant index (zero-based) |
| `stream_id` | string | New stream record id |
| `status` | integer | gRPC status code (`0` = OK) |
| `status_message` | string | gRPC status message (when non-empty) |
| `response_message_count` | integer | Receive-direction Data envelope count |
| `response_total_bytes` | integer | Total response payload size in bytes |
| `payloads` | object | Map of position path -> decoded payload string |
| `error` | string | Error message when the variant failed |
| `duration_ms` | integer | Per-variant duration |

Full response payloads are not ferried into the result. Retrieve them via the [`query`](query.md) tool keyed by `stream_id`.

## Pipeline placement

Each Start and Data envelope per variant traverses `PluginStepPost -> RecordStep` (`PluginStepPre` is bypassed per RFC-001 §9.3). End is observation-only.

## Examples

### Fuzz a metadata value

```json
// fuzz_grpc
{
  "flow_id": "grpc-abc-123",
  "messages": [{"payload": "{\"name\":\"world\"}"}],
  "positions": [
    {
      "path": "metadata[0].value",
      "payloads": ["Bearer token-A", "Bearer token-B", "Bearer admin"]
    }
  ]
}
```

### Fuzz a method name

```json
// fuzz_grpc
{
  "target_addr": "127.0.0.1:50051",
  "scheme": "http",
  "service": "pkg.Greeter",
  "method": "SayHello",
  "messages": [{"payload": "AAAAAAk=", "body_encoding": "base64"}],
  "positions": [
    {"path": "method", "payloads": ["SayHello", "SayHelloAdmin", "Reflect"]}
  ],
  "stop_on_non_ok": true
}
```

## Related pages

- [resend_grpc](resend-grpc.md) -- gRPC RPC resend (single shot)
- [fuzz_http](fuzz-http.md) -- HTTP fuzz
- [fuzz_ws](fuzz-ws.md) -- WebSocket fuzz
- [fuzz_raw](fuzz-raw.md) -- Raw byte fuzz
- [Fuzzer](../features/fuzzer.md) -- Fuzzer feature guide
