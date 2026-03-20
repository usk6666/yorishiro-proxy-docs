# WebSocket

yorishiro-proxy intercepts and records WebSocket connections at the message level. When an HTTP Upgrade request for WebSocket is detected, the proxy establishes the WebSocket tunnel and records each frame as a separate flow message with direction tracking.

## HTTP upgrade detection

WebSocket connections begin as regular HTTP requests with the upgrade mechanism defined in RFC 6455. The proxy detects WebSocket upgrades by checking for both headers before hop-by-hop header removal:

- `Connection: Upgrade` (case-insensitive, may be comma-separated)
- `Upgrade: websocket` (case-insensitive)

When detected, the request is handed off to the WebSocket handler instead of the normal HTTP request processing pipeline.

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

## Message-level recording

Each WebSocket data frame is recorded as a flow message:

### Text frames (opcode `0x1`)

Stored in the message `body` field as UTF-8 text. This makes text messages searchable and human-readable in the flow store.

### Binary frames (opcode `0x2`)

Stored in the message `raw_bytes` field. Binary payloads are preserved exactly as received.

### Control frames

Control frames (Close `0x8`, Ping `0x9`, Pong `0xA`) are recorded with their payload in `raw_bytes`. A Close frame terminates the relay.

### Message metadata

Each recorded message includes metadata:

| Key | Value |
|-----|-------|
| `opcode` | Frame opcode as integer (1=text, 2=binary, 8=close, 9=ping, 10=pong) |
| `fin` | Always `"true"` for recorded messages (fragments are assembled) |
| `masked` | `"true"` if the frame was masked (client-to-server frames) |

## Direction tracking

Every message is recorded with a direction:

- **`send`** -- frames from the client to the server (client_to_server)
- **`receive`** -- frames from the server to the client (server_to_client)

Messages use an atomic sequence counter shared across both directions, preserving the interleaved ordering of the conversation.

## Fragment assembly

WebSocket allows messages to be split across multiple frames using the fragmentation mechanism:

1. First frame: data opcode (text/binary) with `FIN=0`
2. Continuation frames: opcode `0x0` with `FIN=0`
3. Final frame: opcode `0x0` with `FIN=1`

The proxy assembles fragmented messages before recording them, so each recorded message represents a complete logical message. Fragment accumulation is capped at a configurable maximum size to prevent memory exhaustion. If a fragmented message exceeds the limit, the proxy sends a Close frame with status code 1009 (Message Too Big) and terminates the connection.

## Flow structure

A WebSocket flow is recorded with:

- **Protocol**: `WebSocket`
- **Flow type**: `bidirectional`
- **State**: `active` while the relay is running, `complete` when the connection closes normally, `error` if a connection error occurs

The flow starts when the WebSocket upgrade succeeds and ends when either side closes the connection or an error occurs.

## Plugin hooks

The WebSocket handler dispatches plugin hooks for each data frame (not control frames):

| Direction | Hooks dispatched |
|-----------|-----------------|
| Client to server | `on_receive_from_client` then `on_before_send_to_server` |
| Server to client | `on_receive_from_server` then `on_before_send_to_client` |

Plugin capabilities:

- **Drop frames**: returning `ActionDrop` silently skips the frame (only supported in the `send` direction for `on_receive_from_client`)
- **Modify payloads**: returning modified `payload` in the result data updates the frame payload before forwarding
- **Size limits**: if a plugin-modified payload exceeds the maximum WebSocket message size, the modification is discarded and the original payload is preserved

Control frames (Close, Ping, Pong) are **not** sent to plugins. This prevents plugins from dropping Close frames, which would cause the relay to hang.

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

## permessage-deflate

The proxy supports the `permessage-deflate` WebSocket extension (RFC 7692). When the upstream server negotiates permessage-deflate in the `Sec-WebSocket-Extensions` response header, the proxy:

1. **Forwards compressed frames on the wire** -- Compressed frames are relayed between client and server without re-encoding, preserving wire transparency.
2. **Decompresses frames before recording** -- Stored message data is always the decompressed plaintext, making recorded payloads readable and searchable.
3. **Records compression metadata** -- A `compressed: true` metadata flag is set on messages that were decompressed for storage.

The proxy respects `server_no_context_takeover` and `client_no_context_takeover` parameters, and configures decompression window bits per the negotiated extension parameters.

## Limitations

- **No subprotocol awareness** -- the proxy treats all WebSocket traffic as opaque frames regardless of the negotiated subprotocol
- **Frame-level forwarding** -- frames are forwarded individually, preserving masking and fragmentation as received from the client

## Related pages

- [HTTP/1.x](http.md) -- the HTTP upgrade mechanism that initiates WebSocket connections
- [HTTPS MITM](https-mitm.md) -- WSS connections through TLS tunnels
- [Intercept feature](../features/intercept.md) -- intercept rules and the WebSocket frame phase
- [WebUI: Intercept](../webui/intercept.md) -- managing intercepted WebSocket frames in the UI
- [Plugin hook reference](../plugins/hook-reference.md) -- detailed hook documentation
