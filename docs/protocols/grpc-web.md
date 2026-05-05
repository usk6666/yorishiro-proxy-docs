# gRPC-Web

yorishiro-proxy records gRPC-Web traffic with a dedicated Layer (`internal/layer/grpcweb/`). gRPC-Web is the browser-facing variant of gRPC: instead of relying on HTTP/2 trailers (which most browsers cannot reach from JavaScript), it embeds the trailer block as a final length-prefixed frame inside the HTTP body and rides on either HTTP/1.1 or HTTP/2.

L7 structured view: yes. L4 raw bytes: yes (via HTTP/1.x or HTTP/2). The Layer parses the framing and yields the same `GRPCStartMessage` / `GRPCDataMessage` / `GRPCEndMessage` envelope types as the [native gRPC Layer](grpc.md), so intercept rules and tooling that type-switch on `env.Message` are shared between the two transports.

## Why gRPC-Web needs a separate Layer

Native gRPC over HTTP/2 relies on three things browsers cannot do directly:

1. Forcing HTTP/2 from JavaScript
2. Sending arbitrary HTTP/2 frames (e.g. inspecting trailer HEADERS)
3. Reading the final trailer headers attached to a response

gRPC-Web sidesteps all three: it uses regular HTTP requests (which a browser can issue), it carries the binary length-prefixed-message (LPM) framing in the request and response bodies, and it appends the gRPC trailer block as a final LPM frame inside the response body (with the high bit of the flags byte set).

Because the wire format differs from native gRPC, a separate Layer parses it. Because the Message types are the same, downstream Pipeline Steps and plugins do not need to know which transport produced the envelope — they see `GRPCStart` / `Data` / `End` either way.

## Wire formats

gRPC-Web supports two wire formats discriminated by Content-Type:

| Wire format | Content-Type | Body encoding |
|-------------|---------------|----------------|
| Binary | `application/grpc-web`, `application/grpc-web+proto`, `application/grpc-web+json` | LPM frames as binary bytes |
| Base64 (text) | `application/grpc-web-text`, `application/grpc-web-text+proto`, `application/grpc-web-text+json` | LPM frames base64-encoded |

The Layer detects the encoding from the inbound `HTTPMessage`'s content-type header and parses LPM frames out of the body accordingly. For binary wire, `Envelope.Raw` on emitted `GRPCDataMessage` / `GRPCEndMessage` envelopes carries the raw 5-byte prefix + payload bytes. For base64 wire, `Envelope.Raw` is kept in its base64-encoded form so analysts running fuzzers see the bytes exactly as they appeared on the wire.

## Channel composition

`grpcweb.Wrap` consumes any `layer.Channel` whose `Next` produces `*envelope.HTTPMessage` envelopes:

- HTTP/1.x bodies arrive fully buffered (`Body` or `BodyBuffer` non-nil) directly from `internal/layer/http1/`
- HTTP/2 bodies arrive after `internal/layer/httpaggregator/` has folded the H2 event stream into a single `HTTPMessage`

This makes gRPC-Web Channel-type-agnostic: the wrapper does not type-assert on the inner Channel implementation, and the connector picks the appropriate underlying stack (HTTP/1.x or HTTP/2 + aggregator) based on what the client used.

## Embedded trailer frame

On the response side, gRPC-Web encodes the trailer block as a final LPM frame whose flags byte has the high bit set (MSB=1, "trailer flag"). The Layer:

1. Parses preceding LPM frames into `GRPCDataMessage` envelopes
2. Detects the trailer-flagged final frame
3. Decodes its text payload as `name: value\r\n` lines
4. Emits a `GRPCEndMessage` envelope with parsed `Status`, `Message`, `StatusDetails`, and any remaining `Trailers`

On the request side, gRPC-Web bodies must **not** carry an embedded trailer frame; the trailer is a response-side artifact only. If the Layer observes one on a Send-direction body it stamps the `AnomalyUnexpectedGRPCWebRequestTrailer` anomaly on the resulting `GRPCEndMessage` so the malformed bytes remain visible.

## Anomalies

The Layer records observability signals for recoverable wire-format deviations without terminating the stream:

| Anomaly | Meaning |
|---------|---------|
| `MalformedGRPCWebBase64` | Content-Type advertises `grpc-web-text` but the body is not valid base64. The bytes are preserved verbatim on `Envelope.Raw`. |
| `MalformedGRPCWebLPM` | LPM framing is malformed: incomplete 5-byte header, invalid flags byte, or declared payload length exceeds remaining bytes. |
| `MalformedGRPCWebTrailer` | The embedded trailer frame's text payload could not be parsed as `name: value\r\n` lines. |
| `MissingGRPCWebTrailer` | A Receive-direction body parsed cleanly but had no terminating trailer LPM frame. The Layer synthesizes a `GRPCEndMessage` with `Status=0` and an empty `Raw` slice (so it is distinguishable from a wire-observed End) and stamps this anomaly on it. |
| `UnexpectedGRPCWebRequestTrailer` | A Send-direction (request) body contained an embedded trailer LPM frame. |

Stream-terminating problems (oversize LPM, gzip-bomb cap, unsupported encoding) continue to surface as `*layer.StreamError` instead of these anomalies — they are termination contracts, not parser deviations.

## Compression

Identity (passthrough) and gzip are supported. Other `grpc-encoding` values produce a `*layer.StreamError{Code: ErrorInternalError}` on both Send and Receive paths. The decompressed-length cap is enforced via `io.LimitReader` to mitigate decompression-bomb attacks.

## Envelope.Protocol asymmetry

Emitted envelopes carry `Envelope.Protocol = grpc-web` to tag the actual transport. The Message types' `Protocol()` method, however, returns `grpc` because `GRPCStart`/`Data`/`End` are shared by type identity with the native gRPC Layer. Pipeline Steps that care about transport (HTTP/1 vs HTTP/2 framing, base64 vs binary) inspect `env.Protocol`; Steps that type-switch on Message see `GRPC*` and share intercept rules with native gRPC.

## Querying gRPC-Web flows

```json
// query
{
  "resource": "flows",
  "filter": {"protocol": "grpc-web"}
}
```

## Replaying a gRPC-Web envelope

Because gRPC-Web shares Message types with native gRPC, the same `resend_grpc` typed tool variant replays envelopes from either transport:

```json
// resend_grpc
{
  "flow_id": "flow-abc-123",
  "envelope_index": 0
}
```

The Layer reuses the original `Envelope.Protocol` to pick the correct wire format on replay (binary vs base64, HTTP/1.x vs HTTP/2 + aggregator).

## Limitations

- **Compression algorithms** — only `identity` and `gzip` are supported; other `grpc-encoding` values surface as a stream error
- **Reassembly cap** — LPMs exceeding `MaxGRPCMessageSize` (254 MiB) on the wire or after gunzip are rejected
- **No protobuf decoding** — payloads are stored as raw bytes; the Layer does not decode Protocol Buffers messages

## Related pages

- [gRPC](grpc.md) — native gRPC over HTTP/2 sharing the same Message types
- [HTTP/1.x](http.md) — one of the underlying transports
- [HTTP/2](http2.md) — the other underlying transport (folded by `httpaggregator/`)
- [resend_grpc](../tools/resend-grpc.md) — replay an envelope (gRPC-Web reuses the gRPC tools)
- [fuzz_grpc](../tools/fuzz-grpc.md) — mutate and replay envelopes (gRPC-Web reuses the gRPC tools)
