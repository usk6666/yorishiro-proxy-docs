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

## Raw bytes mode

By default, the intercept tool operates in **structured mode** -- it parses and serializes requests at the HTTP (L7) layer. This means modifications go through the standard HTTP library, which may normalize headers, adjust `Content-Length`, and apply transfer encoding.

**Raw bytes mode** bypasses this entirely. When you set `"mode": "raw"`, the proxy sends bytes directly on the wire without any HTTP library processing. This gives you full control over every byte of the request or response.

### Structured vs raw mode

| Aspect | Structured mode (default) | Raw mode |
|--------|--------------------------|----------|
| Layer | L7 (HTTP semantics) | L4 (raw bytes on the wire) |
| Modifications | `override_method`, `override_headers`, etc. | `raw_override_base64` (entire payload) |
| Normalization | HTTP library may adjust headers, Content-Length, encoding | None -- bytes are forwarded as-is |
| Auto-transform | Applied (e.g., gzip decompression for display) | Not applied |
| Use case | Standard request/response editing | Protocol-level anomaly testing |

### HTTP/1.x raw forwarding

In raw mode for HTTP/1.x, the proxy writes your raw bytes directly to a TCP (or TLS) connection to the upstream server, completely bypassing `net/http.Transport`. The raw response from upstream is read back and forwarded to the client as-is.

This enables testing scenarios where HTTP library normalization would otherwise mask the issue:

- **HTTP Request Smuggling** -- craft requests with conflicting `Content-Length` and `Transfer-Encoding` headers (CL/TE, TE/CL)
- **Header injection** -- send malformed headers that a standard HTTP library would reject or rewrite
- **Protocol downgrade testing** -- send intentionally non-conformant HTTP to observe server behavior

### HTTP/2 raw forwarding

For HTTP/2, raw mode operates on individual **frames** rather than the complete connection stream. When you provide raw bytes, the proxy:

1. Parses the 9-byte frame headers to locate stream ID fields
2. Rewrites stream IDs to match the upstream connection's allocated stream (connection-level frames with stream ID 0 are not rewritten)
3. Forwards the frames as-is, preserving all other bytes including flags, padding, and payload

This means you can craft custom HEADERS, DATA, or other frame types while the proxy handles the stream multiplexing automatically.

### Using raw mode

**Release with raw bytes** (forward original wire bytes without HTTP normalization):

```json
// intercept
{
  "action": "release",
  "params": {
    "intercept_id": "int-abc-123",
    "mode": "raw"
  }
}
```

**Modify and forward with raw bytes** (replace the entire payload):

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

The `raw_override_base64` value is the Base64 encoding of the complete raw bytes you want sent on the wire. For HTTP/1.x, this is the full request including the request line, headers, and body. For HTTP/2, this is one or more serialized frames.

!!! note
    Raw mode requires raw bytes to be available for the intercepted item. The response includes `raw_bytes_available: true` when this is the case.

## WebUI integration

The intercept queue is also accessible through the Web UI, where you can visually inspect and act on queued items. The WebUI provides a hex dump editor and text editor for raw bytes editing. See [WebUI Intercept](../webui/intercept.md) for details.

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
