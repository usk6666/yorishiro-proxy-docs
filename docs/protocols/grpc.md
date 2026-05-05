# gRPC

yorishiro-proxy records gRPC traffic flowing over HTTP/2 with a dedicated gRPC Layer (`internal/layer/grpc/`). The Layer wraps the event-granular HTTP/2 stream Channel produced by `internal/layer/http2/` and emits one Envelope per gRPC event:

- `GRPCStartMessage` — translated from the initial HEADERS frame on either direction
- `GRPCDataMessage` — produced by length-prefixed-message (LPM) reassembly across one or more `H2DataEvent` payloads
- `GRPCEndMessage` — translated from the trailer HEADERS frame, or synthesized from the same `END_STREAM` HEADERS frame in the trailers-only response case

L7 structured view: yes. L4 raw bytes: yes (via HTTP/2). Native LPM reassembly produces `GRPCStart`/`Data`/`End` envelope events.

## How gRPC is detected

gRPC uses HTTP/2 as its transport. Detection lives in the connector's `dispatchH2Stream` helper (`internal/connector/h2_dispatch.go`), which peeks the first `H2HeadersEvent` and inspects the `Content-Type` header on each HTTP/2 stream:

- `application/grpc`
- `application/grpc+proto`
- `application/grpc+json`
- Any value matching `application/grpc+*`

When a matching content type is found, the connector wraps the stream Channel with the gRPC Layer instead of the [HTTPAggregator](http2.md). The gRPC Layer itself contains no detection logic — it is a strict upper-layer boundary.

## Service and method extraction

gRPC encodes the service and method in the URL path following the pattern:

```
/package.ServiceName/MethodName
```

The Layer parses this path and stores the extracted service and method names on `GRPCStartMessage.Service` / `.Method`. For example, a request to `/helloworld.Greeter/SayHello` produces:

- `Service`: `helloworld.Greeter`
- `Method`: `SayHello`

Path parsing is tolerant: malformed `:path` values (missing, empty, no leading slash, no separator, single segment) yield `Service=""` and `Method=""` together with a warning log. The malformed path remains observable on `Envelope.Raw` for diagnostic purposes — the wire bytes are never altered.

## Protobuf framing and LPM reassembly

gRPC messages use a Length-Prefixed Message (LPM) format with a 5-byte header:

```
+------------------+------------------+-------------------+
| Compressed (1B)  | Length (4B, BE)  |  Payload (N bytes) |
+------------------+------------------+-------------------+
```

- **Compressed flag** (1 byte): `0` = uncompressed, `1` = compressed (per `grpc-encoding` header)
- **Length** (4 bytes): big-endian uint32 indicating the payload size
- **Payload** (N bytes): the serialized Protocol Buffers (or JSON) message

The 5-byte LPM prefix is parsed across `H2DataEvent` boundaries. A single LPM may span many DATA events; one DATA event may carry many LPMs. Each reassembled LPM produces one `GRPCDataMessage` envelope.

`Envelope.Raw` for a `GRPCDataMessage` envelope contains the exact wire bytes (5-byte LPM prefix + compressed payload). `GRPCDataMessage.Payload` is always the decompressed bytes for inspection convenience.

### Reassembly limits

The per-channel reassembly buffer is bounded by `config.MaxGRPCMessageSize` (254 MiB). The cap applies to both the wire LPM length (the 4-byte length prefix) and the decompressed length after gunzip — the latter is enforced via `io.LimitReader` inside gunzip to mitigate decompression-bomb attacks. Exceeding the cap yields `*layer.StreamError{Code: ErrorInternalError}` and marks the wrapper terminated.

## Progressive recording

Unlike HTTP/1.x flows that are recorded only after the full request/response cycle, gRPC flows use progressive (envelope-by-envelope) recording. The Layer emits one envelope per gRPC event (`GRPCStartMessage` for HEADERS, `GRPCDataMessage` for each LPM, `GRPCEndMessage` for trailers), so active streams are visible in the flow list before they complete.

### Sequence numbering

A per-channel monotonic counter starts at 0 and increments on every emitted envelope regardless of direction. This is correct for bidirectional streams — sequences interleave Send-direction and Receive-direction events in observation order.

### Streaming transport

The Layer handles all four gRPC streaming patterns natively. Each `H2DataEvent` is parsed into LPM frames and emitted as `GRPCDataMessage` envelopes as soon as a complete LPM is reassembled, without waiting for the stream to end.

### Streaming classification

The proxy classifies each gRPC session based on the number of request and response frames:

| Request frames | Response frames | Flow type |
|---------------|----------------|-----------|
| 0-1 | 0-1 | `unary` |
| >1 | >1 | `bidirectional` |
| Other combinations | | `stream` (client or server streaming) |

This covers all four gRPC patterns:

- **Unary RPC**: single request, single response
- **Server streaming**: single request, multiple responses
- **Client streaming**: multiple requests, single response
- **Bidirectional streaming**: multiple requests, multiple responses

The flow type starts as `"unary"` when the flow is created and is updated to the final classification when the stream completes.

## Trailer handling

gRPC relies on HTTP/2 trailers to carry status information. The Layer translates the trailer HEADERS frame into a `GRPCEndMessage` envelope.

### Standard trailers

In a normal gRPC response, trailers arrive as a HEADERS frame with `END_STREAM` after all DATA frames. The Layer extracts:

- `grpc-status` → `GRPCEndMessage.Status`
- `grpc-message` → `GRPCEndMessage.Message`
- `grpc-status-details-bin` → `GRPCEndMessage.StatusDetails` (raw protobuf bytes)
- All remaining trailer key/values → `GRPCEndMessage.Trailers` (order and casing preserved)

### Trailers-only responses

A trailers-only response occurs when the server sends `grpc-status` in the initial response HEADERS frame with `END_STREAM=true` (no DATA frames). This happens for immediate errors like `UNIMPLEMENTED` or `PERMISSION_DENIED`.

When the Layer sees a Receive-side `H2HeadersEvent` with `EndStream=true` carrying `grpc-status`, it emits **both** a `GRPCStartMessage` envelope (sequence N) **and** a synthetic `GRPCEndMessage` envelope (sequence N+1) parsed from the same headers. The synthetic End envelope has `Envelope.Raw=nil` so analysts can distinguish it from a wire-observed End.

## Flow recording structure

A gRPC flow is recorded with:

- **Protocol**: `grpc`
- **Flow type**: `unary`, `stream`, or `bidirectional`
- **State**: `active` during streaming, `complete` when finished

### Metadata strip set

`GRPCStartMessage.Metadata` excludes pseudo-headers (any name beginning with `:`), `content-type`, `grpc-encoding`, `grpc-accept-encoding`, and `grpc-timeout` — those values surface on dedicated `GRPCStartMessage` fields (`ContentType`, `Encoding`, `AcceptEncoding`, `Timeout`). Order and casing of remaining metadata are preserved.

`GRPCEndMessage.Trailers` excludes `grpc-status`, `grpc-message`, and `grpc-status-details-bin`; those populate `Status`, `Message`, `StatusDetails` respectively.

### gRPC status codes

The proxy extracts the gRPC status from trailers (or response headers for trailers-only responses):

| Code | Name |
|------|------|
| 0 | OK |
| 1 | CANCELLED |
| 2 | UNKNOWN |
| 3 | INVALID_ARGUMENT |
| 4 | DEADLINE_EXCEEDED |
| 5 | NOT_FOUND |
| 6 | ALREADY_EXISTS |
| 7 | PERMISSION_DENIED |
| 8 | RESOURCE_EXHAUSTED |
| 9 | FAILED_PRECONDITION |
| 10 | ABORTED |
| 11 | OUT_OF_RANGE |
| 12 | UNIMPLEMENTED |
| 13 | INTERNAL |
| 14 | UNAVAILABLE |
| 15 | DATA_LOSS |
| 16 | UNAUTHENTICATED |

## Plugin hooks

The gRPC Layer participates in the standard Pipeline. Plugin authors register hooks via the pluginv2 engine using the `(protocol, event, phase)` triple — for example `(grpc, on_data, pre)` to observe each `GRPCDataMessage` envelope before intercept. See the [Plugin hook reference](../plugins/hook-reference.md) for the full surface.

A per-session transaction context (`ctx.transaction_state`) is shared across all hook dispatches within the same gRPC session, allowing plugins to correlate Start / Data / End envelopes.

## Querying gRPC flows

The canonical Envelope.Protocol value for gRPC is `grpc`:

```json
// query
{
  "resource": "flows",
  "filter": {"protocol": "grpc"}
}
```

You can also filter by flow type to find streaming RPCs:

```json
// query
{
  "resource": "flows",
  "filter": {"protocol": "grpc", "flow_type": "bidirectional"}
}
```

## Compression

The Layer's compression policy is strict. v1 supports only `identity` (no-op) and `gzip` (compress/gzip). Any other `grpc-encoding` value on a `Compressed=true` LPM returns `*layer.StreamError{Code: ErrorProtocol}` from `Next` or `Send`.

## Limitations

- **No protobuf decoding** -- payloads are stored as raw bytes; the proxy does not decode Protocol Buffers messages
- **HTTP/2 only** -- this Layer handles gRPC over HTTP/2. For browser-style gRPC over HTTP/1.x or HTTP/2 see [gRPC-Web](grpc-web.md)
- **Compression algorithms** -- only `identity` and `gzip` are supported; other `grpc-encoding` values surface as a stream error
- **Reassembly cap** -- LPMs exceeding `MaxGRPCMessageSize` (254 MiB) on the wire or after gunzip are rejected with a `*layer.StreamError`

## Related pages

- [HTTP/2](http2.md) -- the transport layer for gRPC
- [gRPC-Web](grpc-web.md) -- browser-style gRPC sharing the same `GRPCStart`/`Data`/`End` Message types
- [HTTPS MITM](https-mitm.md) -- TLS interception for gRPC over TLS
- [resend_grpc](../tools/resend-grpc.md) -- replay a gRPC envelope
- [fuzz_grpc](../tools/fuzz-grpc.md) -- mutate and replay gRPC envelopes
- [Plugin hook reference](../plugins/hook-reference.md) -- full hook documentation
