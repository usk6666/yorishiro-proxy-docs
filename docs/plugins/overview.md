# Plugin system overview

yorishiro-proxy supports user-defined plugins written in [Starlark](https://github.com/google/starlark-go), a Python-like language designed for configuration and extension. Plugins hook into the proxy pipeline to inspect, modify, or block traffic in real time.

## Design principles

### Fail-open

If a plugin errors at runtime, the default behavior (`on_error: "skip"`) skips that plugin and continues processing. Traffic is never silently blocked by a broken plugin unless you explicitly set `on_error: "abort"`.

### Sandboxed execution

Starlark scripts run with a step limit (default: 1,000,000 steps) to prevent infinite loops. Scripts cannot access the filesystem, network, or other system resources.

### Registration order

Plugins are called in the order they are registered. A plugin's modifications are visible to subsequent plugins in the chain.

## Two types of plugins

yorishiro-proxy provides two distinct plugin types:

| Aspect | Hook plugins | Codec plugins |
|--------|-------------|---------------|
| Purpose | Inspect, modify, or block traffic in the proxy pipeline | Define custom string encode/decode transformations |
| Configuration | `plugins` section | `codec_plugins` section |
| Starlark API | Hook functions + `action` module | `name` variable + `encode`/`decode` functions |
| State | `state` and `store` modules for cross-call data | Stateless (pure functions) |
| Execution limit | Configurable `max_steps` per hook call | Default 1,000,000 steps |

**Hook plugins** run at specific points in the proxy pipeline (e.g., when a request arrives from the client, before it is sent to the server). They receive protocol-specific data and return an action that controls how the proxy proceeds.

**Codec plugins** define encoding and decoding transformations that integrate with the built-in codec registry. Once loaded, they can be used in fuzzer encoding chains, macro templates, and from other Starlark plugins via the `codec` module.

## Configuration

### Hook plugins

Hook plugins are configured in the `plugins` section of the config file:

```json
{
  "plugins": [
    {
      "path": "examples/plugins/add_auth_header.star",
      "protocol": "http",
      "hooks": ["on_before_send_to_server"],
      "on_error": "skip",
      "max_steps": 1000000,
      "vars": {
        "token": "Bearer eyJhbGciOiJIUzI1NiJ9.example"
      }
    }
  ]
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `path` | string | Yes | Filesystem path to the `.star` script |
| `protocol` | string | Yes | Protocol this plugin applies to: `http`, `https`, `h2`, `grpc`, `websocket`, `tcp`, `socks5` |
| `hooks` | string[] | Yes | List of hook function names to register |
| `on_error` | string | No | Error behavior: `"skip"` (default) or `"abort"` |
| `max_steps` | uint64 | No | Maximum Starlark execution steps per hook call (default: 1,000,000) |
| `vars` | object | No | Key-value pairs injected as a frozen `config` dict in the Starlark runtime |

### Codec plugins

Codec plugins are configured in the `codec_plugins` section:

```json
{
  "codec_plugins": [
    {"path": "codecs/sql_escape.star"},
    {"path": "codecs/"}
  ]
}
```

Each entry specifies a path to a `.star` file or a directory containing `.star` files. See [Codec plugins](codec-plugins.md) for details.

## Runtime management with the MCP plugin tool

The `plugin` MCP tool provides runtime management of loaded plugins.

### List plugins

```json
// plugin
{"action": "list"}
```

Response:

```json
{
  "plugins": [
    {
      "name": "add_auth_header",
      "path": "/path/to/add_auth_header.star",
      "protocol": "http",
      "hooks": ["on_before_send_to_server"],
      "enabled": true
    }
  ],
  "count": 1
}
```

### Reload plugins

Reload a single plugin or all plugins from disk. The enabled/disabled state is preserved across reloads.

```json
// plugin
{"action": "reload", "params": {"name": "add_auth_header"}}
```

```json
// plugin
{"action": "reload"}
```

### Enable / disable plugins

```json
// plugin
{"action": "enable", "params": {"name": "add_auth_header"}}
```

```json
// plugin
{"action": "disable", "params": {"name": "ws_filter"}}
```

Disabled plugins' hooks are skipped during dispatch.

## Related pages

- [Writing plugins](writing-plugins.md) -- How to write Starlark hook plugins
- [Hook reference](hook-reference.md) -- All hook functions and their call timing
- [Data map reference](data-map-reference.md) -- Protocol-specific data keys
- [Codec plugins](codec-plugins.md) -- Custom encode/decode transformations
- [Examples](examples.md) -- Ready-to-use plugin samples
- [plugin tool](../tools/plugin.md) -- MCP tool reference
