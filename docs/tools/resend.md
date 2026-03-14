# resend

Resend and replay recorded proxy requests with optional mutations, or compare two flows structurally. The original flow is preserved; a new flow is created for each resend.

## Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `action` | string | Yes | Action to perform: `resend`, `resend_raw`, `tcp_replay`, `compare` |
| `params` | object | Yes | Action-specific parameters (see below) |

## Actions

### resend

Resend a recorded HTTP/HTTP2/WebSocket request with optional mutations. Records the result as a new flow.

**Multi-protocol support:**

- **HTTP/1.x and HTTPS** -- standard HTTP resend with full mutation support.
- **HTTP/2** -- resends the request using HTTP/1.1 fallback. All mutation options apply.
- **WebSocket** -- requires `message_sequence` to identify the message. Supports `override_body`, `override_body_base64`, `target_addr`, and `use_tls`.

#### Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `flow_id` | string | Yes | | ID of the flow to resend |
| `message_sequence` | integer | No | | Message sequence for WebSocket/streaming resend |
| `override_method` | string | No | | Override HTTP method |
| `override_url` | string | No | | Override target URL (must include scheme and host) |
| `override_headers` | array/object | No | | Header overrides (replaces matching headers) |
| `add_headers` | array/object | No | | Headers to add (appended to existing values) |
| `remove_headers` | string[] | No | | Header names to remove |
| `override_body` | string | No | | Override request body (text) |
| `override_body_base64` | string | No | | Override request body (Base64, mutually exclusive with `override_body`) |
| `body_patches` | array | No | | Partial body modifications |
| `override_host` | string | No | | TCP connection target as `host:port` |
| `follow_redirects` | boolean | No | `false` | Follow HTTP redirects automatically |
| `timeout_ms` | integer | No | `30000` | Request timeout in ms |
| `dry_run` | boolean | No | `false` | Preview the modified request without sending |
| `tag` | string | No | | Tag to attach to the result flow |
| `hooks` | object | No | | Pre/post hooks for macro integration |

**Header mutation order:** `remove_headers` -> `override_headers` -> `add_headers`.

**Body mutation priority:** `override_body`/`override_body_base64` (full replace) > `body_patches` (partial) > original body.

#### body_patches

Each patch is one of:

- **JSON path patch**: `{"json_path": "$.key", "value": "new"}`
- **Regex patch**: `{"regex": "pattern", "replace": "replacement"}`

An optional `"encoding"` array applies codec transformations as a pipeline (max 10 codecs). Available codecs: `base64`, `base64url`, `url_encode_query`, `url_encode_path`, `url_encode_full`, `double_url_encode`, `hex`, `html_entity`, `html_escape`, `unicode_escape`, `md5`, `sha256`, `lower`, `upper`.

#### Response

| Field | Type | Description |
|-------|------|-------------|
| `new_flow_id` | string | ID of the new flow |
| `status_code` | integer | HTTP status code |
| `response_headers` | object | Response headers |
| `response_body` | string | Response body |
| `response_body_encoding` | string | `"text"` or `"base64"` |
| `duration_ms` | integer | Request duration in ms |
| `tag` | string | Tag (if provided) |

In dry-run mode: `dry_run`, `request_preview` (method, url, headers, body, body_encoding).

### resend_raw

Resend raw bytes from a recorded flow over TCP/TLS. Useful for testing HTTP smuggling or protocol-level issues.

#### Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `flow_id` | string | Yes | | ID of the flow to resend |
| `target_addr` | string | No | original target | Target address as `host:port` |
| `use_tls` | boolean | No | original protocol | Force TLS on/off |
| `override_raw_base64` | string | No | | Full replacement of raw bytes (Base64) |
| `patches` | array | No | | Byte-level patches (offset overwrite, find-replace) |
| `timeout_ms` | integer | No | `30000` | Request timeout in ms |
| `dry_run` | boolean | No | `false` | Preview raw bytes without sending |
| `tag` | string | No | | Tag to attach to the result flow |

#### Response

| Field | Type | Description |
|-------|------|-------------|
| `new_flow_id` | string | ID of the new flow |
| `response_data` | string | Response data (Base64) |
| `response_size` | integer | Response size in bytes |
| `duration_ms` | integer | Request duration in ms |
| `tag` | string | Tag (if provided) |

### tcp_replay

Replay a Raw TCP flow by sending all recorded send messages sequentially to the target.

#### Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `flow_id` | string | Yes | | ID of the TCP flow to replay |
| `target_addr` | string | No | original server address | Target address as `host:port` |
| `use_tls` | boolean | No | `false` | Use TLS for the connection |
| `timeout_ms` | integer | No | `30000` | Connection timeout in ms |
| `tag` | string | No | | Tag to attach to the result flow |

#### Response

| Field | Type | Description |
|-------|------|-------------|
| `new_flow_id` | string | ID of the new flow |
| `messages_sent` | integer | Number of messages sent |
| `messages_received` | integer | Number of messages received |
| `total_bytes_sent` | integer | Total bytes sent |
| `total_bytes_received` | integer | Total bytes received |
| `duration_ms` | integer | Total duration in ms |
| `tag` | string | Tag (if provided) |

### compare

Compare two flows structurally. Returns a summary of differences in status code, headers, body length, and timing.

#### Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `flow_id_a` | string | Yes | ID of the first flow |
| `flow_id_b` | string | Yes | ID of the second flow |

#### Response

| Field | Type | Description |
|-------|------|-------------|
| `status_code` | object | `a`, `b`, `changed` |
| `body_length` | object | `a`, `b`, `delta` |
| `headers_added` | string[] | Headers present in B but not A |
| `headers_removed` | string[] | Headers present in A but not B |
| `headers_changed` | string[] | Headers with different values |
| `timing_ms` | object | `a`, `b`, `delta` |
| `body` | object | `content_type`, `identical`, `json_diff` (for JSON responses) |

For JSON responses, `json_diff` includes: `keys_added`, `keys_removed`, `keys_changed`.

## Hooks

The `resend` action supports optional hooks that execute macros before sending and after receiving.

### pre_send

| Field | Type | Description |
|-------|------|-------------|
| `macro` | string | Name of the stored macro to execute |
| `vars` | object | Runtime variable overrides |
| `run_interval` | string | `"always"` (default), `"once"`, `"every_n"`, `"on_error"` |
| `n` | integer | Interval count for `"every_n"` |

### post_receive

| Field | Type | Description |
|-------|------|-------------|
| `macro` | string | Name of the stored macro to execute |
| `vars` | object | Runtime variable overrides |
| `run_interval` | string | `"always"` (default), `"on_status"`, `"on_match"` |
| `status_codes` | integer[] | Status codes for `"on_status"` |
| `match_pattern` | string | Regex pattern for `"on_match"` |
| `pass_response` | boolean | Pass `__response_status` and `__response_body` as vars |

## Examples

### Resend with method override

```json
// resend
{
  "action": "resend",
  "params": {
    "flow_id": "abc-123",
    "override_method": "PUT",
    "override_headers": [{"key": "Content-Type", "value": "application/json"}]
  }
}
```

### Resend with body patches

```json
// resend
{
  "action": "resend",
  "params": {
    "flow_id": "abc-123",
    "body_patches": [
      {"json_path": "$.user.role", "value": "admin"},
      {"regex": "csrf_token=[^&]+", "replace": "csrf_token=injected", "encoding": ["url_encode_query"]}
    ]
  }
}
```

### Dry-run preview

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

### Raw TCP resend

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

### Replay Raw TCP flow

```json
// resend
{
  "action": "tcp_replay",
  "params": {
    "flow_id": "tcp-flow-456",
    "target_addr": "db.target.com:3306",
    "tag": "tcp-replay"
  }
}
```

### Compare two flows

```json
// resend
{
  "action": "compare",
  "params": {
    "flow_id_a": "original-flow-123",
    "flow_id_b": "mutated-flow-456"
  }
}
```

### Resend WebSocket message

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

## Related pages

- [Resender](../features/resender.md) -- Resender feature guide
- [Comparer](../features/comparer.md) -- Comparer feature guide
- [query](query.md) -- Query flow data and results
- [fuzz](fuzz.md) -- Fuzz testing campaigns
- [macro](macro.md) -- Multi-step workflows with hooks
