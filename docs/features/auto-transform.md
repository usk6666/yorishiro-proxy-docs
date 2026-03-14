# Auto-transform rules

Auto-transform rules automatically modify matching requests and responses as they pass through the proxy. You can add, set, or remove headers and replace body content based on URL patterns, methods, and header conditions.

## How auto-transform works

When a request or response passes through the proxy:

1. Each enabled auto-transform rule is evaluated in priority order (lower values first)
2. If all conditions match, the rule's action is applied
3. Multiple rules can apply to the same request or response

## Rule definition

Each auto-transform rule has the following fields:

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Unique rule identifier |
| `enabled` | boolean | Whether the rule is active |
| `priority` | integer | Execution order (lower values applied first) |
| `direction` | string | `"request"`, `"response"`, or `"both"` |
| `conditions` | object | Matching criteria |
| `action` | object | Transformation to apply |

## Matching conditions

| Field | Type | Description |
|-------|------|-------------|
| `url_pattern` | string | Regex matched against the full request URL |
| `methods` | array | HTTP method whitelist (case-insensitive) |
| `header_match` | object | Header name to regex mapping (AND logic) |

All specified conditions must match (AND logic). An empty conditions object matches everything.

## Action types

### Set a header

Replace a header value (or add it if not present):

```json
// configure
{
  "auto_transform": {
    "add": [
      {
        "id": "inject-auth",
        "enabled": true,
        "priority": 10,
        "direction": "request",
        "conditions": {
          "url_pattern": "/api/.*"
        },
        "action": {
          "type": "set_header",
          "header": "Authorization",
          "value": "Bearer test-token-123"
        }
      }
    ]
  }
}
```

### Add a header

Add a header value (appends to existing values):

```json
// configure
{
  "auto_transform": {
    "add": [
      {
        "id": "add-proxy-header",
        "enabled": true,
        "priority": 20,
        "direction": "request",
        "conditions": {},
        "action": {
          "type": "add_header",
          "header": "X-Proxied-By",
          "value": "yorishiro-proxy"
        }
      }
    ]
  }
}
```

### Remove a header

Remove a header from requests or responses:

```json
// configure
{
  "auto_transform": {
    "add": [
      {
        "id": "strip-csp",
        "enabled": true,
        "priority": 10,
        "direction": "response",
        "conditions": {},
        "action": {
          "type": "remove_header",
          "header": "Content-Security-Policy"
        }
      }
    ]
  }
}
```

### Replace body content

Search and replace text in the body using regex:

```json
// configure
{
  "auto_transform": {
    "add": [
      {
        "id": "replace-host",
        "enabled": true,
        "priority": 10,
        "direction": "request",
        "conditions": {},
        "action": {
          "type": "replace_body",
          "pattern": "production\\.example\\.com",
          "value": "staging.example.com"
        }
      }
    ]
  }
}
```

## Priority

Rules are applied in priority order, with lower values executed first. This lets you control the order of transformations:

```json
// configure
{
  "auto_transform": {
    "add": [
      {
        "id": "step-1",
        "enabled": true,
        "priority": 10,
        "direction": "request",
        "conditions": {},
        "action": {"type": "set_header", "header": "X-Step", "value": "1"}
      },
      {
        "id": "step-2",
        "enabled": true,
        "priority": 20,
        "direction": "request",
        "conditions": {},
        "action": {"type": "set_header", "header": "X-Step", "value": "2"}
      }
    ]
  }
}
```

In this example, `step-1` runs first but `step-2` overwrites the header value because it runs second.

## Managing rules

### Enable and disable

```json
// configure
{
  "auto_transform": {
    "disable": ["inject-auth"],
    "enable": ["strip-csp"]
  }
}
```

### Remove rules

```json
// configure
{
  "auto_transform": {
    "remove": ["inject-auth"]
  }
}
```

### Replace all rules

```json
// configure
{
  "operation": "replace",
  "auto_transform": {
    "rules": [
      {
        "id": "only-rule",
        "enabled": true,
        "priority": 0,
        "direction": "both",
        "conditions": {},
        "action": {
          "type": "add_header",
          "header": "X-Proxy",
          "value": "yorishiro"
        }
      }
    ]
  }
}
```

## Practical use cases

### Inject authentication headers

Automatically add authentication headers to all API requests:

```json
// configure
{
  "auto_transform": {
    "add": [
      {
        "id": "auto-auth",
        "enabled": true,
        "priority": 10,
        "direction": "request",
        "conditions": {
          "url_pattern": "api\\.target\\.com/.*"
        },
        "action": {
          "type": "set_header",
          "header": "Authorization",
          "value": "Bearer your-api-token"
        }
      }
    ]
  }
}
```

### Strip security headers for testing

Remove security headers from responses to test client-side behavior:

```json
// configure
{
  "auto_transform": {
    "add": [
      {
        "id": "strip-hsts",
        "enabled": true,
        "priority": 10,
        "direction": "response",
        "conditions": {},
        "action": {"type": "remove_header", "header": "Strict-Transport-Security"}
      },
      {
        "id": "strip-csp",
        "enabled": true,
        "priority": 11,
        "direction": "response",
        "conditions": {},
        "action": {"type": "remove_header", "header": "Content-Security-Policy"}
      },
      {
        "id": "strip-xfo",
        "enabled": true,
        "priority": 12,
        "direction": "response",
        "conditions": {},
        "action": {"type": "remove_header", "header": "X-Frame-Options"}
      }
    ]
  }
}
```

### Replace API endpoints

Redirect requests from production to staging:

```json
// configure
{
  "auto_transform": {
    "add": [
      {
        "id": "redirect-to-staging",
        "enabled": true,
        "priority": 10,
        "direction": "request",
        "conditions": {
          "methods": ["POST", "PUT", "DELETE"]
        },
        "action": {
          "type": "replace_body",
          "pattern": "api\\.production\\.com",
          "value": "api.staging.com"
        }
      }
    ]
  }
}
```

## Related pages

- [Configure tool reference](../tools/configure.md) -- Full configuration reference
- [Intercept](intercept.md) -- Manual request/response interception
