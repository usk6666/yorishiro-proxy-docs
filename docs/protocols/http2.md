# HTTP/2

yorishiro-proxy supports HTTP/2 in both cleartext (h2c) and TLS (h2) modes. Each HTTP/2 stream is recorded as an individual flow, providing per-request visibility even when multiple requests share a single connection.

## Custom frame engine

yorishiro-proxy uses its own HTTP/2 frame engine instead of `golang.org/x/net/http2`. This gives the proxy full frame-level control over connection management, header compression, and raw byte recording. The custom implementation consists of:

- **Frame reader/writer** (`frame.Reader` / `frame.Writer`) -- reads and writes individual HTTP/2 frames with size enforcement per `SETTINGS_MAX_FRAME_SIZE`
- **HPACK codec** (`hpack.Encoder` / `hpack.Decoder`) -- custom header compression and decompression per RFC 7541
- **Transport** -- an HTTP/2 upstream transport with connection pooling that uses the custom frame engine for all upstream communication

Every frame read from the wire has its raw bytes preserved in the `Frame.RawBytes` field, enabling both structured L7 views and raw L4 byte inspection.

## Frame types

The frame engine implements all ten HTTP/2 frame types defined in RFC 9113 Section 6:

| Type | Code | Description |
|------|------|-------------|
| DATA | 0x00 | Carries request/response body data |
| HEADERS | 0x01 | Opens a stream and carries HPACK-compressed headers |
| PRIORITY | 0x02 | Specifies stream priority (dependency and weight) |
| RST_STREAM | 0x03 | Terminates a stream with an error code |
| SETTINGS | 0x04 | Communicates connection-level parameters |
| PUSH_PROMISE | 0x05 | Reserves a server-initiated stream (not used by the proxy) |
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

## Raw frame recording

Every HTTP/2 frame's raw bytes are preserved as they appear on the wire. This follows the L7-first, L4-capable architecture principle -- you get structured headers, methods, and bodies by default, but you can always access the exact bytes.

For each flow message, raw frames are concatenated into `Message.RawBytes` with metadata:

| Metadata key | Description |
|-------------|-------------|
| `h2_frame_count` | Number of HTTP/2 frames in the message |
| `h2_total_wire_bytes` | Total bytes across all frames |
| `h2_truncated` | `"true"` if total wire bytes exceeded the 2 MB capture limit |

You can view raw bytes through:

- The **query** tool with `resource: "flow"` -- the `raw_bytes` field contains the concatenated frame bytes
- The **WebUI Raw tab** -- displays the hex dump of the wire-observed frames

When a request is modified by an intercept rule, the proxy records two send messages: the original wire-observed bytes (variant `"original"`) and the modified version (variant `"modified"`). The original raw bytes are never altered.

## Trailer support

HTTP/2 trailers are fully supported. Trailers are sent as a HEADERS frame with the `END_STREAM` flag set, arriving after all DATA frames for a stream. The proxy records trailers in the flow's response message headers.

This is particularly important for gRPC, which relies on trailers to carry `grpc-status` and `grpc-message`. See the [gRPC page](grpc.md) for details on trailer handling in gRPC flows.

## h2c (cleartext HTTP/2)

Cleartext HTTP/2 connections are detected by the HTTP/2 connection preface -- the string `PRI * HTTP/2.0\r\n` that clients send at the start of an h2c connection.

When the proxy detects this preface in the peeked bytes, the connection is dispatched directly to the HTTP/2 handler without any TLS involvement.

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

HTTP/2 over TLS is handled through the HTTPS MITM tunnel. During the TLS handshake with the client, the proxy advertises both `h2` and `http/1.1` via ALPN (Application-Layer Protocol Negotiation):

1. Client sends `CONNECT example.com:443` to the proxy
2. Proxy responds with `200 Connection Established`
3. Client initiates TLS handshake, proposing `h2` and `http/1.1` in ALPN
4. If `h2` is negotiated, the proxy delegates to the HTTP/2 handler via `HandleH2`
5. If `http/1.1` is negotiated, the connection stays with the HTTP/1.x handler

This means the protocol is automatically selected based on what the client and proxy negotiate -- no manual configuration is needed.

## Stream-level flow recording

Unlike HTTP/1.x where each connection processes requests sequentially, HTTP/2 multiplexes multiple streams over a single connection. The proxy records each stream as a separate flow:

- Each stream gets its own flow with `FlowType: "unary"`
- Stream-level request and response messages are recorded independently
- Concurrent streams on the same connection do not interfere with each other

The custom frame engine dispatches each stream to a separate goroutine. All in-flight stream goroutines are tracked and waited on before the connection is closed, ensuring complete flow recording.

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
| Frame engine | N/A (standard library) | Custom frame reader/writer with raw byte capture |
| Hop-by-hop headers | `Connection`, `Keep-Alive`, `Proxy-Connection`, `Transfer-Encoding`, `Upgrade`, `Te`, `Trailer`, `Proxy-Authenticate`, `Proxy-Authorization` | `Connection`, `Keep-Alive`, `Proxy-Connection`, `Transfer-Encoding`, `Upgrade` |

## gRPC detection

When the HTTP/2 handler detects a `Content-Type: application/grpc` (or `application/grpc+proto`, `application/grpc+json`, etc.) on a stream, it delegates flow recording to the [gRPC handler](grpc.md). The stream is still proxied by the HTTP/2 handler, but the recorded flow includes gRPC-specific metadata (service, method, grpc-status).

## Upstream connection

The proxy uses the custom HTTP/2 transport (`Transport`) for upstream connections. The transport manages a connection pool keyed by `host:port` and supports:

- Standard `crypto/tls` connections
- Custom TLS transports (e.g., uTLS fingerprint spoofing, mTLS)
- Configurable dial timeouts (default 30 seconds)

Raw response frames from the upstream are captured in `RoundTripResult.RawFrames` for L4 recording.

## Limitations

- **No server push** -- HTTP/2 server push is not supported by the proxy
- **No h2c upgrade from HTTP/1.1** -- the proxy only supports h2c via the connection preface (`PRI`), not the HTTP/1.1 Upgrade mechanism
- **Body size limits** -- request and response bodies are limited to the configured maximum body size
- **Raw capture limit** -- raw frame bytes are capped at 2 MB per message; frames exceeding this are truncated (indicated by `h2_truncated` metadata)
- **No CONNECT-UDP** -- only TCP-based HTTP/2 is supported

## Related pages

- [HTTP/1.x](http.md) -- plaintext HTTP handling
- [HTTPS MITM](https-mitm.md) -- TLS interception and ALPN negotiation
- [gRPC](grpc.md) -- gRPC protocol handling over HTTP/2
- [Intercept](../features/intercept.md) -- pausing and modifying HTTP/2 streams
