# intercept

Act on intercepted requests, responses, or streaming frames currently held by the proxy. Held items are released, modified and forwarded, or dropped.

The intercept rule engine dispatches via type-switch on the held envelope's `Message`, routing to per-protocol rule engines (`internal/rules/{http,ws,grpc,sse,raw,common}/`). The MCP tool surface accepts a discriminated union of typed modify payloads -- exactly one of `http`, `ws`, `grpc_start`, `grpc_data`, or `raw` must be supplied for `modify_and_forward`, and it must match the held envelope's `Message` type.

## Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `action` | string | Yes | Action to perform: `release`, `modify_and_forward`, `drop` |
| `params` | object | Yes | Common parameters (see below) |
| `http` | object | Conditional | Typed modify payload for an `HTTPMessage` envelope |
| `ws` | object | Conditional | Typed modify payload for a `WSMessage` envelope |
| `grpc_start` | object | Conditional | Typed modify payload for a `GRPCStartMessage` envelope |
| `grpc_data` | object | Conditional | Typed modify payload for a `GRPCDataMessage` envelope |
| `raw` | object | Conditional | Typed modify payload for a `RawMessage` envelope |

### params

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `intercept_id` | string | Yes | ID of the held envelope |
| `mode` | string | No | Forwarding mode: `"structured"` (default) routes through the typed dispatch; `"raw"` expects `raw_override_base64` and forwards a synthetic `RawMessage` envelope verbatim |
| `raw_override_base64` | string | Conditional | Base64-encoded raw bytes for raw-mode forwarding (max 10 MiB). Required when `mode` is `"raw"` and the held envelope is non-Raw |

## Actions

### release

Forward the held envelope as-is.

### modify_and_forward

Apply a typed modify payload (matching the held envelope's `Message` type) and forward the result. The typed modify schemas are described below. With `mode="raw"` plus `raw_override_base64`, the proxy builds a synthetic `RawMessage` envelope from the supplied bytes and forwards them verbatim regardless of the original protocol.

### drop

Discard the held envelope and unblock the pipeline.

## Typed modify payloads

Headers and metadata are always **ordered arrays** of `{name, value}` objects. Map shapes are rejected to preserve wire fidelity (RFC-001 §3.1: no normalization).

### http

For `HTTPMessage` envelopes. Pointer-style fields (`*string`, `*int`, `*bool`) distinguish "field omitted" from "field set to zero value"; an omitted field leaves the held envelope's field untouched.

| Field | Type | Description |
|-------|------|-------------|
| `method` | string | HTTP method override (request side) |
| `scheme` | string | Scheme override (request side) |
| `authority` | string | Authority override (request side) |
| `path` | string | Request path override |
| `raw_query` | string | Raw query string override |
| `status` | integer | HTTP status code override (response side) |
| `status_reason` | string | HTTP/1.x status reason phrase override |
| `headers` | array | Ordered header list replacement |
| `trailers` | array | Ordered trailer list replacement |
| `body` | string | Body replacement (`text` or `base64` per `body_encoding`) |
| `body_encoding` | string | `"text"` or `"base64"` |
| `body_patches` | array | Body patches applied on top of the body replacement |
| `auto_content_length` | boolean | Auto-sync `Content-Length` on body change. Default `true`; set `false` to preserve CL/TE for smuggling tests |

### ws

For `WSMessage` envelopes.

| Field | Type | Description |
|-------|------|-------------|
| `opcode` | string or integer | Opcode name (`text`, `binary`, `close`, `ping`, `pong`, `continuation`) or numeric opcode in `[0, 15]` |
| `fin` | boolean | FIN bit override |
| `payload` | string | Frame payload (`text` or `base64` per `body_encoding`) |
| `body_encoding` | string | `"text"` or `"base64"` |
| `close_code` | integer | RFC 6455 status code (Close frames only) |
| `close_reason` | string | Close reason text (Close frames only) |

### grpc_start

For `GRPCStartMessage` envelopes (HEADERS frame opening one side of an RPC).

| Field | Type | Description |
|-------|------|-------------|
| `service` | string | gRPC service name override |
| `method` | string | gRPC method name override |
| `encoding` | string | `grpc-encoding` override |
| `metadata` | array | Ordered metadata list replacement (transport pseudo-headers excluded) |

Trailers belong to a distinct `GRPCEndMessage` envelope and are out of scope for `grpc_start`.

### grpc_data

For `GRPCDataMessage` envelopes (one length-prefixed message on a gRPC stream).

| Field | Type | Description |
|-------|------|-------------|
| `payload` | string | Decompressed gRPC payload (`text` or `base64` per `payload_encoding`) |
| `payload_encoding` | string | `"text"` or `"base64"` |
| `compressed` | boolean | Set the compression bit in the LPM prefix |
| `end_stream` | boolean | `END_STREAM` flag on the carrying H2 DATA frame |

### raw

For `RawMessage` envelopes. `bytes_override` and `patches` are mutually exclusive.

| Field | Type | Description |
|-------|------|-------------|
| `bytes_override` | string | Replacement bytes (`text` or `base64` per `bytes_encoding`) |
| `bytes_encoding` | string | `"text"` or `"base64"` |
| `patches` | array | Byte-level patches applied to the held envelope's `RawMessage.Bytes` |

## Response

| Field | Type | Description |
|-------|------|-------------|
| `intercept_id` | string | ID of the held envelope |
| `action` | string | Action performed |
| `status` | string | Result status (`"released"`, `"forwarded"`, `"dropped"`) |
| `protocol` | string | Held envelope's message-type discriminator (`"http"`, `"websocket"`, `"grpc_start"`, `"grpc_data"`, `"grpc_end"`, `"raw"`) |
| `direction` | string | Envelope direction (`"send"`, `"receive"`) |
| `matched_rules` | string[] | Rule names that fired to hold the envelope (when present) |
| `flow_id` | string | Flow id of the held envelope (when present) |
| `stream_id` | string | Stream id of the held envelope (when present) |

## Examples

### Release a held HTTP request

```json
// intercept
{
  "action": "release",
  "params": {"intercept_id": "int-abc-123"}
}
```

### Modify and forward an HTTP request

```json
// intercept
{
  "action": "modify_and_forward",
  "params": {"intercept_id": "int-abc-123"},
  "http": {
    "method": "POST",
    "headers": [
      {"name": "Authorization", "value": "Bearer injected-token"}
    ],
    "body": "{\"role\":\"admin\"}"
  }
}
```

### Modify and forward a WebSocket frame

```json
// intercept
{
  "action": "modify_and_forward",
  "params": {"intercept_id": "int-ws-456"},
  "ws": {
    "opcode": "text",
    "payload": "{\"action\":\"admin\"}"
  }
}
```

### Patch a gRPC payload

```json
// intercept
{
  "action": "modify_and_forward",
  "params": {"intercept_id": "int-grpc-789"},
  "grpc_data": {
    "payload": "AAAAAAk=",
    "payload_encoding": "base64"
  }
}
```

### Forward synthetic raw bytes (smuggling test)

```json
// intercept
{
  "action": "modify_and_forward",
  "params": {
    "intercept_id": "int-abc-123",
    "mode": "raw",
    "raw_override_base64": "R0VUIC8gSFRUUC8xLjENCkhvc3Q6IGV4YW1wbGUuY29tDQoNCg=="
  }
}
```

### Drop a held request

```json
// intercept
{
  "action": "drop",
  "params": {"intercept_id": "int-abc-123"}
}
```

## Related pages

- [Intercept](../features/intercept.md) -- Intercept feature guide
- [configure](configure.md) -- Configure intercept rules and queue settings
- [query](query.md) -- Query the intercept queue with `{"resource": "intercept_queue"}`
