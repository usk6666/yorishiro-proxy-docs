# Writing plugins

This guide covers how to write a `pluginv2` Starlark plugin: how to register hooks, the hook function signature, the mutation API for `msg`, and the sandbox modules available to your script.

## Starlark basics

Starlark is a subset of Python designed for configuration and extension. Key differences from Python:

- No `import` statement — use the predeclared modules (`action`, `config`, `crypto`, `state`, `store`, `proxy`).
- No classes; only functions and primitive types (string, bytes, int, float, bool, list, tuple, dict, `None`).
- `print()` outputs to the proxy log as a structured entry with `plugin` and `message` fields.
- No file I/O, network access, or system calls.
- Module-level values are frozen after script load. Use `state` for mutable state across hook calls.

## Anatomy of a plugin

A plugin is a `.star` file. Its top-level body runs once at proxy boot to register hooks; the registered functions then fire on each matching wire event.

```python
def stamp_auth(msg, ctx):
    msg["headers"].append("X-Auth", config["api_key"])
    return None  # CONTINUE

register_hook("http", "on_request", stamp_auth, phase="post_pipeline")
```

The `register_hook` builtin and the predeclared modules are bound to the script's global scope.

## `register_hook(protocol, event, fn, phase="pre_pipeline")`

Subscribe `fn` to fire on `(protocol, event)` at the given phase.

| Argument   | Type     | Description                                                              |
|------------|----------|--------------------------------------------------------------------------|
| `protocol` | string   | One of `http`, `ws`, `grpc`, `grpc-web`, `sse`, `raw`, `tls`, `connection`, `socks5`. |
| `event`    | string   | Protocol-specific event name (see [Hook reference](hook-reference.md)).  |
| `fn`       | callable | Starlark function with signature `fn(msg, ctx)`.                          |
| `phase`    | string   | `"pre_pipeline"` (default) or `"post_pipeline"`. Omit for lifecycle events. |

Returns `None`. Errors are raised at script load time:

- Unknown `(protocol, event)` pair → load fails with `not in RFC §9.3 hook surface`.
- Invalid `phase` value → load fails with `must be "pre_pipeline" or "post_pipeline"`.
- Passing `phase=` to a lifecycle event (`*.on_close`, `*.on_end`, `tls.on_handshake`, `connection.*`, `socks5.on_connect`) → load fails with `this event is lifecycle/observation-only; do not pass phase=`.
- `fn` not callable → load fails.

If a plugin's top-level body raises after some `register_hook` calls have already been made, none of the staged hooks are committed — the partial registration is discarded and the plugin is treated as failed-to-load.

## Hook function signature

Every hook receives two positional arguments:

```python
def my_hook(msg, ctx):
    # ... inspect / mutate msg ...
    return None
```

| Argument | Type   | Description                                                          |
|----------|--------|----------------------------------------------------------------------|
| `msg`    | dict-like | The protocol-specific message. Snake-case keys, plus a writable `raw` key. See [Data map reference](data-map-reference.md). |
| `ctx`    | object | Per-call metadata (`client_addr`, `tls`, `transaction_state`, `stream_state`). |

### Return values

The dispatcher interprets the return value as follows:

| Return                     | Effect                                                       |
|----------------------------|--------------------------------------------------------------|
| `None`                     | CONTINUE with whatever in-place mutations were applied to `msg`. |
| `action.CONTINUE`          | Same as `None`.                                              |
| `action.DROP`              | Drop the connection / envelope (only valid where the surface allows DROP). |
| `action.RESPOND(...)`      | Synthesize an HTTP response and short-circuit the upstream call. |
| `action.RESPOND_GRPC(...)` | Synthesize a gRPC end and short-circuit the upstream call.   |
| Anything else              | Logged as a warning; treated as CONTINUE.                    |

In-place mutations to `msg` are committed regardless of the return value (per RFC §9.3 D2 — "no silent drop"). Even if a hook raises, mutations applied before the error are kept and the hook is treated as CONTINUE.

A return of `action.DROP` or `action.RESPOND(...)` from an event whose surface forbids it is logged and demoted to CONTINUE. See [Hook reference](hook-reference.md) for the per-event action permissions.

## Mutating `msg`

The `msg` argument behaves like a dict over snake-case fields plus a magic `"raw"` key.

### Scalar fields

Reassign with item assignment:

```python
msg["method"] = "POST"
msg["path"] = "/v2" + msg["path"]
```

Unknown keys (typos like `msg["headerz"]`) and read-only fields (`anomalies`, denormalized `service`/`method` on `grpc.on_data`) raise an error at hook return — never silently dropped.

### Headers and trailers

Header- and trailer-typed fields (HTTP `headers`/`trailers`, gRPC `metadata`, gRPC end `trailers`) are exposed as a `headers` value: an order-preserved, case-preserved sequence of `(name, value)` 2-tuples. Iteration and indexing yield 2-tuples; mutation goes through four methods:

| Method                              | Behavior                                                                |
|-------------------------------------|-------------------------------------------------------------------------|
| `headers.append(name, value)`       | Append a new entry. Wire case is preserved verbatim.                     |
| `headers.replace_at(i, name, value)`| Replace the entry at index `i`. Out-of-range raises.                     |
| `headers.delete_first(name)`        | Remove the first ASCII-case-insensitive match. Returns `True`/`False`.  |
| `headers.get_first(name)`           | Read the first ASCII-case-insensitive match. Returns the value or `None`. |

Plus `name in headers` for ASCII-case-insensitive membership.

```python
msg["headers"].append("X-Auth", config["api_key"])
msg["headers"].delete_first("Cookie")
auth = msg["headers"].get_first("authorization")
```

`sort`, `dedup`, `extend`, `clear`, `remove`, `pop`, `insert` are deliberately not exposed — silently normalizing header order or duplicates would violate the wire-fidelity contract. To bulk-replace, assign a new sequence:

```python
msg["headers"] = [("Host", "example.com"), ("X-Auth", "abc")]
```

### `msg["raw"]`: byte injection

`msg["raw"]` reads as the original wire-observed bytes (`bytes`). Assigning to it injects raw wire bytes for the outgoing envelope:

```python
msg["raw"] = b"GET / HTTP/1.1\r\nHost: x\r\n\r\nGET /admin HTTP/1.1\r\nHost: y\r\n\r\n"
```

This is the lever for HTTP request smuggling, malformed-frame injection, and other tests where the structured `msg` view cannot represent what you want on the wire. Per RFC §9.3 D4, when both `msg["raw"]` and message fields are mutated, **raw wins** — the raw bytes ship verbatim and the structured mutations are recorded only in the variant snapshot. The raw write is capped at 16 MiB.

`msg["raw"]` only accepts `bytes`; assigning a string or any other type raises.

## The `ctx` argument

`ctx` exposes per-invocation metadata and the two scoped state dicts.

| Attribute               | Type                  | Description                                                                  |
|-------------------------|-----------------------|------------------------------------------------------------------------------|
| `ctx.client_addr`       | string or `None`      | Client IP (no port). `None` for events with no client connection.            |
| `ctx.tls`               | dict (frozen) or `None` | TLS snapshot: `sni`, `alpn`, `version_name`, `cipher_name`, `peer_cert_subject`, `client_fingerprint`. `None` when no TLS layer is in the stack. |
| `ctx.transaction_state` | scoped dict           | Per-request/response (HTTP) or per-channel (streaming) KV.                   |
| `ctx.stream_state`      | scoped dict           | Per-stream KV (`(ConnID, StreamID)`).                                        |

### Scoped state API

Both `ctx.transaction_state` and `ctx.stream_state` expose:

| Method     | Description                                                       |
|------------|-------------------------------------------------------------------|
| `.get(key)`    | Returns the value or `None`.                                  |
| `.set(key, v)` | Sets the key. Only primitive values: string, bytes, int, float, bool. |
| `.delete(key)` | Removes the key. No-op if absent.                             |
| `.keys()`      | Returns a list of current keys.                               |
| `.clear()`     | Empties this scope.                                           |

Limits per scope: 10,000 keys, 1 MiB per string/bytes value.

The two scopes have different lifetimes and are managed by the producing Layer:

- **transaction_state** — for HTTP, two hooks on the same envelope (a `pre_pipeline` and a `post_pipeline` invocation) share the dict. For streaming protocols (WS, SSE, gRPC, raw), the dict spans the channel's lifetime so an `on_start` hook can stash a value an `on_data` hook reads many frames later.
- **stream_state** — keyed on `(ConnID, StreamID)` regardless of protocol. Useful for grouping events that belong to the same wire stream.

The Layer releases each scope at terminal-state transitions (channel close, RST_STREAM, peer GOAWAY, etc.). After release, captured references read as empty and writes are silently dropped — a stale capture cannot resurrect a freed scope.

## Sandbox modules

These modules are predeclared in every script.

### `action`

Action sentinels and response builders.

| Member                                 | Description                                                       |
|----------------------------------------|-------------------------------------------------------------------|
| `action.CONTINUE`                      | Sentinel return value for "continue with mutations applied".      |
| `action.DROP`                          | Sentinel return value for "drop this envelope" (where surface allows). |
| `action.RESPOND(status, headers, body)` | Builds an HTTP synthesized response. See below.                   |
| `action.RESPOND_GRPC(status, message, trailers)` | Builds a gRPC synthesized end. See below.                         |

#### `action.RESPOND(status_code, headers=[], body=b"")`

Returns a respond value. When the dispatcher sees this, it short-circuits the upstream call and synthesizes the response envelope.

- `status_code`: int in `[100, 999]`.
- `headers`: sequence of `(name, value)` 2-tuples. May be omitted.
- `body`: `bytes` or `string`. May be omitted (empty body).

Valid for `http.on_request`, `http.on_response`, `ws.on_upgrade`, `grpc.on_start`, `grpc-web.on_start`. Other events log a warning and treat the return as CONTINUE.

```python
return action.RESPOND(401, [("WWW-Authenticate", "Bearer")], b"unauthorized")
```

#### `action.RESPOND_GRPC(status, message="", trailers=[])`

Returns a respond value carrying a gRPC end status. Use for `grpc.on_start` / `grpc-web.on_start`.

- `status`: uint32 grpc-status code (e.g. `7` PERMISSION_DENIED, `16` UNAUTHENTICATED).
- `message`: optional `grpc-message` text.
- `trailers`: optional sequence of `(name, value)` 2-tuples.

```python
return action.RESPOND_GRPC(7, "denied", [("x-deny-reason", "policy")])
```

### `config`

Frozen dict containing the plugin's `vars` (see [Configuration](overview.md#configuration)). Read-only.

```python
api_key = config["api_key"]
```

### `state`

Per-plugin volatile in-memory KV. Survives plugin reloads, lost on process restart.

| Member            | Description                                                |
|-------------------|------------------------------------------------------------|
| `state.get(key)`  | Returns the value or `None`.                               |
| `state.set(key, v)` | Sets the key. Primitives only: string, bytes, int, float, bool. |
| `state.delete(key)` | Removes the key.                                         |
| `state.keys()`    | Returns a list of current keys.                            |
| `state.clear()`   | Removes all keys.                                          |

Limits: 10,000 keys, 1 MiB per string/bytes value.

### `store`

Per-plugin persistent KV backed by SQLite. Same API as `state` but values survive process restarts. Only string and bytes values are accepted. Available only when the proxy is started with a database connection — otherwise `store` is not predeclared.

### `crypto`

Wraps Go's standard library hash, HMAC, AES, and encoding helpers. All hash/HMAC functions accept `bytes` and return `bytes`.

| Family   | Functions                                                                |
|----------|--------------------------------------------------------------------------|
| Hash     | `crypto.md5`, `crypto.sha1`, `crypto.sha256`, `crypto.sha512`            |
| HMAC     | `crypto.hmac_md5`, `crypto.hmac_sha1`, `crypto.hmac_sha256`, `crypto.hmac_sha512` |
| AES      | `crypto.aes_encrypt_cbc`, `crypto.aes_decrypt_cbc`, `crypto.aes_encrypt_gcm`, `crypto.aes_decrypt_gcm` |
| Encoding | `crypto.hex_encode`, `crypto.hex_decode`, `crypto.base64_encode`, `crypto.base64_decode`, `crypto.base64url_encode`, `crypto.base64url_decode` |

### `proxy`

Runtime control.

| Member                  | Description                                              |
|-------------------------|----------------------------------------------------------|
| `proxy.shutdown(reason)` | Triggers proxy shutdown with the given non-empty reason string. |

## Resend, Macro, and fuzz semantics

Per RFC §9.3 Decision item 1: when a request is replayed via `resend_*` MCP tools, fanned out via Macro, or generated via `fuzz_*`, **only `phase="post_pipeline"` hooks fire**. `pre_pipeline` hooks run only on fresh wire receive.

The practical implication: a signing or auth-stamping plugin that needs to run on every variant — normal traffic, resend, Macro, fuzz — should register a single `phase="post_pipeline"` hook. No special-case routing is required.

```python
def sign(msg, ctx):
    body = msg["body"]
    sig = crypto.hex_encode(crypto.hmac_sha256(config["secret"], body))
    msg["headers"].append("X-Signature", sig)

register_hook("http", "on_request", sign, phase="post_pipeline")
```

## Errors and debugging

### `on_error: "skip"` vs `"abort"`

| Value             | Behavior on hook error                                       |
|-------------------|--------------------------------------------------------------|
| `"skip"` (default) | Log a warning, continue to the next hook in the chain.       |
| `"abort"`         | Stop the chain and surface the error to the caller.          |

Use `"abort"` during development to surface errors immediately. Switch to `"skip"` in production.

### print() for logging

`print()` in Starlark emits a structured log entry tagged with `plugin` and `hook`:

```python
def trace(msg, ctx):
    print("method=%s path=%s" % (msg["method"], msg["path"]))
```

### `plugin_introspect`

The `plugin_introspect` MCP tool surfaces every loaded plugin and its `(protocol, event, phase)` registrations. Use it to confirm that your `register_hook()` calls landed as expected. See [plugin_introspect](../tools/plugin-introspect.md).

## Related pages

- [Hook reference](hook-reference.md) — The 17-entry surface table.
- [Data map reference](data-map-reference.md) — Per-protocol `msg` dict shape.
- [Examples](examples.md) — Ready-to-use plugin samples.
