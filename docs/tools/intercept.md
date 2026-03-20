# intercept

Act on intercepted requests or responses in the intercept queue. Intercepted items are held by the proxy until you decide to release, modify, or drop them.

Items have a `phase` field indicating when they were intercepted: `"request"` (before sending to the upstream server) or `"response"` (after receiving from the upstream server).

## Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `action` | string | Yes | Action to perform: `release`, `modify_and_forward`, `drop` |
| `params` | object | Yes | Action-specific parameters (see below) |

### Common parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `intercept_id` | string | Yes | ID of the intercepted request/response |
| `mode` | string | No | Forwarding mode: `"structured"` (default) or `"raw"`. See [raw bytes mode](#raw-bytes-mode) |

## Actions

### release

Forward the intercepted item as-is without any modifications.

In structured mode (default), the item is forwarded through the normal HTTP library pipeline. In raw mode (`"mode": "raw"`), the original raw bytes captured on the wire are forwarded directly, bypassing HTTP library serialization.

### modify_and_forward

Forward the intercepted item with mutations. The available parameters depend on the `mode`.

#### Structured mode parameters (default)

Use request parameters for request-phase items and response parameters for response-phase items.

**Request phase:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `override_method` | string | No | Override HTTP method |
| `override_url` | string | No | Override target URL |
| `override_headers` | object | No | Header overrides as key-value pairs |
| `add_headers` | object | No | Headers to add |
| `remove_headers` | string[] | No | Header names to remove |
| `override_body` | string | No | Override request body |

**Response phase:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `override_status` | integer | No | Override HTTP status code |
| `override_response_headers` | object | No | Response header overrides |
| `add_response_headers` | object | No | Response headers to add |
| `remove_response_headers` | string[] | No | Response header names to remove |
| `override_response_body` | string | No | Override response body |

#### Raw mode parameters

When `mode` is `"raw"`, all structured (L7) override fields are ignored. Instead, provide:

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `raw_override_base64` | string | Yes | Base64-encoded raw bytes that replace the entire request or response on the wire |

### drop

Discard the intercepted item. Returns a 502 Bad Gateway response to the client.

## Response

All actions return:

| Field | Type | Description |
|-------|------|-------------|
| `intercept_id` | string | ID of the intercepted item |
| `action` | string | Action performed |
| `status` | string | Result status (`"released"`, `"forwarded"`, `"forwarded_raw"`, `"dropped"`) |
| `phase` | string | `"request"`, `"response"`, or `"websocket_frame"` |
| `protocol` | string | `"http"` or `"websocket"` |

### Raw bytes fields

When raw bytes are available for the intercepted item, the response includes additional fields:

| Field | Type | Description |
|-------|------|-------------|
| `raw_bytes_available` | boolean | `true` when raw bytes are present |
| `raw_bytes_size` | integer | Size of the raw bytes in bytes |
| `raw_bytes_encoding` | string | Encoding of the `raw_bytes` field (`"text"` or `"base64"`) |
| `raw_bytes` | string | The raw captured bytes (subject to output filtering) |

## Raw bytes mode

The `mode` parameter controls how the proxy forwards intercepted items:

- **`"structured"`** (default) -- The proxy applies your modifications at the HTTP (L7) layer. The request or response is serialized through the standard HTTP library, which may normalize headers, adjust `Content-Length`, and apply transfer encoding.
- **`"raw"`** -- The proxy sends bytes directly on the wire, bypassing HTTP library serialization entirely. This gives you full control over every byte of the request or response.

Raw mode is available for both `release` and `modify_and_forward`:

- **`release` + raw mode** -- Forwards the original raw bytes as captured on the wire, without HTTP library normalization.
- **`modify_and_forward` + raw mode** -- Forwards your `raw_override_base64` bytes as-is. Requires `raw_override_base64`.

## Examples

### Release an intercepted request

```json
// intercept
{
  "action": "release",
  "params": {"intercept_id": "int-abc-123"}
}
```

### Modify and forward a request (structured mode)

```json
// intercept
{
  "action": "modify_and_forward",
  "params": {
    "intercept_id": "int-abc-123",
    "override_method": "POST",
    "override_headers": {"Authorization": "Bearer injected-token"},
    "override_body": "{\"role\":\"admin\"}"
  }
}
```

### Modify and forward a response

```json
// intercept
{
  "action": "modify_and_forward",
  "params": {
    "intercept_id": "int-resp-456",
    "override_status": 200,
    "override_response_headers": {"X-Custom": "modified"},
    "override_response_body": "{\"authorized\": true}"
  }
}
```

### Modify and forward with raw bytes

Send a hand-crafted HTTP request with intentional CL/TE ambiguity for smuggling testing:

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

### Release with raw bytes (bypass HTTP normalization)

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

### Drop an intercepted request

```json
// intercept
{
  "action": "drop",
  "params": {"intercept_id": "int-abc-123"}
}
```

## Related pages

- [Intercept](../features/intercept.md) -- Intercept feature guide
- [configure](configure.md) -- Configure intercept rules and queue settings
- [query](query.md) -- Query the intercept queue with `{"resource": "intercept_queue"}`
