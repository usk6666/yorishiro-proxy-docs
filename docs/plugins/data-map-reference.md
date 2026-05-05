# Data map reference

This page documents the `msg` argument shape passed to each hook function. The shape depends on the protocol and event — for example, `http.on_request` receives an `HTTPMessage` view, `ws.on_message` receives a `WSMessage` view.

## Naming convention

Field names are mechanically converted from the underlying Go struct's PascalCase identifier to `snake_case`:

| Go field        | Dict key         |
|-----------------|------------------|
| `Method`        | `method`         |
| `RawQuery`      | `raw_query`      |
| `StatusReason`  | `status_reason`  |
| `EndStream`     | `end_stream`     |
| `ContentType`   | `content_type`   |
| `AcceptEncoding`| `accept_encoding`|

Acronym runs collapse to a single lowercase block (`URL` → `url`, `JA3` → `ja3`); a run followed by a lowercase letter breaks correctly (`HTTPCode` → `http_code`).

## Shared shape conventions

### `headers` / `trailers` / `metadata`

Header-typed fields are exposed as a `headers` value: an order-preserved, case-preserved sequence of `(name, value)` 2-tuples. Iteration and indexing yield 2-tuples; mutation goes through `.append(name, value)`, `.replace_at(i, name, value)`, `.delete_first(name)`, and the read-only `.get_first(name)`. See [Writing plugins → Headers and trailers](writing-plugins.md#headers-and-trailers).

To bulk-replace, assign a new sequence of 2-tuples:

```python
msg["headers"] = [("Host", "example.com"), ("X-Auth", "abc")]
```

### `raw`

Every `msg` exposes a writable `"raw"` key bound to the wire-observed bytes of the envelope. Reads return `bytes`. Assigning `bytes` injects the new value verbatim onto the wire — see [Writing plugins → `msg["raw"]`](writing-plugins.md#msgraw-byte-injection). Per RFC §9.3 D4, when both `raw` and structured fields are mutated, raw wins.

### `anomalies`

Read-only field on `HTTPMessage`, `GRPCStartMessage`, `GRPCEndMessage`, and `SSEMessage`. A list of frozen 2-key dicts (`{"type": str, "detail": str}`) describing parser-detected wire anomalies (CL/TE conflict, malformed framing, etc.). Assigning to it raises a load-time error; iterate it for read-only inspection.

### Body fields

`body` (HTTP) and `payload` (WS, gRPC data, raw bytes) are `bytes`. HTTP bodies larger than 1 MiB cause the plugin to be skipped for that envelope (the body is never presented in truncated form). To mutate larger payloads, write `msg["raw"]` directly.

## HTTPMessage (`http.on_request`, `http.on_response`, `ws.on_upgrade`)

| Key             | Type    | Direction  | Description                                                       |
|-----------------|---------|------------|-------------------------------------------------------------------|
| `method`        | string  | request    | HTTP method (`GET`, `POST`, ...).                                 |
| `scheme`        | string  | request    | `"http"` or `"https"`.                                            |
| `authority`     | string  | request    | `Host` header or `:authority` pseudo-header.                       |
| `path`          | string  | request    | URL path.                                                         |
| `raw_query`     | string  | request    | Raw query string without leading `?`.                              |
| `status`        | int     | response   | HTTP status code.                                                 |
| `status_reason` | string  | response   | HTTP/1.x reason phrase (empty for HTTP/2).                        |
| `headers`       | headers | both       | Order- and case-preserved header list.                            |
| `trailers`      | headers | both       | Order- and case-preserved trailer list.                           |
| `body`          | bytes   | both       | Body bytes (1 MiB cap; oversize skips the plugin).                |
| `anomalies`     | list    | both, RO   | Parser-detected anomalies.                                        |
| `raw`           | bytes   | both       | Wire-observed bytes; writable.                                    |

Request fields (`method`, `scheme`, `authority`, `path`, `raw_query`) are valid when the envelope is a request; response fields (`status`, `status_reason`) are valid when it is a response. Reading a request field on a response envelope yields a zero value (empty string / `0`).

## WSMessage (`ws.on_message`, `ws.on_close`)

| Key            | Type   | Description                                                       |
|----------------|--------|-------------------------------------------------------------------|
| `opcode`       | int    | RFC 6455 opcode: `0x0`=continuation, `0x1`=text, `0x2`=binary, `0x8`=close, `0x9`=ping, `0xA`=pong. |
| `fin`          | bool   | Final-fragment bit.                                               |
| `masked`       | bool   | True for client-to-server frames.                                 |
| `mask`         | bytes  | 4-byte masking key when `masked` is true.                          |
| `payload`      | bytes  | Unmasked payload.                                                 |
| `close_code`   | int    | RFC 6455 close status (uint16). Zero for non-Close frames.        |
| `close_reason` | string | Optional UTF-8 close reason.                                      |
| `compressed`   | bool   | RSV1 bit (per-message-deflate, RFC 7692).                         |
| `raw`          | bytes  | Wire-observed bytes; writable.                                    |

For `ws.on_close` the dict is frozen — mutations are ignored. Otherwise the same shape applies.

## GRPCStartMessage (`grpc.on_start`, `grpc-web.on_start`)

| Key               | Type    | Description                                                    |
|-------------------|---------|----------------------------------------------------------------|
| `service`         | string  | gRPC service name (derived from `:path`).                      |
| `method`          | string  | gRPC method name.                                              |
| `metadata`        | headers | gRPC metadata list. Pseudo-headers are not included here.      |
| `timeout`         | int     | Parsed `grpc-timeout` in nanoseconds. Zero when unset.         |
| `content_type`    | string  | `application/grpc[+proto|+json|...]`.                          |
| `encoding`        | string  | Parsed `grpc-encoding` (`identity`, `gzip`, ...).              |
| `accept_encoding` | list[str] | Parsed `grpc-accept-encoding` list. Reassign as a list to mutate. |
| `anomalies`       | list    | Read-only parser anomalies.                                    |
| `raw`             | bytes   | Wire-observed bytes; writable.                                 |

## GRPCDataMessage (`grpc.on_data`, `grpc-web.on_data`)

| Key           | Type   | Description                                                              |
|---------------|--------|--------------------------------------------------------------------------|
| `service`     | string | Read-only; denormalized from the start message.                          |
| `method`      | string | Read-only; denormalized from the start message.                          |
| `compressed`  | bool   | First byte of the 5-byte length-prefixed-message (LPM) prefix.            |
| `wire_length` | int    | uint32 length field of the LPM prefix (compressed-payload length).        |
| `payload`     | bytes  | Decompressed payload bytes regardless of `compressed`.                    |
| `end_stream`  | bool   | Mirrors the wire-level `END_STREAM` flag of the underlying H2 DATA frame. |
| `raw`         | bytes  | Wire-observed bytes (5-byte LPM prefix + payload as on the wire); writable. |

To inject malformed compressed bytes or alternate framing, write `msg["raw"]` directly — `payload` is always presented decompressed.

## GRPCEndMessage (`grpc.on_end`, `grpc-web.on_end`)

| Key              | Type    | Description                                                  |
|------------------|---------|--------------------------------------------------------------|
| `status`         | int     | uint32 grpc-status code (`0`=OK, `1`=CANCELLED, ...).        |
| `message`        | string  | Percent-decoded `grpc-message` value.                        |
| `status_details` | bytes   | Raw protobuf bytes of `grpc-status-details-bin`, if present. |
| `trailers`       | headers | Remaining trailer metadata after status/message/details are stripped. |
| `anomalies`      | list    | Read-only parser anomalies.                                  |

For lifecycle dispatch (`on_end` is `none`-phase) the dict is frozen.

## SSEMessage (`sse.on_event`)

| Key         | Type   | Description                                                     |
|-------------|--------|-----------------------------------------------------------------|
| `event`     | string | Parsed event name (`event:` field).                             |
| `data`      | string | Joined `data:` lines, separated by newline.                     |
| `id`        | string | `id:` field (last-event-id).                                    |
| `retry`     | int    | `retry:` field as nanoseconds. Zero when unset.                 |
| `anomalies` | list   | Read-only parser anomalies.                                     |
| `raw`       | bytes  | Wire-observed bytes; writable.                                  |

## RawMessage (`raw.on_chunk`)

| Key     | Type  | Description                                                     |
|---------|-------|-----------------------------------------------------------------|
| `bytes` | bytes | The bytes received in one Read call (or sent in one Write call). |
| `raw`   | bytes | Same as `bytes` for `raw.on_chunk`; writable. (Mutating `bytes` and `raw` follow the same "raw wins" rule.) |

## Lifecycle dicts

Lifecycle events (phase `none`) deliver a frozen dict instead of a `MessageDict`. There is no `raw` key and mutations are ignored — see [Hook reference → Lifecycle events in detail](hook-reference.md#lifecycle-events-in-detail) for the per-event keys.

## Envelope-level identifiers

Envelope-level fields — `StreamID`, `FlowID`, `Sequence`, `Direction`, `Protocol` — are not exposed as `msg` keys. Group events that belong to the same wire stream by stashing values into `ctx.stream_state` (per `(ConnID, StreamID)`) or `ctx.transaction_state`. See [Writing plugins → The `ctx` argument](writing-plugins.md#the-ctx-argument).

## Related pages

- [Hook reference](hook-reference.md) — Which event delivers which message shape.
- [Writing plugins](writing-plugins.md) — Mutation API and sandbox modules.
- [Examples](examples.md) — Ready-to-use plugin samples.
