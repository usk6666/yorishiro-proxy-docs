# intercept

Act on intercepted requests or responses in the intercept queue. Intercepted items are held by the proxy until you decide to release, modify, or drop them.

Items have a `phase` field indicating when they were intercepted: `"request"` (before sending to the upstream server) or `"response"` (after receiving from the upstream server).

## Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `action` | string | Yes | Action to perform: `release`, `modify_and_forward`, `drop` |
| `params` | object | Yes | Action-specific parameters (see below) |

## Actions

### release

Forward the intercepted item as-is without any modifications.

#### Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `intercept_id` | string | Yes | ID of the intercepted request/response |

### modify_and_forward

Forward the intercepted item with mutations. Use request parameters for request-phase items and response parameters for response-phase items.

#### Request phase parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `intercept_id` | string | Yes | ID of the intercepted request |
| `override_method` | string | No | Override HTTP method |
| `override_url` | string | No | Override target URL |
| `override_headers` | object | No | Header overrides as key-value pairs |
| `add_headers` | object | No | Headers to add |
| `remove_headers` | string[] | No | Header names to remove |
| `override_body` | string | No | Override request body |

#### Response phase parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `intercept_id` | string | Yes | ID of the intercepted response |
| `override_status` | integer | No | Override HTTP status code |
| `override_response_headers` | object | No | Response header overrides |
| `add_response_headers` | object | No | Response headers to add |
| `remove_response_headers` | string[] | No | Response header names to remove |
| `override_response_body` | string | No | Override response body |

### drop

Discard the intercepted item. Returns a 502 Bad Gateway response to the client.

#### Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `intercept_id` | string | Yes | ID of the intercepted request/response |

## Response

All actions return:

| Field | Type | Description |
|-------|------|-------------|
| `intercept_id` | string | ID of the intercepted item |
| `action` | string | Action performed |
| `status` | string | Result status (`"released"`, `"forwarded"`, `"dropped"`) |

## Examples

### Release an intercepted request

```json
// intercept
{
  "action": "release",
  "params": {"intercept_id": "int-abc-123"}
}
```

### Modify and forward a request

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
