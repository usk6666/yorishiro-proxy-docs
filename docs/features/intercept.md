# Intercept

The intercept feature lets you pause HTTP requests and responses in transit, inspect them, and decide whether to release, modify, or drop them. This gives you real-time control over traffic flowing through the proxy.

## How interception works

1. Define intercept rules that match specific requests or responses
2. When a matching request or response passes through the proxy, it is held in the intercept queue
3. The AI agent (or WebUI user) reviews the queued item
4. The item is released as-is, modified and forwarded, or dropped

## Configuring intercept rules

Intercept rules define which requests and responses are captured. You configure rules using the `configure` tool:

```json
// configure
{
  "intercept_rules": {
    "add": [
      {
        "id": "api-requests",
        "enabled": true,
        "direction": "request",
        "conditions": {
          "host_pattern": "api\\.target\\.com",
          "path_pattern": "/api/.*",
          "methods": ["POST", "PUT", "DELETE"]
        }
      }
    ]
  }
}
```

### Rule fields

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Unique rule identifier |
| `enabled` | boolean | Whether the rule is active |
| `direction` | string | `"request"`, `"response"`, or `"both"` |
| `conditions` | object | Matching criteria |

### Condition fields

| Field | Type | Description |
|-------|------|-------------|
| `host_pattern` | string | Regex matched against the hostname (port excluded) |
| `path_pattern` | string | Regex matched against the URL path |
| `methods` | array | HTTP method whitelist (case-insensitive) |
| `header_match` | object | Header name to regex mapping (AND logic) |

All specified conditions must match for a rule to trigger (AND logic).

### Direction

- **`request`** -- intercepts the request before it reaches the upstream server
- **`response`** -- intercepts the response before it reaches the client
- **`both`** -- intercepts both the request and the response

Intercepted items include a `phase` field (`"request"` or `"response"`) so you know which direction was captured.

## Managing the queue

### Viewing queued items

Use the `query` tool to list items waiting in the intercept queue:

```json
// query
{
  "action": "intercept_queue"
}
```

Each queued item has an `intercept_id` that you use for subsequent actions.

### Release

Forward the intercepted item as-is, without modifications:

```json
// intercept
{
  "action": "release",
  "params": {
    "intercept_id": "int-abc-123"
  }
}
```

### Modify and forward

Modify the intercepted item before forwarding. The available mutation parameters depend on the phase.

**Request phase mutations:**

```json
// intercept
{
  "action": "modify_and_forward",
  "params": {
    "intercept_id": "int-abc-123",
    "override_method": "POST",
    "override_url": "https://api.target.com/v2/users",
    "override_headers": {"Authorization": "Bearer injected-token"},
    "add_headers": {"X-Custom": "value"},
    "remove_headers": ["X-Debug"],
    "override_body": "{\"role\":\"admin\"}"
  }
}
```

**Response phase mutations:**

```json
// intercept
{
  "action": "modify_and_forward",
  "params": {
    "intercept_id": "int-resp-456",
    "override_status": 200,
    "override_response_headers": {"Content-Type": "application/json"},
    "add_response_headers": {"X-Modified": "true"},
    "remove_response_headers": ["X-Server-Info"],
    "override_response_body": "{\"success\": true}"
  }
}
```

### Drop

Discard the intercepted item. For requests, this returns a 502 Bad Gateway response to the client:

```json
// intercept
{
  "action": "drop",
  "params": {
    "intercept_id": "int-abc-123"
  }
}
```

## Request phase parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `override_method` | string | Override the HTTP method |
| `override_url` | string | Override the target URL |
| `override_headers` | object | Header overrides (key-value pairs) |
| `add_headers` | object | Headers to add |
| `remove_headers` | array | Header names to remove |
| `override_body` | string | Override the request body |

## Response phase parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `override_status` | integer | Override the HTTP status code |
| `override_response_headers` | object | Response header overrides |
| `add_response_headers` | object | Response headers to add |
| `remove_response_headers` | array | Response header names to remove |
| `override_response_body` | string | Override the response body |

## Queue configuration

Configure the intercept queue behavior using the `configure` tool:

```json
// configure
{
  "intercept_queue": {
    "timeout_ms": 300000,
    "timeout_behavior": "auto_release"
  }
}
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `timeout_ms` | integer | `300000` (5 min) | Timeout for blocked requests (minimum 1000 ms) |
| `timeout_behavior` | string | `auto_release` | What happens on timeout: `auto_release` or `auto_drop` |

When a blocked request times out:

- **`auto_release`** -- the request is forwarded as-is (default)
- **`auto_drop`** -- the request is dropped with a 502 response

## Managing rules

### Enable and disable rules

```json
// configure
{
  "intercept_rules": {
    "disable": ["api-requests"],
    "enable": ["other-rule"]
  }
}
```

### Remove rules

```json
// configure
{
  "intercept_rules": {
    "remove": ["api-requests"]
  }
}
```

### Replace all rules

```json
// configure
{
  "operation": "replace",
  "intercept_rules": {
    "rules": [
      {
        "id": "new-rule",
        "enabled": true,
        "direction": "both",
        "conditions": {
          "path_pattern": "/api/.*"
        }
      }
    ]
  }
}
```

## WebUI integration

The intercept queue is also accessible through the Web UI, where you can visually inspect and act on queued items. See [WebUI Intercept](../webui/intercept.md) for details.

## Practical use cases

### Inspect login requests

Intercept login requests to test different credential combinations in real time:

```json
// configure
{
  "intercept_rules": {
    "add": [
      {
        "id": "login-intercept",
        "enabled": true,
        "direction": "request",
        "conditions": {
          "path_pattern": "/login",
          "methods": ["POST"]
        }
      }
    ]
  }
}
```

### Modify API responses

Intercept responses to test how the client handles different server behaviors:

```json
// configure
{
  "intercept_rules": {
    "add": [
      {
        "id": "api-responses",
        "enabled": true,
        "direction": "response",
        "conditions": {
          "host_pattern": "api\\.target\\.com",
          "header_match": {"Content-Type": "application/json"}
        }
      }
    ]
  }
}
```

### Intercept by header

Intercept only requests with specific headers:

```json
// configure
{
  "intercept_rules": {
    "add": [
      {
        "id": "auth-requests",
        "enabled": true,
        "direction": "request",
        "conditions": {
          "header_match": {"Authorization": "Bearer.*"}
        }
      }
    ]
  }
}
```

## Related pages

- [Intercept tool reference](../tools/intercept.md) -- MCP tool parameter reference
- [Configure tool reference](../tools/configure.md) -- Rule configuration reference
- [WebUI Intercept](../webui/intercept.md) -- WebUI intercept guide
