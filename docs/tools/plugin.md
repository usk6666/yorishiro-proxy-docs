# plugin

Manage Starlark plugins at runtime. List registered plugins, reload them from disk, and enable or disable individual plugins.

## Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `action` | string | Yes | Action to perform: `list`, `reload`, `enable`, `disable` |
| `params` | object | No | Action-specific parameters (see below) |

## Actions

### list

Returns all registered plugins with metadata including name, path, enabled status, and registered hooks. No parameters required.

#### Response

| Field | Type | Description |
|-------|------|-------------|
| `plugins` | array | Plugin list with name, path, enabled, hooks |
| `count` | integer | Total number of plugins |

### reload

Reload a plugin by name from disk. If `name` is empty, all plugins are reloaded.

#### Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `name` | string | No | Plugin name to reload. If empty, reloads all plugins |

#### Response

| Field | Type | Description |
|-------|------|-------------|
| `reloaded` | string | Plugin name or `"all"` |
| `message` | string | Status message |

### enable

Enable a disabled plugin by name.

#### Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `name` | string | Yes | Plugin name to enable |

#### Response

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Plugin name |
| `enabled` | boolean | `true` |

### disable

Disable a plugin by name. Hooks from disabled plugins are skipped during dispatch.

#### Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `name` | string | Yes | Plugin name to disable |

#### Response

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Plugin name |
| `enabled` | boolean | `false` |

## Examples

### List all plugins

```json
// plugin
{
  "action": "list"
}
```

### Reload a specific plugin

```json
// plugin
{
  "action": "reload",
  "params": {"name": "my-auth-plugin"}
}
```

### Reload all plugins

```json
// plugin
{
  "action": "reload",
  "params": {}
}
```

### Enable a plugin

```json
// plugin
{
  "action": "enable",
  "params": {"name": "my-auth-plugin"}
}
```

### Disable a plugin

```json
// plugin
{
  "action": "disable",
  "params": {"name": "my-auth-plugin"}
}
```

## Related pages

- [Plugins overview](../plugins/overview.md) -- Plugin system overview
- [Writing plugins](../plugins/writing-plugins.md) -- Plugin development guide
- [Hook reference](../plugins/hook-reference.md) -- Available plugin hooks
