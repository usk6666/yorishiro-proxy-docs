# Plugin system overview

yorishiro-proxy supports user-defined plugins written in [Starlark](https://github.com/google/starlark-go), a Python-like language designed for configuration and extension. Plugins hook into the proxy pipeline to inspect, modify, replace, or block traffic in real time.

The plugin engine is `pluginv2` — the RFC-001 §9.3 Starlark engine. It replaces the legacy `internal/plugin/` engine with a three-axis hook identity model. If you are migrating an older script, see the [plugin-migration guide](https://github.com/usk6666/yorishiro-proxy/blob/main/docs/rfc/plugin-migration.md).

## Design principles

### Fail-open

If a plugin errors at runtime, the default behavior (`on_error: "skip"`) logs the error and continues processing. Traffic is never silently blocked by a broken plugin unless you explicitly set `on_error: "abort"`.

### Sandboxed execution

Starlark scripts run with a per-call step budget (default: 1,000,000 steps; configurable via `max_steps`) to prevent runaway loops. Scripts cannot access the filesystem, network, or other system resources directly — only the predeclared modules (`state`, `crypto`, `store`, `proxy`, `action`, `config`).

### Registration order

Hooks fire in the order their plugins are loaded, and within a plugin in the order `register_hook()` is called. A plugin's mutations to `msg` are visible to subsequent plugins in the same chain.

### Hook subscriptions live in the script

Unlike the legacy engine, hook subscriptions are no longer declared in the config file. Each plugin calls `register_hook(protocol, event, fn, phase=...)` from its top-level body to subscribe its own functions. This keeps the surface under the operator's version control alongside the script logic and makes the plugin self-describing.

## The three-axis hook identity

Every hook is identified by a `(protocol, event, phase)` tuple.

### Protocol

The wire protocol the hook observes:

| Protocol      | Surface                                                    |
|---------------|------------------------------------------------------------|
| `http`        | HTTP/1.x and HTTP/2 request/response messages              |
| `ws`          | WebSocket upgrade, frames, and close                       |
| `grpc`        | gRPC over HTTP/2 (start metadata, data frames, end status) |
| `grpc-web`    | gRPC-Web over HTTP/1.x or HTTP/2                           |
| `sse`         | Server-Sent Events                                         |
| `raw`         | Raw TCP / TLS-passthrough byte chunks                      |
| `tls`         | TLS handshake observation                                  |
| `connection`  | Connection lifecycle (accept / close)                      |
| `socks5`      | SOCKS5 CONNECT negotiation                                 |

### Event

The protocol-specific wire event the hook fires on. Examples: `on_request`, `on_response`, `on_message`, `on_chunk`, `on_event`, `on_start`, `on_data`, `on_end`, `on_upgrade`, `on_close`, `on_handshake`, `on_connect`, `on_disconnect`. The same event name may appear under multiple protocols with distinct semantics — see [Hook reference](hook-reference.md) for the full enumeration.

### Phase

When the hook fires relative to the Pipeline Step chain:

| Phase           | Position                                                    | Default for                |
|-----------------|-------------------------------------------------------------|----------------------------|
| `pre_pipeline`  | After Safety, before Intercept (pristine wire-fresh data)   | Most transaction events    |
| `post_pipeline` | After Transform + Macro fan-out, before Record + wire encode | Pass `phase="post_pipeline"` |
| `none`          | Lifecycle / observation only — no Pipeline coupling          | Lifecycle events (auto)    |

Pass `phase="post_pipeline"` to `register_hook()` to subscribe to the post-Transform phase. Lifecycle events accept no `phase=` argument; passing one is a load-time error.

## Configuration

Plugins are configured in the `plugins` array of the proxy config:

```json
{
  "plugins": [
    {
      "path": "plugins/add_auth_header.star",
      "on_error": "skip",
      "max_steps": 1000000,
      "vars": {
        "token": "Bearer eyJhbGciOiJIUzI1NiJ9.example"
      }
    }
  ]
}
```

| Field        | Type     | Required | Description                                                          |
|--------------|----------|----------|----------------------------------------------------------------------|
| `path`       | string   | yes      | Filesystem path to the `.star` script.                               |
| `name`       | string   | no       | Stable identifier; defaults to the basename of `path` without `.star`. |
| `on_error`   | string   | no       | `"skip"` (default) or `"abort"`. Controls hook-chain behavior on error. |
| `max_steps`  | uint64   | no       | Per-call Starlark step budget. Zero means use the default (1,000,000). |
| `vars`       | object   | no       | Primitive values exposed to the script as the frozen `config` dict.  |
| `redact_keys`| string[] | no       | `vars` keys to hide from the `plugin_introspect` MCP tool.            |

The legacy `protocol:` and `hooks:` fields no longer exist. A config that still carries them is rejected at startup with this error:

> `field hooks/protocol removed in RFC-001; use register_hook() in your script. See docs/rfc/plugin-migration.md`

There is no codec plugin section. The `codec_plugins` config key was removed at N9; codec functionality is no longer exposed to plugins.

## Loading and reloading

Plugins load once at proxy boot from `config.plugins`. To change the loaded set — add a plugin, remove one, edit a script — edit the config and restart the proxy. The proxy provides no runtime `reload`/`enable`/`disable` for plugins by design (RFC-001 §9.3 D2): plugin behavior is fixed at boot so resend, fuzz, and Macro fan-out are reproducible against a stable hook surface.

The `plugin_introspect` MCP tool (read-only) lists the currently loaded plugins and their registered `(protocol, event, phase)` tuples. See [plugin_introspect](../tools/plugin-introspect.md).

## Per-plugin scopes

Two state surfaces are available to hooks:

- **Per-plugin volatile state** — `state.get` / `state.set` / etc. Survives plugin reloads, lost on process restart. Isolated per plugin.
- **Per-plugin persistent store** — `store.get` / `store.set` / etc. Backed by SQLite when a database is configured. Survives process restarts.

Two scoped dicts attach to each hook invocation:

- `ctx.transaction_state` — per request/response transaction. For HTTP, scoped to one envelope (FlowID); for streaming protocols, scoped to the channel's lifetime (StreamID).
- `ctx.stream_state` — per stream. Shared across all events on the same `(ConnID, StreamID)`.

See [Writing plugins](writing-plugins.md) for the full module surface.

## Related pages

- [Writing plugins](writing-plugins.md) — Hook function signature, mutation API, sandbox modules.
- [Hook reference](hook-reference.md) — The 17-entry `(protocol, event)` surface and lifecycle events.
- [Data map reference](data-map-reference.md) — Per-protocol `msg` dict shape.
- [Examples](examples.md) — Ready-to-use plugin samples.
- [plugin_introspect tool](../tools/plugin-introspect.md) — List loaded plugins and registered hooks.
