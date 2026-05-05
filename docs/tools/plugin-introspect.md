# plugin_introspect

Read-only introspect of the loaded `pluginv2` engine. Returns the list of loaded plugins together with their `register_hook` registrations and (redacted) `PluginConfig.Vars` map.

This is the **only** plugin MCP tool. There is no `reload`, `enable`, or `disable` action by design -- plugin lifecycle is owned by the engine's static load on startup. Re-load the proxy to pick up plugin changes.

## Parameters

This tool accepts no parameters. Pass an empty object:

```json
// plugin_introspect
{}
```

## Response

| Field | Type | Description |
|-------|------|-------------|
| `plugins` | array | One entry per loaded plugin (see below). Empty array when the `pluginv2` engine is not configured |

Each `plugins[]` entry:

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Plugin's stable identifier |
| `path` | string | Filesystem location of the plugin script |
| `enabled` | boolean | `true` for every successfully loaded plugin (the engine considers it live) |
| `registrations` | array | Each `register_hook` call the plugin made, in script order (see below) |
| `vars` | object | `PluginConfig.Vars` with `redact_keys` applied. Redacted values become the literal string `"<redacted>"`; large values are truncated by the engine |

Each `registrations[]` entry:

| Field | Type | Description |
|-------|------|-------------|
| `protocol` | string | Protocol family the hook subscribes to (e.g. `"http"`, `"ws"`, `"grpc"`, `"raw"`) |
| `event` | string | Event name (e.g. `"on_send"`, `"on_receive"`, `"on_data"`) |
| `phase` | string | Pipeline phase (e.g. `"pre"`, `"post"`) |

## Examples

### List loaded plugins

```json
// plugin_introspect
{}
```

Response:

```json
{
  "plugins": [
    {
      "name": "csrf-signer",
      "path": "/etc/yorishiro/plugins/csrf_signer.star",
      "enabled": true,
      "registrations": [
        {"protocol": "http", "event": "on_send", "phase": "post"}
      ],
      "vars": {
        "secret": "<redacted>",
        "domain": "example.com"
      }
    }
  ]
}
```

## Related pages

- [Plugins overview](../plugins/overview.md) -- Plugin system overview
- [Writing plugins](../plugins/writing-plugins.md) -- Plugin development guide
- [Hook reference](../plugins/hook-reference.md) -- Available plugin hooks
