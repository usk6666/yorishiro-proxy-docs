# Hook reference

This page enumerates every `(protocol, event)` pair the `pluginv2` engine recognizes — the RFC-001 §9.3 hook surface — and documents the valid phases and action permissions for each.

## The 17-entry surface table

Every `register_hook(protocol, event, fn, phase=...)` call must target one of the 17 rows below. Any other `(protocol, event)` pair fails at script load time with `not in RFC §9.3 hook surface`.

| Protocol     | Event            | Phases                          | Allowed actions                | Message                  |
|--------------|------------------|---------------------------------|--------------------------------|--------------------------|
| `http`       | `on_request`     | `pre_pipeline`, `post_pipeline` | `CONTINUE`, `DROP`, `RESPOND`  | `HTTPMessage` (request)  |
| `http`       | `on_response`    | `pre_pipeline`, `post_pipeline` | `CONTINUE`, `RESPOND`          | `HTTPMessage` (response) |
| `ws`         | `on_upgrade`     | `pre_pipeline`, `post_pipeline` | `CONTINUE`, `DROP`, `RESPOND`  | `HTTPMessage` (request)  |
| `ws`         | `on_message`     | `pre_pipeline`, `post_pipeline` | `CONTINUE`                     | `WSMessage`              |
| `ws`         | `on_close`       | `none` (lifecycle)              | `CONTINUE`                     | WS close dict            |
| `grpc`       | `on_start`       | `pre_pipeline`, `post_pipeline` | `CONTINUE`, `DROP`, `RESPOND`  | `GRPCStartMessage`       |
| `grpc`       | `on_data`        | `pre_pipeline`, `post_pipeline` | `CONTINUE`                     | `GRPCDataMessage`        |
| `grpc`       | `on_end`         | `none` (lifecycle)              | `CONTINUE`                     | `GRPCEndMessage` dict    |
| `grpc-web`   | `on_start`       | `pre_pipeline`, `post_pipeline` | `CONTINUE`, `DROP`, `RESPOND`  | `GRPCStartMessage`       |
| `grpc-web`   | `on_data`        | `pre_pipeline`, `post_pipeline` | `CONTINUE`                     | `GRPCDataMessage`        |
| `grpc-web`   | `on_end`         | `none` (lifecycle)              | `CONTINUE`                     | `GRPCEndMessage` dict    |
| `sse`        | `on_event`       | `pre_pipeline`, `post_pipeline` | `CONTINUE`                     | `SSEMessage`             |
| `raw`        | `on_chunk`       | `pre_pipeline`, `post_pipeline` | `CONTINUE`                     | `RawMessage`             |
| `tls`        | `on_handshake`   | `none` (lifecycle)              | `CONTINUE`                     | TLS handshake dict       |
| `connection` | `on_connect`     | `none` (lifecycle)              | `CONTINUE`, `DROP`             | Connection dict          |
| `connection` | `on_disconnect`  | `none` (lifecycle)              | `CONTINUE`                     | Connection dict          |
| `socks5`     | `on_connect`     | `none` (lifecycle)              | `CONTINUE`, `DROP`             | SOCKS5 connect dict      |

The `Message` column links the `msg` argument shape — see [Data map reference](data-map-reference.md) for the per-type field list.

### Reading the action column

`CONTINUE` is always permitted. The other actions are only available where listed:

- `DROP` — drop the connection or envelope. Available on transaction-start events (`http.on_request`, `ws.on_upgrade`, `grpc.on_start`, `grpc-web.on_start`) and connection-level events (`connection.on_connect`, `socks5.on_connect`).
- `RESPOND` — short-circuit with a synthesized response via `action.RESPOND(...)` (HTTP) or `action.RESPOND_GRPC(...)` (gRPC). Available on `http.on_request` and `http.on_response` (replacement of the upstream response — dropping a response would hang the client, so only replacement is allowed), and on the four transaction-start request events.

Mid-stream events (`on_data`, `on_message`, `on_event`, `on_chunk`) accept `CONTINUE` only — terminating a stateful stream uses native termination (gRPC `RST_STREAM`, WS close frame), not an action enum.

A return of `DROP` or `RESPOND` from an event whose surface forbids it is logged and demoted to `CONTINUE` (fail-soft: a misbehaving plugin must not break wire traffic).

## Phases

Pipeline events accept `pre_pipeline` (default) or `post_pipeline`. See [Writing plugins](writing-plugins.md) for the semantics.

Lifecycle events have phase `none`. Passing any `phase=` argument to `register_hook` for a lifecycle event is a load-time error:

```python
# Load-time error: lifecycle event takes no phase=
register_hook("ws", "on_close", on_close, phase="post_pipeline")
```

## Lifecycle events in detail

The seven lifecycle events do not run through the Pipeline Step chain. They fire from the producing Layer with a frozen dict shaped for that specific event. Mutations to the dict are ignored — there is no Envelope to commit changes to.

### `connection.on_connect`

Fires when a new TCP connection is accepted, before any Layer is constructed. Returning `DROP` closes the accepting connection without further setup.

`msg` keys: `conn_id`, `client_addr`, `listener_name`.

### `connection.on_disconnect`

Fires when a connection closes.

`msg` keys: `conn_id`, `client_addr`, `duration_ms`.

### `tls.on_handshake`

Fires after a TLS handshake completes. Fires once for the proxy's MITM-server side and again for the upstream-client side.

`msg` keys: `side` (`"server"` or `"client"`), `sni`, `alpn`, `version_name`, `cipher_name`, `peer_cert_subject`, `client_fingerprint`.

### `socks5.on_connect`

Fires after SOCKS5 CONNECT negotiation succeeds. Returning `DROP` aborts the tunnel before any upstream connection is made.

`msg` keys: `conn_id`, `client_addr`, `target_addr`.

### `ws.on_close`

Fires when the WebSocket channel closes. The `msg` mirrors the close-frame `WSMessage` shape so the same handler can read `msg["close_code"]` regardless of normal vs abnormal closure.

`msg` keys: `opcode`, `fin`, `masked`, `payload`, `close_code`, `close_reason`, `compressed`.

### `grpc.on_end` and `grpc-web.on_end`

Fires when the gRPC RPC terminates. `msg` may be a wire-observed end or a synthesized end (e.g. `Status=2 UNKNOWN` for an abnormal termination).

`msg` keys: `status`, `message`, `status_details`, `trailers` (sequence of `(name, value)` 2-tuples).

## Plugin chain behavior

When multiple plugins register hooks for the same `(protocol, event, phase)`, the dispatcher fires them in registration order:

1. Each hook's mutations to `msg` are visible to subsequent hooks.
2. The first hook that returns `DROP` or `RESPOND` short-circuits the chain.
3. A hook that errors is handled per `on_error`: `"skip"` logs and continues; `"abort"` stops the chain.
4. Lifecycle `connection.on_connect` and `socks5.on_connect` short-circuit on the first `DROP`; the others always run every hook.

## Related pages

- [Writing plugins](writing-plugins.md) — `register_hook` signature, phase semantics, mutation API.
- [Data map reference](data-map-reference.md) — Per-protocol `msg` dict shape.
- [Examples](examples.md) — Ready-to-use plugin samples.
