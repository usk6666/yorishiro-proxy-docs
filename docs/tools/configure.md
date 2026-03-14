# configure

Configure runtime proxy settings while the proxy is running. Supports incremental changes (merge) and full replacement of configuration sections. All sections are optional -- only specified sections are modified.

## Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `operation` | string | No | `"merge"` | Operation type: `"merge"` or `"replace"` |
| `upstream_proxy` | string | No | | Upstream proxy URL (empty string to disable, omit to keep current) |
| `capture_scope` | object | No | | Capture scope configuration |
| `tls_passthrough` | object | No | | TLS passthrough configuration |
| `intercept_rules` | object | No | | Intercept rules configuration |
| `intercept_queue` | object | No | | Intercept queue configuration |
| `auto_transform` | object | No | | Auto-transform rules configuration |
| `socks5_auth` | object | No | | SOCKS5 authentication configuration |
| `tls_fingerprint` | string | No | | TLS fingerprint profile (`chrome`, `firefox`, `safari`, `edge`, `random`, `none`) |
| `max_connections` | integer | No | | Maximum concurrent connections (1--100000) |
| `peek_timeout_ms` | integer | No | | Protocol detection timeout in ms (100--600000) |
| `request_timeout_ms` | integer | No | | HTTP request header read timeout in ms (100--600000) |
| `budget` | object | No | | Diagnostic session budget configuration |
| `client_cert` | object | No | | Global mTLS client certificate configuration |

### capture_scope

**Merge operation fields** (incremental add/remove):

| Field | Type | Description |
|-------|------|-------------|
| `add_includes` | array | Scope rules to add to the include list |
| `remove_includes` | array | Scope rules to remove from the include list |
| `add_excludes` | array | Scope rules to add to the exclude list |
| `remove_excludes` | array | Scope rules to remove from the exclude list |

**Replace operation fields** (full replacement):

| Field | Type | Description |
|-------|------|-------------|
| `includes` | array | Full replacement of include rules |
| `excludes` | array | Full replacement of exclude rules |

### tls_passthrough

**Merge operation fields:**

| Field | Type | Description |
|-------|------|-------------|
| `add` | string[] | Patterns to add |
| `remove` | string[] | Patterns to remove |

**Replace operation fields:**

| Field | Type | Description |
|-------|------|-------------|
| `patterns` | string[] | Full replacement of all passthrough patterns |

### intercept_rules

**Merge operation fields:**

| Field | Type | Description |
|-------|------|-------------|
| `add` | array | Intercept rules to add |
| `remove` | string[] | Rule IDs to remove |
| `enable` | string[] | Rule IDs to enable |
| `disable` | string[] | Rule IDs to disable |

**Replace operation fields:**

| Field | Type | Description |
|-------|------|-------------|
| `rules` | array | Full replacement of all intercept rules |

### intercept_queue

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `timeout_ms` | integer | `300000` | Timeout in ms for blocked requests (min 1000) |
| `timeout_behavior` | string | `"auto_release"` | Timeout behavior: `"auto_release"` or `"auto_drop"` |

### auto_transform

**Merge operation fields:**

| Field | Type | Description |
|-------|------|-------------|
| `add` | array | Transform rules to add |
| `remove` | string[] | Rule IDs to remove |
| `enable` | string[] | Rule IDs to enable |
| `disable` | string[] | Rule IDs to disable |

**Replace operation fields:**

| Field | Type | Description |
|-------|------|-------------|
| `rules` | array | Full replacement of all auto-transform rules |

Each auto-transform rule has:

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Unique rule identifier |
| `enabled` | boolean | Whether the rule is active |
| `priority` | integer | Execution order (lower values applied first) |
| `direction` | string | `"request"`, `"response"`, or `"both"` |
| `conditions` | object | Matching criteria (url_pattern, methods, header_match) |
| `action` | object | Transformation to apply |

Action types:

| Type | Fields | Description |
|------|--------|-------------|
| `"add_header"` | `header`, `value` | Add a header |
| `"set_header"` | `header`, `value` | Set/replace a header |
| `"remove_header"` | `header` | Remove a header |
| `"replace_body"` | `pattern`, `value` | Replace body content (regex) |

### socks5_auth

| Field | Type | Description |
|-------|------|-------------|
| `method` | string | `"none"` or `"password"` |
| `username` | string | Username for password auth |
| `password` | string | Password for password auth |
| `listener_name` | string | Listener name to configure (default: `"default"`) |

### budget

| Field | Type | Description |
|-------|------|-------------|
| `max_total_requests` | integer | Max total requests for the session (0 to clear) |
| `max_duration` | string | Max session duration, e.g. `"30m"` (`"0s"` to clear) |

### client_cert

| Field | Type | Description |
|-------|------|-------------|
| `cert_path` | string | PEM client certificate path (empty to remove) |
| `key_path` | string | PEM client private key path (empty to remove) |

## Response

| Field | Type | Description |
|-------|------|-------------|
| `status` | string | `"configured"` |
| `upstream_proxy` | string | Current upstream proxy URL (if changed) |
| `capture_scope` | object | Current scope state: `include_count`, `exclude_count` (if changed) |
| `tls_passthrough` | object | Current passthrough state: `total_patterns` (if changed) |
| `intercept_rules` | object | Current rules state: `total_rules`, `enabled_rules` (if changed) |
| `intercept_queue` | object | Current queue state: `timeout_ms`, `timeout_behavior`, `queued_items` (if changed) |
| `auto_transform` | object | Current transform state: `total_rules`, `enabled_rules` (if changed) |
| `socks5_auth` | object | Current auth state: `method` (if changed) |
| `tls_fingerprint` | string | Current TLS fingerprint profile (if changed) |
| `max_connections` | integer | Current max connections (if changed) |
| `peek_timeout_ms` | integer | Current peek timeout in ms (if changed) |
| `request_timeout_ms` | integer | Current request timeout in ms (if changed) |
| `budget` | object | Current budget state (if changed) |
| `client_cert` | object | Current client cert state (if changed) |

## Examples

### Add capture scope rules (merge)

```json
// configure
{
  "capture_scope": {
    "add_includes": [{"hostname": "api.target.com"}],
    "add_excludes": [{"hostname": "static.target.com"}]
  }
}
```

### Replace all capture scope rules

```json
// configure
{
  "operation": "replace",
  "capture_scope": {
    "includes": [{"hostname": "new-target.com"}],
    "excludes": []
  }
}
```

### Add TLS passthrough patterns

```json
// configure
{
  "tls_passthrough": {
    "add": ["*.googleapis.com", "accounts.google.com"]
  }
}
```

### Add intercept rules

```json
// configure
{
  "intercept_rules": {
    "add": [
      {
        "id": "json-api",
        "enabled": true,
        "direction": "request",
        "conditions": {
          "header_match": {"Content-Type": "application/json"}
        }
      }
    ]
  }
}
```

### Add auto-transform rules

```json
// configure
{
  "auto_transform": {
    "add": [
      {
        "id": "add-auth",
        "enabled": true,
        "priority": 10,
        "direction": "request",
        "conditions": {
          "url_pattern": "/api/.*"
        },
        "action": {
          "type": "set_header",
          "header": "Authorization",
          "value": "Bearer test-token"
        }
      }
    ]
  }
}
```

### Set upstream proxy

```json
// configure
{
  "upstream_proxy": "http://corporate-proxy:3128"
}
```

### Change TLS fingerprint

```json
// configure
{
  "tls_fingerprint": "firefox"
}
```

### Set diagnostic budget

```json
// configure
{
  "budget": {
    "max_total_requests": 1000,
    "max_duration": "30m"
  }
}
```

### Combined update

```json
// configure
{
  "capture_scope": {
    "add_includes": [{"hostname": "api.target.com", "url_prefix": "/v2/"}]
  },
  "tls_passthrough": {
    "add": ["pinned.service.com"]
  },
  "intercept_rules": {
    "add": [
      {
        "id": "json-api",
        "enabled": true,
        "direction": "request",
        "conditions": {
          "header_match": {"Content-Type": "application/json"}
        }
      }
    ]
  }
}
```

## Related pages

- [proxy_start](proxy-start.md) -- Start the proxy with initial configuration
- [Auto-transform](../features/auto-transform.md) -- Auto-transform feature guide
- [Intercept](../features/intercept.md) -- Intercept feature guide
- [Rate limits & budgets](../features/rate-limits.md) -- Rate limits and budget feature guide
