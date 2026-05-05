# fuzz_ws

Synchronously fuzz a WebSocket frame with `WSMessage`-typed positions. The schema mirrors [`resend_ws`](resend-ws.md) plus a `positions[]` list. Each variant performs a fresh TCP (and TLS for `wss`) dial, an HTTP/1.1 Upgrade dance, sends one frame, and records the response.

This tool is **synchronous**: every variant runs in-process and the full per-variant result list is returned when finished. There is no concurrency or rate limit -- pace from the client side.

## Limits

- Maximum variants per call: **1000** (cartesian product across all positions)

Each variant is executed sequentially with its own connection, upgrade dance, and Stream row. Per-variant `SafetyFilter` input gating runs before the upstream send -- blocked variants are recorded with `error` and the run continues.

## Parameters

`fuzz_ws` inherits every base field from [`resend_ws`](resend-ws.md) (`flow_id`, `target_addr`, `scheme`, `path`, `raw_query`, `opcode`, `fin`, `payload`, `body_encoding`, `payload_set`, `masked`, `mask`, `close_code`, `close_reason`, `compressed`, `tls_fingerprint`, `timeout_ms`, `tag`).

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| (base fields) | -- | -- | -- | See [`resend_ws`](resend-ws.md). `opcode` is required; `target_addr` + `path` are required when `flow_id` is empty |
| `positions` | array | Yes | | Ordered position list (see below). At least one entry |
| `stop_on_close` | boolean | No | `false` | Abort remaining variants once any variant receives a Close frame from upstream |

### positions

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `path` | string | Yes | | Typed path: `payload` or `close_reason` |
| `payloads` | string[] | Yes | | List of values to substitute at this path. At least one element |
| `encoding` | string | No | `"text"` | `"text"` or `"base64"`. Applies to every payload |

## Response

| Field | Type | Description |
|-------|------|-------------|
| `total_variants` | integer | Total variant count from the cartesian product |
| `completed_variants` | integer | Variants actually executed before completion or stop |
| `stopped_reason` | string | Empty when all variants ran; `"stop_on_close: ..."` or similar otherwise |
| `variants` | array | Per-variant result rows (see below) |
| `duration_ms` | integer | Total run duration in milliseconds |
| `tag` | string | Echo of the supplied tag (when set) |

Each variant row:

| Field | Type | Description |
|-------|------|-------------|
| `index` | integer | Variant index (zero-based, in execution order) |
| `stream_id` | string | New stream record id |
| `opcode` | string | Opcode of the upstream's terminating frame |
| `fin` | boolean | FIN bit on the result frame |
| `payload_size` | integer | Result-frame payload size in bytes (full payload not stored on the row) |
| `compressed` | boolean | `true` when the result frame used per-message-deflate |
| `close_code` | integer | RFC 6455 status code on Close frames |
| `close_reason` | string | Close reason text on Close frames |
| `payloads` | object | Map of position path -> decoded payload string |
| `error` | string | Error message when the variant failed |
| `duration_ms` | integer | Per-variant duration |

Full result-frame payloads are not stored on the row (a malicious upstream could amplify memory use up to 1000 variants times the per-frame Layer cap). Retrieve them via the [`query`](query.md) tool keyed by `stream_id`.

## Pipeline placement

Each variant traverses the same self-contained `PluginStepPost -> RecordStep` pipeline as `resend_ws` (`PluginStepPre` is bypassed per RFC-001 §9.3).

## Examples

### Fuzz the payload of a text frame

```json
// fuzz_ws
{
  "flow_id": "ws-abc-123",
  "opcode": "text",
  "positions": [
    {
      "path": "payload",
      "payloads": [
        "{\"action\":\"ping\"}",
        "{\"action\":\"admin\"}",
        "<script>alert(1)</script>"
      ]
    }
  ]
}
```

### Fuzz a close reason

```json
// fuzz_ws
{
  "flow_id": "ws-abc-123",
  "opcode": "close",
  "close_code": 1000,
  "positions": [
    {"path": "close_reason", "payloads": ["bye", "exit", ""]}
  ],
  "stop_on_close": true
}
```

## Related pages

- [resend_ws](resend-ws.md) -- WebSocket frame resend (single shot)
- [fuzz_http](fuzz-http.md) -- HTTP fuzz
- [fuzz_grpc](fuzz-grpc.md) -- gRPC fuzz
- [fuzz_raw](fuzz-raw.md) -- Raw byte fuzz
- [Fuzzer](../features/fuzzer.md) -- Fuzzer feature guide
