# resend_ws

Resend a single WebSocket frame via a freshly dialled upstream connection with `WSMessage`-typed schema fields. Each call performs a fresh TCP (and TLS for `wss`) dial, an HTTP/1.1 Upgrade dance, then sends one frame and records the response.

## Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `flow_id` | string | No | | Recorded WebSocket stream id. When set, the upgrade dance inherits URL, request headers, and the negotiated `permessage-deflate` extension |
| `target_addr` | string | Conditional | | Upstream `host:port`. Overrides the dial target while preserving the recovered `:authority`. Required when `flow_id` is empty |
| `scheme` | string | Conditional | `"ws"` | `"ws"` or `"wss"`. Required when `flow_id` is empty |
| `path` | string | Conditional | | Upgrade request path. Required when `flow_id` is empty |
| `raw_query` | string | No | | Upgrade request raw query string without the leading `?` |
| `opcode` | string | Yes | | Frame opcode: `"text"`, `"binary"`, `"close"`, `"ping"`, `"pong"` |
| `fin` | boolean | No | `true` | FIN bit |
| `payload` | string | No | | Frame payload interpreted per `body_encoding` |
| `body_encoding` | string | No | `"text"` | `"text"` or `"base64"` |
| `payload_set` | boolean | No | `false` | Set `true` to send an empty payload; otherwise an empty `payload` field is treated as no override |
| `masked` | boolean | No | | Informational. The upstream-facing layer auto-masks per RFC 6455 §5.3 regardless of this value |
| `mask` | string | No | | Informational 4-byte mask key (base64). Ignored on Send for client-to-server frames |
| `close_code` | integer | No | | RFC 6455 status code for Close frames |
| `close_reason` | string | No | | Optional UTF-8 reason for Close frames |
| `compressed` | boolean | No | | Per-message-deflate (RFC 7692). Requires the upgrade to negotiate deflate via `flow_id` |
| `timeout_ms` | integer | No | `30000` | Per-call timeout in milliseconds covering dial + upgrade + send + receive |
| `tls_fingerprint` | string | No | | Informational v1; per-call selection is deferred |
| `tag` | string | No | | Tag stored on the new flow's `Tags` map |

CR/LF in `path`, `raw_query`, `scheme`, or `target_addr` is rejected to prevent request smuggling on the upgrade leg.

## Response

| Field | Type | Description |
|-------|------|-------------|
| `stream_id` | string | New stream record id holding the resend's send Flow plus every received frame |
| `opcode` | string | Opcode of the upstream's terminating frame (first non-control frame or Close) |
| `fin` | boolean | FIN bit on the result frame |
| `payload` | string | Result frame payload interpreted per `payload_encoding` |
| `payload_encoding` | string | `"text"` or `"base64"` |
| `compressed` | boolean | `true` when the result frame used per-message-deflate |
| `close_code` | integer | RFC 6455 status code (Close frames only) |
| `close_reason` | string | Close reason text (Close frames only) |
| `duration_ms` | integer | Total duration in milliseconds |
| `tag` | string | Echo of the supplied tag (when set) |

Auto-Pong replies for incoming Pings are emitted by the receive loop and recorded as their own flows on the same stream.

## Pipeline placement

The resend traverses `PluginStepPost -> RecordStep`. `PluginStepPre` and `InterceptStep` are bypassed (RFC-001 §9.3).

The resent stream is finalised on completion: `Stream.State` transitions to `"complete"` on success or `"error"` on failure (USK-789). The new flow records `origin = "resend"` so callers can filter live capture out of analyses via `query` `filter.origin`.

## Examples

### Send a text frame to a recorded stream

```json
// resend_ws
{
  "flow_id": "ws-abc-123",
  "opcode": "text",
  "payload": "{\"action\":\"ping\"}"
}
```

### Send a binary frame from scratch

```json
// resend_ws
{
  "target_addr": "ws.example.com:443",
  "scheme": "wss",
  "path": "/socket",
  "opcode": "binary",
  "payload": "AAECAw==",
  "body_encoding": "base64"
}
```

### Send a Close frame

```json
// resend_ws
{
  "flow_id": "ws-abc-123",
  "opcode": "close",
  "close_code": 1000,
  "close_reason": "bye"
}
```

## Related pages

- [resend_http](resend-http.md) -- HTTP request resend
- [resend_grpc](resend-grpc.md) -- gRPC RPC resend
- [resend_raw](resend-raw.md) -- Raw bytes resend
- [fuzz_ws](fuzz-ws.md) -- WebSocket frame fuzz with the same schema
- [Resender](../features/resender.md) -- Resender feature guide
