# resend_grpc

Resend a gRPC RPC via a freshly dialled HTTP/2 upstream connection with `GRPCStartMessage` / `GRPCDataMessage` / `GRPCEndMessage`-typed schema. Each call dials a fresh TCP (and TLS for `https`) connection and opens one HTTP/2 stream.

## Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `flow_id` | string | No | | Recorded gRPC stream id. When set, omitted Start fields and the encoding hint are inherited from the original RPC |
| `target_addr` | string | Conditional | | Upstream `host:port`. Required when `flow_id` is empty. When supplied with `flow_id`, redirects the dial target while preserving the recovered `:authority`. Without it, the dial follows the recorded, client-declared `:authority` -- see [What decides the dial](#what-decides-the-dial-with-flow_id) |
| `scheme` | string | No | `"https"` | `"http"` or `"https"`. `http` selects plaintext h2c. With `flow_id`, the recorded `:scheme` is used, except that a recovered `http` is dialed over TLS when the proxy observed a TLS upstream for that stream. An explicit `"http"` always forces cleartext |
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
| `body_encoding` | string | No | `"text"` | `"text"`, `"base64"`, `"proto-json"`, or `"proto-schemaless-json"` |
| `compressed` | boolean | No | `false` | Set the LPM compression flag. Requires `encoding` to be set |

#### Body encodings

- `"text"` / `"base64"` -- send the literal bytes after decoding the text/base64 wrapper. Use these when you have the wire payload in hand.
- `"proto-json"` -- the payload is JSON keyed by the registered `.proto`'s field names. The proxy parses it against the schema bound to `(service, method)` via [`grpc_schema`](grpc-schema.md), encodes it to proto wire bytes, and LPM-frames it. `protojson.UnmarshalOptions{DiscardUnknown:true}` silently drops keys outside the schema; type mismatches still fail. Without a registered schema this returns a hard error.
- `"proto-schemaless-json"` -- synthetic-key JSON via `internal/encoding/protobuf`. Lossless for unknown fields and extensions, at the cost of opaque numeric keys.

When `flow_id` is supplied and the source flow carries fields outside the registered schema, the response includes a non-fatal `warnings[]` entry. See [Lossy round-trip](grpc-schema.md#lossy-round-trip-warning) on the `grpc_schema` page for the full mechanics.

### End semantics

- When `trailer_metadata` is omitted (the common case): the final `messages` LPM carries `END_STREAM=true` and the request terminates via END_STREAM on the last DATA frame.
- When `trailer_metadata` is supplied: the trailing DATA keeps `END_STREAM=false` and a Send-direction trailer HEADERS frame is sent afterwards (a non-standard but diagnostic-useful client-side trailer).

### What decides the dial with `flow_id`

A gRPC stream's recorded `:authority` and `:scheme` are copied verbatim from the client's HEADERS block, so both are **client-declared and untrusted**. In `flow_id` mode:

- **Address** -- without `target_addr`, the dial goes to the recorded `:authority`. Pass `target_addr` to pin the destination when the recorded client may have been hostile. Every such call reports this in `warnings[]`.
- **Transport** -- a recovered `scheme=http` is reconciled against the TLS state the proxy observed on the upstream leg. When TLS was observed, the resend dials with TLS anyway and reports it in `warnings[]`, so a spoofed `:scheme: http` cannot replay recorded `authorization` metadata in cleartext. An unobserved TLS state never downgrades a recovered `https`.
- **Wire fidelity** -- the resent HEADERS frame still carries the recorded `:scheme`; only the TCP/TLS handshake differs.
- **Overrides win** -- an explicit `scheme` is always honored.

!!! note "tcp_forwards false positive"
    With a `tcp_forwards` entry that has a plaintext client leg and `upstream_tls: true`, the recorded `:authority` is the plaintext forward listener while the observed TLS belongs to the upstream beyond it. Pass `"scheme": "http"` for that topology.

## Response

| Field | Type | Description |
|-------|------|-------------|
| `stream_id` | string | New stream record id holding the resend's send and receive flows |
| `start_metadata` | array | Ordered receive-direction GRPCStart metadata `[{"name": "...", "value": "..."}]` |
| `messages` | array | Decoded response-side LPMs (see below) |
| `end` | object | Optional trailer summary (see below). May be `null` when the upstream terminated without a trailer HEADERS frame |
| `duration_ms` | integer | Total duration in milliseconds |
| `tag` | string | Echo of the supplied tag (when set) |
| `warnings` | string[] | Optional non-fatal advisories. Dial-provenance and observed-transport TLS-upgrade notices come first, followed by `proto-json` unknown-field notices |

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
- [grpc_schema](grpc-schema.md) -- register `.proto` schemas for `body_encoding="proto-json"`
- [Resender](../features/resender.md) -- Resender feature guide
