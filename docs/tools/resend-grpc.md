# resend_grpc

Resend a gRPC RPC via a freshly dialled HTTP/2 upstream connection with `GRPCStartMessage` / `GRPCDataMessage` / `GRPCEndMessage`-typed schema. Each call dials a fresh TCP (and TLS for `https`) connection and opens one HTTP/2 stream.

## Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `flow_id` | string | No | | Recorded gRPC stream id. When set, omitted Start fields and the encoding hint are inherited from the original RPC |
| `target_addr` | string | Conditional | | Upstream `host:port`. Required when `flow_id` is empty. When supplied with `flow_id`, redirects the dial target while preserving the recovered `:authority` |
| `scheme` | string | No | `"https"` | `"http"` or `"https"`. `http` selects plaintext h2c |
| `service` | string | Conditional | | gRPC service name (e.g. `"pkg.Greeter"`). Required when `flow_id` is empty |
| `method` | string | Conditional | | gRPC method name (e.g. `"SayHello"`). Required when `flow_id` is empty |
| `metadata` | array | No | | Ordered metadata list as `[{"name": "...", "value": "..."}]`. Preserves wire case, order, and duplicates |
| `encoding` | string | No | | `grpc-encoding` for outgoing messages (`"identity"` or `"gzip"`) |
| `accept_encoding` | string[] | No | | `grpc-accept-encoding` list (e.g. `["gzip", "identity"]`) |
| `messages` | array | Yes | | Request-side LPM list (see below). At least one element required |
| `trailer_metadata` | array | No | | Optional Send-direction trailer HEADERS. When supplied, the request terminates via a trailer frame instead of `END_STREAM` on the last DATA |
| `timeout_ms` | integer | No | `30000` | Per-call timeout in milliseconds |
| `tls_fingerprint` | string | No | | Informational v1; per-call selection is deferred |
| `tag` | string | No | | Tag stored on the new stream's `Tags` map |

### messages

Each entry is one length-prefixed message (LPM) on the gRPC stream:

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `payload` | string | Yes | | LPM payload interpreted per `body_encoding` |
| `body_encoding` | string | No | `"text"` | `"text"` or `"base64"` |
| `compressed` | boolean | No | `false` | Set the LPM compression flag. Requires `encoding` to be set |

### End semantics

- When `trailer_metadata` is omitted (the common case): the final `messages` LPM carries `END_STREAM=true` and the request terminates via END_STREAM on the last DATA frame.
- When `trailer_metadata` is supplied: the trailing DATA keeps `END_STREAM=false` and a Send-direction trailer HEADERS frame is sent afterwards (a non-standard but diagnostic-useful client-side trailer).

## Response

| Field | Type | Description |
|-------|------|-------------|
| `stream_id` | string | New stream record id holding the resend's send and receive flows |
| `start_metadata` | array | Ordered receive-direction GRPCStart metadata `[{"name": "...", "value": "..."}]` |
| `messages` | array | Decoded response-side LPMs (see below) |
| `end` | object | Optional trailer summary (see below). May be `null` when the upstream terminated without a trailer HEADERS frame |
| `duration_ms` | integer | Total duration in milliseconds |
| `tag` | string | Echo of the supplied tag (when set) |

Each `messages[]` entry:

| Field | Type | Description |
|-------|------|-------------|
| `payload` | string | Decompressed LPM payload |
| `payload_encoding` | string | `"text"` or `"base64"` |
| `compressed` | boolean | `true` when the LPM had its compression bit set on the wire |

`end` object:

| Field | Type | Description |
|-------|------|-------------|
| `status` | integer | gRPC status code (`0` = OK) |
| `message` | string | gRPC status message (when non-empty) |
| `trailers` | array | Other trailer fields. Excludes `grpc-status`, `grpc-message`, and `grpc-status-details-bin` |

## Pipeline placement

Each Start and Data envelope traverses `PluginStepPost -> RecordStep`. End is observation-only (RFC-001 §9.3 surface table marks `(grpc, on_end) = PhaseSupportNone`).

The resent stream is finalised on completion: `Stream.State` transitions to `"complete"` on success or `"error"` on failure (USK-789). The new flow records `origin = "resend"`.

## Examples

### Replay a recorded RPC

```json
// resend_grpc
{
  "flow_id": "grpc-abc-123",
  "messages": [
    {"payload": "{\"name\":\"world\"}"}
  ]
}
```

### Construct a new RPC against an h2c target

```json
// resend_grpc
{
  "target_addr": "127.0.0.1:50051",
  "scheme": "http",
  "service": "pkg.Greeter",
  "method": "SayHello",
  "metadata": [
    {"name": "authorization", "value": "Bearer token"}
  ],
  "messages": [
    {"payload": "AAAAAAk=", "body_encoding": "base64"}
  ]
}
```

## Related pages

- [resend_http](resend-http.md) -- HTTP request resend
- [resend_ws](resend-ws.md) -- WebSocket frame resend
- [resend_raw](resend-raw.md) -- Raw bytes resend
- [fuzz_grpc](fuzz-grpc.md) -- gRPC fuzz with the same schema
- [Resender](../features/resender.md) -- Resender feature guide
