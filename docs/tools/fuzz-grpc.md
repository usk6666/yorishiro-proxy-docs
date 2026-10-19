# fuzz_grpc

Synchronously fuzz a gRPC unary RPC with `GRPCStartMessage` / `GRPCDataMessage`-typed positions. The schema mirrors [`resend_grpc`](resend-grpc.md) plus a `positions[]` list. Each variant becomes one independent gRPC stream -- fresh dial, fresh `ConnID`, fresh `StreamID`.

This tool is **synchronous**: every variant runs in-process and the full per-variant result list is returned when finished. There is no concurrency or rate limit -- pace from the client side.

## Limits

- Maximum variants per call: **1000** (cartesian product across all positions)
- Maximum positions per call: **32**
- Maximum decoded payload size per position: **1 MiB**

Each variant is re-checked against the current [target scope](../features/target-scope.md) on its post-substitution message (including `service` / `method` positions), then passes per-variant `SafetyFilter` input gating, immediately before the upstream send. A scope change made mid-run (an agent `security` call or a `SIGHUP` policy reload) takes effect on the next variant. A blocked variant is recorded with `error` (no flow) and the run continues.

## Parameters

`fuzz_grpc` inherits every base field from [`resend_grpc`](resend-grpc.md) (`flow_id`, `target_addr`, `scheme`, `service`, `method`, `metadata`, `encoding`, `accept_encoding`, `messages`, `trailer_metadata`, `tls_fingerprint`, `timeout_ms`, `tag`).

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| (base fields) | -- | -- | -- | See [`resend_grpc`](resend-grpc.md). When `flow_id` is empty, `target_addr` + `service` + `method` are required. `messages[]` must have at least one element |
| `positions` | array | Yes | | Ordered position list (see below). At least one entry |
| `stop_on_non_ok` | boolean | No | `false` | Abort remaining variants once any variant returns a non-OK gRPC status (or terminates without a trailer) |
| `pre_macro` | object | No | | Macro hook run before a variant (see [Macro hooks](#macro-hooks)) |
| `post_macro` | object | No | | Macro hook run after a variant (see [Macro hooks](#macro-hooks)) |

### positions

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `path` | string | Yes | | Typed path: `service`, `method`, `metadata[N].name`, `metadata[N].value`, `messages[N].payload` |
| `payloads` | string[] | Yes | | List of values to substitute at this path. At least one element |
| `encoding` | string | No | `"text"` | `"text"` or `"base64"`. Applies to every payload |

`scheme`, `target_addr`, and `encoding` are intentionally **not** fuzz positions -- they affect connection setup. Issue separate `fuzz_grpc` calls when you need to vary those.

## Macro hooks

`pre_macro` / `post_macro` take the same shape and semantics as on [`fuzz_http`](fuzz-http.md#macro-hooks) (`name`, `scope`, `on_error`, `vars`, `run_interval`, `n`, `status_codes`, `match_pattern`), with these gRPC-specific differences:

| Hook | When it fires |
|------|---------------|
| `pre_macro` (`scope="iteration"`) | Before each variant's dial (before the TLS handshake / HTTP/2 SETTINGS) |
| `post_macro` (`scope="iteration"`) | After each variant's end trailer (or stream termination on abnormal close) |
| `pre_macro` (`scope="job"`) | Once before the variant loop |
| `post_macro` (`scope="job"`) | Once after the variant loop -- on natural completion or a `stop_on_non_ok` exit, not on cancellation |

`post_macro` with `scope="iteration"` receives these reserved keys:

| Key | Source |
|-----|--------|
| `__response_status` | gRPC end-trailer status code (`0`-`16`) |
| `__response_status_message` | gRPC end-trailer status message (empty on OK) |
| `__response_body` | Concatenated receive-direction DATA payloads, capped at 64 KiB |
| `__response_message_count` | Number of receive-direction DATA messages |
| `__response_total_bytes` | Pre-truncation byte total of all receive-direction DATA payloads |

Header and trailer-metadata keys are not populated; drill into the variant's `stream_id` with [`query`](query.md) instead.

!!! warning "gRPC status codes, not HTTP"
    `status_codes` for `run_interval="on_status"` matches the **gRPC** status (`0` OK, `7` PERMISSION_DENIED, `14` UNAVAILABLE, `16` UNAUTHENTICATED, ...). HTTP codes such as `503` never match.

`run_interval="on_error"` fires when the previous variant ended with a transport error (dial, TLS / HTTP/2 preface failure, deadline) **or** a non-OK gRPC status. `match_pattern` for `run_interval="on_match"` is matched against `__response_body`.

Unlike `fuzz_http`, `fuzz_grpc` does not expand `§var§` templates in the variant's metadata or messages. Use the hooks for side effects (session setup, teardown, alerting) around each variant.

## Response

| Field | Type | Description |
|-------|------|-------------|
| `fuzz_id` | string | ID of the `fuzz_jobs` row for this run. Pass it to [`query`](query.md) `fuzz_results` to aggregate variants |
| `total_variants` | integer | Total variant count |
| `completed_variants` | integer | Variants actually executed before completion or stop |
| `stopped_reason` | string | Empty when all variants ran; otherwise the stop reason |
| `variants` | array | Per-variant result rows (see below) |
| `duration_ms` | integer | Total run duration in milliseconds |
| `tag` | string | Echo of the supplied tag (when set) |
| `warnings` | string[] | Non-fatal advisories about the whole run (for example, the Mode A dial-provenance notice or an observed-transport TLS upgrade), mirroring [`resend_grpc`](resend-grpc.md) |

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

## Job and result persistence

Synchronous fuzz runs persist a row to `fuzz_jobs` and one row per variant to `fuzz_results`, so `query` (`resource = "fuzz_jobs"` / `"fuzz_results"`) can list and aggregate variants after the call returns. Per-variant flows carry `origin = "fuzz"`.

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

### Post hook gated on UNAVAILABLE

```json
// fuzz_grpc
{
  "flow_id": "grpc-flow-123",
  "positions": [
    {"path": "messages[0].payload", "encoding": "base64", "payloads": ["CgVhbGljZQ==", "AAEC", "////"]}
  ],
  "post_macro": {
    "name": "alert-on-unavailable",
    "run_interval": "on_status",
    "status_codes": [14]
  }
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
- [macro](macro.md) -- Define the macros used by `pre_macro` / `post_macro`
