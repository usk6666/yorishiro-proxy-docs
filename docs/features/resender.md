# Resender

The resender lets you replay recorded proxy requests with optional mutations. You can override the method, URL, headers, and body, apply JSON patches or regex replacements, preview changes with dry-run, and resend raw bytes for protocol-level testing.

## Actions overview

| Action | Description |
|--------|-------------|
| `resend` | Resend an HTTP/HTTP2/WebSocket request with optional mutations |
| `resend_raw` | Resend raw bytes over TCP/TLS with byte-level patches |
| `tcp_replay` | Replay a Raw TCP flow by sending all send messages sequentially |
| `compare` | Compare two flows structurally (see [Comparer](comparer.md)) |

## Resending with overrides

The `resend` action replays a recorded flow and lets you override any part of the request. The original flow is not modified; a new flow is created with the result.

### Method override

Change the HTTP method of the request:

```json
// resend
{
  "action": "resend",
  "params": {
    "flow_id": "abc-123",
    "override_method": "PUT"
  }
}
```

### URL override

Redirect the request to a different URL. The URL must include both the scheme and host:

```json
// resend
{
  "action": "resend",
  "params": {
    "flow_id": "abc-123",
    "override_url": "https://staging.target.com/api/v2/users"
  }
}
```

### Header mutations

Headers are applied in a specific order: `remove_headers` first, then `override_headers`, then `add_headers`.

- **`override_headers`** replaces all values for a given header key
- **`add_headers`** appends values to existing headers (supports multi-value headers)
- **`remove_headers`** removes headers by name

```json
// resend
{
  "action": "resend",
  "params": {
    "flow_id": "abc-123",
    "override_headers": [
      {"key": "Content-Type", "value": "application/json"}
    ],
    "add_headers": [
      {"key": "X-Custom", "value": "value1"},
      {"key": "X-Custom", "value": "value2"}
    ],
    "remove_headers": ["X-Deprecated-Header"]
  }
}
```

### Body override

Replace the entire request body with text or Base64-encoded binary data:

```json
// resend
{
  "action": "resend",
  "params": {
    "flow_id": "abc-123",
    "override_body": "{\"username\": \"admin\", \"role\": \"superuser\"}"
  }
}
```

For binary data, use `override_body_base64`:

```json
// resend
{
  "action": "resend",
  "params": {
    "flow_id": "abc-123",
    "override_body_base64": "SGVsbG8gV29ybGQ="
  }
}
```

!!! note "Mutual exclusivity"
    `override_body` and `override_body_base64` are mutually exclusive. If either is set, `body_patches` are ignored.

## JSON patches

For surgical modifications to JSON bodies, use `body_patches` with `json_path`. The path uses simplified dot notation (`$.key1.key2`):

```json
// resend
{
  "action": "resend",
  "params": {
    "flow_id": "abc-123",
    "body_patches": [
      {"json_path": "$.user.role", "value": "admin"},
      {"json_path": "$.user.active", "value": true},
      {"json_path": "$.settings.limit", "value": 9999}
    ]
  }
}
```

!!! info "JSON path limitations"
    The JSON path parser supports dot notation only (`$.key1.key2.key3`). Array index notation is not supported. The `$` prefix is optional.

## Regex body patches

For text-based modifications, use `body_patches` with `regex` and `replace`. Capture group references (`$1`, `$2`) are supported:

```json
// resend
{
  "action": "resend",
  "params": {
    "flow_id": "abc-123",
    "body_patches": [
      {"regex": "csrf_token=[^&]+", "replace": "csrf_token=injected_value"},
      {"regex": "(role=)(user)", "replace": "${1}admin"}
    ]
  }
}
```

### Encoding chains

Both JSON and regex patches support an `encoding` array that applies codec transformations to the patch value before applying. Codecs are applied in pipeline order:

```json
// resend
{
  "action": "resend",
  "params": {
    "flow_id": "abc-123",
    "body_patches": [
      {
        "json_path": "$.token",
        "value": "<script>alert(1)</script>",
        "encoding": ["url_encode_query", "base64"]
      }
    ]
  }
}
```

Available codecs: `base64`, `base64url`, `url_encode_query`, `url_encode_path`, `url_encode_full`, `double_url_encode`, `hex`, `html_entity`, `html_escape`, `unicode_escape`, `md5`, `sha256`, `lower`, `upper`.

Maximum chain length is 10 codecs.

## Body mutation priority

The body is resolved in this priority order:

1. `override_body` or `override_body_base64` (full replacement, highest priority)
2. `body_patches` (partial modifications)
3. Original body from the recorded flow

## Dry-run preview

Use `dry_run: true` to preview the modified request without sending it. This returns the final method, URL, headers, and body after all mutations are applied:

```json
// resend
{
  "action": "resend",
  "params": {
    "flow_id": "abc-123",
    "override_method": "POST",
    "body_patches": [{"json_path": "$.user.role", "value": "admin"}],
    "dry_run": true
  }
}
```

The response includes a `request_preview` object with `method`, `url`, `headers`, `body`, and `body_encoding`.

## Raw HTTP resend

The `resend_raw` action sends the raw bytes from a recorded flow over a TCP/TLS connection. This is useful for testing HTTP request smuggling, malformed requests, or protocol-level vulnerabilities.

```json
// resend
{
  "action": "resend_raw",
  "params": {
    "flow_id": "abc-123",
    "target_addr": "target.com:443",
    "use_tls": true
  }
}
```

### Raw byte patches

You can modify raw bytes with three patch modes:

**Offset overwrite** -- overwrite bytes at a specific position:

```json
// resend
{
  "action": "resend_raw",
  "params": {
    "flow_id": "abc-123",
    "patches": [
      {"offset": 0, "data_base64": "R0VU"}
    ]
  }
}
```

**Binary find and replace**:

```json
// resend
{
  "action": "resend_raw",
  "params": {
    "flow_id": "abc-123",
    "patches": [
      {"find_base64": "SFRUUC8xLjE=", "replace_base64": "SFRUUC8xLjA="}
    ]
  }
}
```

**Text find and replace**:

```json
// resend
{
  "action": "resend_raw",
  "params": {
    "flow_id": "abc-123",
    "patches": [
      {"find_text": "Host: original.com", "replace_text": "Host: target.com"}
    ]
  }
}
```

You can also fully replace the raw bytes with `override_raw_base64`.

## TCP replay

The `tcp_replay` action replays a Raw TCP flow by sending all recorded send messages sequentially to the target. This records the entire exchange as a new TCP flow:

```json
// resend
{
  "action": "tcp_replay",
  "params": {
    "flow_id": "tcp-flow-456",
    "target_addr": "db.target.com:3306",
    "tag": "tcp-replay-test"
  }
}
```

## Multi-protocol support

The resender handles different protocols automatically:

| Protocol | Behavior |
|----------|----------|
| HTTP/1.x | Standard HTTP resend with full mutation support |
| HTTPS | Same as HTTP/1.x, with TLS |
| HTTP/2 | Resends using HTTP/1.1 fallback; all mutation options apply |
| WebSocket | Requires `message_sequence` to identify the message; sends as raw TCP frame |
| Raw TCP | Use `tcp_replay` action |

### WebSocket resend

For WebSocket flows, specify the `message_sequence` to identify which message to resend:

```json
// resend
{
  "action": "resend",
  "params": {
    "flow_id": "ws-flow-123",
    "message_sequence": 2,
    "target_addr": "ws.target.com:443",
    "use_tls": true
  }
}
```

## Additional options

| Parameter | Type | Description |
|-----------|------|-------------|
| `override_host` | string | TCP connection target as `host:port`, independent of the URL host |
| `follow_redirects` | boolean | Follow HTTP redirects automatically (default: `false`) |
| `timeout_ms` | integer | Request timeout in milliseconds (default: `30000`) |
| `tag` | string | Tag to attach to the result flow for identification |

## Hooks integration

The resender supports pre/post hooks that execute macros before sending and after receiving. See [Macros](macros.md) for details on hook configuration.

```json
// resend
{
  "action": "resend",
  "params": {
    "flow_id": "abc-123",
    "hooks": {
      "pre_send": {
        "macro": "refresh-auth",
        "run_interval": "always"
      },
      "post_receive": {
        "macro": "log-response",
        "run_interval": "on_status",
        "status_codes": [401, 403]
      }
    }
  }
}
```

## Practical use cases

### Authentication testing

Resend a login request with different credentials to test for weak password policies:

```json
// resend
{
  "action": "resend",
  "params": {
    "flow_id": "login-flow-id",
    "body_patches": [
      {"json_path": "$.password", "value": "admin123"}
    ]
  }
}
```

### Parameter tampering

Modify a recorded API request to test for authorization bypass:

```json
// resend
{
  "action": "resend",
  "params": {
    "flow_id": "api-flow-id",
    "body_patches": [
      {"json_path": "$.user_id", "value": 1},
      {"json_path": "$.role", "value": "admin"}
    ],
    "tag": "idor-test"
  }
}
```

### HTTP request smuggling

Use raw resend with text patches to test for request smuggling:

```json
// resend
{
  "action": "resend_raw",
  "params": {
    "flow_id": "abc-123",
    "patches": [
      {"find_text": "Content-Length: 42", "replace_text": "Transfer-Encoding: chunked"}
    ],
    "target_addr": "target.com:80"
  }
}
```

## Related pages

- [Resend tool reference](../tools/resend.md) -- MCP tool parameter reference
- [Fuzzer](fuzzer.md) -- Automated fuzzing with payload injection
- [Comparer](comparer.md) -- Compare responses from resend operations
- [Macros](macros.md) -- Hook integration for multi-step workflows
