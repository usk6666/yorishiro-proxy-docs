# WebSocket

yorishiro-proxy intercepts and records WebSocket connections at the frame level. The WebSocket Layer (`internal/layer/ws/`) parses RFC 6455 wire frames and yields one `WSMessage` envelope per parsed frame, preserving the verbatim wire bytes (header + extended length + mask key + masked payload) on `Envelope.Raw`.

L7 structured view: yes. L4 raw bytes: yes (per frame). Per-message-deflate (RFC 7692) is supported.

## HTTP upgrade detection

WebSocket connections begin as regular HTTP requests with the upgrade mechanism defined in RFC 6455. The HTTP/1.x Layer detects WebSocket upgrades by checking for both headers:

- `Connection: Upgrade` (case-insensitive, may be comma-separated)
- `Upgrade: websocket` (case-insensitive)

When detected, the HTTP/1.x Layer detaches the underlying stream (`http1.Layer.DetachStream`) and the WebSocket Layer takes over. The detached `(reader, writer, closer)` triple — typically with a `*bufio.Reader` so any post-CRLFCRLF bytes are visible — together with a `streamID` and a `Role` (Client / Server) constructs the WSLayer.

!!! note
    WebSocket upgrade requests bypass the safety filter intentionally. Upgrade requests do not carry a meaningful body, so input filtering would not add value.

## Connection flow

### WS (plaintext WebSocket)

```
1. Client ──GET /ws (Upgrade: websocket)──> Proxy
2. Proxy ──GET /ws (Upgrade: websocket)──> Upstream
3. Upstream ──101 Switching Protocols──> Proxy
4. Proxy ──101 Switching Protocols──> Client
5. Bidirectional frame relay with recording
```

### WSS (WebSocket over TLS)

For WSS connections, the WebSocket upgrade happens inside an HTTPS MITM tunnel:

```
1. Client ──CONNECT example.com:443──> Proxy (HTTPS MITM established)
2. Client ──GET /ws (Upgrade: websocket)──> Proxy (inside TLS tunnel)
3. Proxy ──TLS connect──> example.com:443
4. Proxy ──GET /ws (Upgrade: websocket)──> Upstream (over TLS)
5. Upstream ──101 Switching Protocols──> Proxy
6. Proxy ──101 Switching Protocols──> Client
7. Bidirectional frame relay with recording
```

The WSS path uses the configured TLS transport (e.g., uTLS fingerprint) for the upstream connection, and records TLS metadata (version, cipher suite, server certificate subject) on the flow.

## Frame-per-Envelope recording

Each parsed wire frame produces exactly one `WSMessage` envelope. Control frames (Ping / Pong / Close) and continuation frames are **not** coalesced; the Pipeline observes them individually. The Layer never auto-responds to Ping (MITM transparency).

### Text frames (opcode `0x1`)

Stored in `WSMessage.Payload` as UTF-8 text bytes.

### Binary frames (opcode `0x2`)

Stored in `WSMessage.Payload` as raw bytes. Binary payloads are preserved exactly as received.

### Control frames

Control frames (Close `0x8`, Ping `0x9`, Pong `0xA`) are emitted as their own envelopes. A Close frame terminates the Channel.

### Wire-fidelity bytes

`Envelope.Raw` carries the verbatim wire bytes for every frame. RSV2/RSV3 bits are observable only on `Envelope.Raw` — the `WSMessage` struct does not surface them as fields. On `Send` the Layer always emits `RSV2=RSV3=0`; only `RSV1` is set, when permessage-deflate compression is applied.

## Direction tracking

Every message is recorded with a direction:

- **`send`** -- frames from the client to the server (client_to_server)
- **`receive`** -- frames from the server to the client (server_to_client)

Messages use an atomic sequence counter shared across both directions, preserving the interleaved ordering of the conversation.

## Fragment handling

WebSocket allows messages to be split across multiple frames using the fragmentation mechanism:

1. First frame: data opcode (text/binary) with `FIN=0`
2. Continuation frames: opcode `0x0` with `FIN=0`
3. Final frame: opcode `0x0` with `FIN=1`

Each fragment is emitted as its own envelope. For uncompressed messages, every fragment carries that fragment's raw payload bytes verbatim. For compressed (permessage-deflate) messages, the FIN frame's envelope carries the decompressed bytes of the entire reassembled message; the preceding continuation envelopes carry the compressed wire bytes for that fragment.

Send-side fragmentation is the caller's responsibility — the Layer emits exactly one wire frame per `Send` call.

## Flow structure

A WebSocket flow is recorded with:

- **Protocol**: `WebSocket`
- **Flow type**: `bidirectional`
- **State**: `active` while the relay is running, `complete` when the connection closes normally, `error` if a connection error occurs

The flow starts when the WebSocket upgrade succeeds and ends when either side closes the connection or an error occurs.

## Plugin hooks

The WebSocket Layer participates in the standard Pipeline. Plugin authors register hooks via the pluginv2 engine using the `(protocol, event, phase)` triple — for example `(ws, on_frame, pre)` to observe each frame envelope before intercept.

Plugin capabilities (per the canonical 8-step Pipeline):

- **Drop frames** via the Pre-Intercept phase
- **Modify payloads** via Transform / Macro Steps
- **Observe** without modification via Post phase

Control frames (Close, Ping, Pong) are emitted as their own envelopes and are observable by plugins. See the [Plugin hook reference](../plugins/hook-reference.md) for the full surface.

## Payload size limits

- **Individual frame payload**: capped at 16 MB to prevent memory exhaustion
- **Fragmented message**: accumulated size is capped at the configured maximum WebSocket message size
- **Recorded payload**: may be truncated to the configured maximum recording payload size

## Safety filter

The safety filter applies to WebSocket text frames sent from the client to the server (send direction). Binary frames and control frames are not checked.

When a text frame matches a safety filter input rule:

- **Block action** (default) -- The frame is blocked and not forwarded to the upstream server. The frame is still recorded in the flow store with `safety_blocked: true` metadata. If the blocked frame is the first in a fragmented message, all subsequent continuation frames in that message are also dropped.
- **Log-only action** -- The frame is forwarded normally but recorded with `safety_logged: true` metadata for later review.

Response-direction frames (server to client) are checked against the output filter, which can redact sensitive data patterns (PII, credentials) in recorded payloads without affecting the forwarded data.

## Frame-level intercept

Intercept rules can match on WebSocket frames using the `websocket_frame` phase. When a frame matches an intercept rule, it is held in the intercept queue and the WebSocket relay pauses until you take action.

Intercepted WebSocket frames include the following metadata:

| Field | Description |
|-------|-------------|
| `opcode` | Frame opcode name (e.g., `Text`, `Binary`) |
| `direction` | `client_to_server` or `server_to_client` |
| `flow_id` | The WebSocket flow ID this frame belongs to |
| `upgrade_url` | The URL from the original WebSocket upgrade request |
| `sequence` | Frame sequence number within the WebSocket connection |

You can modify the frame payload using `override_body` with the `modify_and_forward` action, or drop the frame entirely with the `drop` action.

## permessage-deflate (RFC 7692)

permessage-deflate is opt-in via the Layer's `WithDeflateEnabled` master switch plus per-direction `WithClientDeflate` / `WithServerDeflate` options. When the upstream server negotiates permessage-deflate in the `Sec-WebSocket-Extensions` response header:

1. **Wire-faithful raw bytes** — `Envelope.Raw` always carries the verbatim wire bytes (compressed if compression was used).
2. **Decompressed payload on the FIN frame** — for fragmented compressed messages, the FIN frame's `WSMessage.Payload` is the decompressed bytes of the entire reassembled message. Continuation envelopes carry the per-fragment compressed bytes; a single fragment is rarely a complete deflate stream and applications should not attempt to decompress one in isolation.
3. **Single-frame compressed messages** — when `Fin=true` on the start frame, `WSMessage.Payload` carries the decompressed bytes directly.

The Layer respects `server_no_context_takeover` and `client_no_context_takeover` parameters and configures decompression window bits per the negotiated extension parameters.

## Limitations

- **No subprotocol awareness** -- the Layer treats all WebSocket traffic as opaque frames regardless of the negotiated subprotocol
- **Send-side fragmentation** -- the Layer emits exactly one wire frame per `Send` call; fragmentation is the caller's responsibility
- **Mask key regeneration** -- on `Send`, `WSMessage.Mask` is informational. The Layer regenerates the 4-byte mask key from `crypto/rand` for every `RoleClient` frame per RFC 6455 §5.3 strong-entropy requirement
- **Late-RST detection** -- the Layer has no background watcher goroutine; if the wire is closed by the peer between `Next` calls, the next `Next` observes either `io.EOF` (graceful) or `*layer.StreamError{Code: ErrorAborted}`

## Related pages

- [HTTP/1.x](http.md) -- the HTTP upgrade mechanism that initiates WebSocket connections
- [HTTPS MITM](https-mitm.md) -- WSS connections through TLS tunnels
- [Intercept feature](../features/intercept.md) -- intercept rules and the WebSocket frame phase
- [WebUI: Intercept](../webui/intercept.md) -- managing intercepted WebSocket frames in the UI
- [resend](../tools/resend.md) -- replay an envelope (per-protocol typed variants for WebSocket)
- [fuzz](../tools/fuzz.md) -- mutate and replay envelopes (per-protocol typed variants for WebSocket)
- [Plugin hook reference](../plugins/hook-reference.md) -- detailed hook documentation
