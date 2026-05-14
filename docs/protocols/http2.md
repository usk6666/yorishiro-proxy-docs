# HTTP/2

yorishiro-proxy supports HTTP/2 in both cleartext (h2c) and TLS (h2) modes. The HTTP/2 Layer (`internal/layer/http2/`) operates per TCP connection: one Layer corresponds to one connection (or one TLS session over one TCP connection), and one Channel corresponds to one HTTP/2 stream — including server-pushed streams.

L7 structured view: yes. L4 raw bytes: yes. The Layer uses a custom frame engine and exposes an event-granular Channel, with a per-stream `BodyBuffer`.

## Custom frame engine

The Layer uses its own HTTP/2 frame engine instead of `golang.org/x/net/http2`. This gives the proxy full frame-level control over connection management, header compression, and raw byte recording. The custom implementation consists of:

- **Frame reader/writer** (`frame.Reader` / `frame.Writer`) -- reads and writes individual HTTP/2 frames with size enforcement per `SETTINGS_MAX_FRAME_SIZE`
- **HPACK codec** (`hpack.Encoder` / `hpack.Decoder`) -- custom header compression and decompression per RFC 7541
- **Transport** -- an HTTP/2 upstream transport with connection pooling (`internal/connector/transport/`) that reuses the same frame engine

Every frame read from the wire has its raw bytes preserved verbatim on `Envelope.Raw`, enabling both structured L7 views and raw L4 byte inspection.

## Frame types

The frame engine implements all ten HTTP/2 frame types defined in RFC 9113 Section 6:

| Type | Code | Description |
|------|------|-------------|
| DATA | 0x00 | Carries request/response body data |
| HEADERS | 0x01 | Opens a stream and carries HPACK-compressed headers |
| PRIORITY | 0x02 | Specifies stream priority (dependency and weight) |
| RST_STREAM | 0x03 | Terminates a stream with an error code |
| SETTINGS | 0x04 | Communicates connection-level parameters |
| PUSH_PROMISE | 0x05 | Reserves a server-initiated stream. The proxy advertises `SETTINGS_ENABLE_PUSH = 0` on the client role and no longer ferries push promises end-to-end (server push has been retired) |
| PING | 0x06 | Measures round-trip time and keeps connections alive |
| GOAWAY | 0x07 | Initiates graceful connection shutdown |
| WINDOW_UPDATE | 0x08 | Manages flow control window sizes |
| CONTINUATION | 0x09 | Continues a header block that did not fit in a single HEADERS frame |

Each frame has a 9-byte header (24-bit length, 8-bit type, 8-bit flags, 31-bit stream ID) followed by the payload. The proxy parses typed fields for each frame type -- for example, SETTINGS frames are decoded into key-value pairs, GOAWAY frames expose the last stream ID and error code, and HEADERS frames have their padding and priority fields stripped before HPACK decoding.

## HPACK codec

The proxy includes a custom HPACK implementation (RFC 7541) for header compression and decompression. The codec supports:

- **Static table** -- the 61 pre-defined header fields from RFC 7541 Appendix A
- **Dynamic table** -- per-connection table that grows/shrinks based on `SETTINGS_HEADER_TABLE_SIZE`
- **Huffman coding** -- optional Huffman encoding of string literals for further compression
- **Integer encoding** -- variable-length integer encoding per RFC 7541 Section 5.1

The decoder enforces safety limits to prevent resource exhaustion:

- Maximum header list size: 64 KB (total decoded header fields per block)
- Maximum string length: 16 KB (per header name or value)

Both client-to-proxy and proxy-to-upstream connections maintain independent HPACK state, allowing the proxy to fully decode, inspect, and re-encode headers at each hop.

### SETTINGS emitted by the proxy

The proxy is conservative about what it advertises so that it does not silently disagree with operator intent:

- **`SETTINGS_ENABLE_PUSH`** -- omitted from `ServerRole` (RFC 9113 §7.2.2 makes the field meaningless from a server) and set to `0` on `ClientRole`. End-to-end server push is no longer recorded.
- **`SETTINGS_ENABLE_CONNECT_PROTOCOL`** -- when an upstream advertises this in its SETTINGS, the proxy mirrors the value to clients so RFC 8441 extended `CONNECT` (the WebSocket-over-HTTP/2 bootstrap) keeps working end-to-end.
- **`SETTINGS_MAX_HEADER_LIST_SIZE`** -- omitted when the value equals the default (the wire saves a `(k,v)` pair and stays bit-for-bit equivalent to a peer that never sent it).
- **`SETTINGS_MAX_CONCURRENT_STREAMS`** -- defaults to **500** (raised from 100 in USK-862). Override per listener via [`proxy_start.max_concurrent_streams`](../tools/proxy-start.md) or globally via the [config file](../configuration/config-file.md). Range `1..65535`.

### Connection-specific headers stripped on send

When forwarding requests upstream the proxy strips the RFC 7540 §8.1.2.2 connection-specific headers (`Connection`, `Keep-Alive`, `Proxy-Connection`, `Transfer-Encoding`, `Upgrade`, plus any `te` value other than `trailers`). This prevents accidentally re-emitting HTTP/1.1 hop-by-hop headers over an HTTP/2 connection.

## Raw frame recording

Every HTTP/2 frame's raw bytes are preserved as they appear on the wire on `Envelope.Raw`. This follows the L7-first, L4-capable architecture principle — you get structured headers, methods, and bodies by default, but you can always access the exact bytes.

You can view raw bytes through:

- The **query** tool with `resource: "flow"` — the `raw_bytes` field contains the wire-observed bytes
- The **WebUI Raw tab** — displays the hex dump of the wire-observed frames

When a request is modified by an intercept rule, the proxy records two send messages: the original wire-observed bytes (variant `"original"`) and the modified version (variant `"modified"`). The original raw bytes are never altered.

## Event-granular Channel

The HTTP/2 Layer exposes the wire as discrete events through its Channel:

- `H2HeadersEvent` — initial HEADERS or trailer HEADERS frame (carrying the `END_STREAM` bit when applicable)
- `H2DataEvent` — DATA frame payload
- `H2TrailersEvent` — trailer HEADERS-after-DATA blocks

A Channel does **not** assume request/response pairing. Sequence is event-order, numbered from 0.

Pipeline Steps that want to operate on full request/response pairs (e.g. intercept rules, transforms) are placed downstream of the **HTTPAggregator** (`internal/layer/httpaggregator/`), which folds the event stream into one `HTTPMessage` envelope per completed message. The connector's `dispatchH2Stream` helper picks between the aggregator and the [gRPC Layer](grpc.md) by peeking the first `H2HeadersEvent` for content-type.

## Per-stream BodyBuffer

DATA frames are aggregated into a reference-counted `BodyBuffer` before each `HTTPMessage` envelope is yielded. The buffer starts in memory and promotes to a temp file once the cumulative body size crosses `BodySpillThreshold` (default 10 MiB). Total size is capped by `MaxBodySize` (default 254 MiB). Exceeding the cap surfaces a `*layer.StreamError` with `Code=ErrorInternalError` and the reader emits `RST_STREAM(INTERNAL_ERROR)` for the offending stream.

## Flow control decoupled from Pipeline latency

`WINDOW_UPDATE` frames are emitted at frame arrival, not after the Pipeline has finished processing the envelope. The Layer eagerly emits a `WINDOW_UPDATE` when the stream-level recv window has been consumed by ≥50% of the local `InitialWindowSize`, or when the connection-level recv window has been consumed by ≥50% of the initial 65535. This means an arbitrarily long Pipeline hold (e.g. waiting on intercept review) on one stream does **not** stall sibling streams sharing the same connection.

## Trailer support

HTTP/2 trailers are fully supported. Trailers are sent as a HEADERS frame with the `END_STREAM` flag set, arriving after all DATA frames for a stream. The proxy records trailers in the flow's response message headers.

This is particularly important for gRPC, which relies on trailers to carry `grpc-status` and `grpc-message`. See the [gRPC page](grpc.md) for details on trailer handling in gRPC flows.

## h2c (cleartext HTTP/2)

Cleartext HTTP/2 connections are detected by the HTTP/2 connection preface — the string `PRI * HTTP/2.0\r\n` that clients send at the start of an h2c connection.

When the connector detects this preface in the peeked bytes, the ConnectionStack is built around the HTTP/2 Layer directly without any TLS Layer.

```json
// proxy_start
{
  "listen_addr": "127.0.0.1:8080"
}
```

Clients that support h2c can connect directly:

```bash
curl --http2-prior-knowledge http://127.0.0.1:8080/api/endpoint
```

## h2 (HTTP/2 over TLS)

HTTP/2 over TLS is handled through the HTTPS MITM tunnel. The TLS Layer (`internal/layer/tlslayer/`) terminates TLS on both the client and upstream sides; ALPN routing then chooses the HTTP/2 Layer when both peers agree on `h2`. See [HTTPS MITM](https-mitm.md) for the CONNECT + ALPN details.

## Extended CONNECT (WebSocket over HTTP/2)

RFC 8441 extended `CONNECT` is supported in both directions. When a peer advertises `SETTINGS_ENABLE_CONNECT_PROTOCOL = 1`, the proxy mirrors the setting to its counterparty so clients can negotiate `CONNECT` with `:protocol = "websocket"` end-to-end. Once the upgrade succeeds, the H2 Layer hands the post-swap stream to the [WebSocket Layer](websocket.md) and DATA frames flow as WebSocket frames thereafter -- the proxy orchestrates the per-stream sub-stack overlay so that hold/intercept and plugin hooks still receive `WSMessage` envelopes.

## GOAWAY handling

When the upstream emits `GOAWAY` while a stream is held in intercept review, the connector re-dials and replays the held request on a fresh H2 connection so that the operator's release/modify decision is honoured rather than failing the flow.

## Stream-level flow recording

Unlike HTTP/1.x where each connection processes requests sequentially, HTTP/2 multiplexes multiple streams over a single connection. Each stream gets its own Channel and is recorded as a separate flow:

- Stream-level events flow as `H2HeadersEvent` / `H2DataEvent` / `H2TrailersEvent` envelopes through the per-stream Channel
- Plain HTTP/2 traffic is folded into one `HTTPMessage` envelope per request/response by `internal/layer/httpaggregator/`
- gRPC streams are handed to the [gRPC Layer](grpc.md) instead

A single reader goroutine reads frames sequentially and dispatches them to per-stream assembler state stored in a connection-scoped map. A single writer goroutine drains write requests from a queue and serializes them to the wire. This single-writer design satisfies the HPACK encoder's sequential-encoding requirement (RFC 7541 §4.1) and avoids mid-frame interleaving.

## Request processing pipeline

Each HTTP/2 stream follows this processing pipeline:

1. **Read request body** -- the full body is read and buffered
2. **Resolve scheme and host** -- determines `https` (for h2) or `http` (for h2c) and sets the target host
3. **Target scope check** -- enforces which destinations are allowed
4. **Rate limit check** -- enforces per-host rate limits
5. **Safety filter** -- checks request body and URL against safety rules
6. **Plugin hooks** -- dispatches `on_receive_from_client`
7. **Build outbound request** -- creates the upstream request with cleaned headers
8. **Intercept check** -- pauses for AI agent review if matching rules exist
9. **Plugin hooks** -- dispatches `on_before_send_to_server`
10. **Forward upstream** -- sends the request and reads the response
11. **Response intercept** -- allows modification of the response
12. **Plugin hooks** -- dispatches response hooks
13. **Output filter** -- masks sensitive data before sending to client
14. **Write response** -- sends the response back to the client
15. **Record flow** -- stores the complete request/response

## Differences from HTTP/1.x

| Aspect | HTTP/1.x | HTTP/2 |
|--------|----------|--------|
| Multiplexing | Sequential (one request at a time per connection) | Concurrent streams on a single connection |
| Flow recording | One flow per request | One flow per stream |
| Protocol detection | HTTP method prefix (`GET `, `POST `, etc.) | Connection preface (`PRI * HTTP/2.0`) or ALPN `h2` |
| Header format | Text-based | Binary (HPACK compressed) |
| Frame engine | N/A (custom parser, `net/http` not used in data path) | Custom frame engine + event-granular Channel |
| Channel granularity | One `HTTPMessage` per request | Per-frame events folded into `HTTPMessage` by `httpaggregator/` |

## gRPC detection

When the connector's `dispatchH2Stream` helper detects a `Content-Type: application/grpc` (or `application/grpc+proto`, `application/grpc+json`, etc.) on a stream by peeking the first `H2HeadersEvent`, it routes the Channel to the [gRPC Layer](grpc.md) instead of the aggregator. The stream is still carried by the HTTP/2 Layer, but envelopes surface as `GRPCStartMessage` / `GRPCDataMessage` / `GRPCEndMessage`.

## Upstream connection

Upstream HTTP/2 connections live in `internal/connector/transport/`. The transport manages a connection pool keyed by `host:port` and supports:

- Standard `crypto/tls` connections
- Custom TLS transports (e.g., uTLS fingerprint spoofing, mTLS)
- Configurable dial timeouts (default 30 seconds)

The upstream Layer is pooled — its lifetime is independent of the client-facing stack — and the connector returns it to the pool (or evicts on failure) when the per-stack callback exits.

## Limitations

- **No h2c upgrade from HTTP/1.1** -- the proxy only supports h2c via the connection preface (`PRI`), not the HTTP/1.1 Upgrade mechanism
- **Body size limits** -- the per-stream `BodyBuffer` enforces `MaxBodySize` (default 254 MiB); exceeding the cap surfaces a `*layer.StreamError` and emits `RST_STREAM(INTERNAL_ERROR)`
- **No CONNECT-UDP** -- only TCP-based HTTP/2 is supported

## Related pages

- [HTTP/1.x](http.md) -- plaintext HTTP handling
- [HTTPS MITM](https-mitm.md) -- TLS interception and ALPN negotiation
- [gRPC](grpc.md) -- gRPC protocol handling over HTTP/2
- [gRPC-Web](grpc-web.md) -- gRPC-Web rides on HTTP/1.x or HTTP/2
- [Intercept](../features/intercept.md) -- pausing and modifying HTTP/2 streams
