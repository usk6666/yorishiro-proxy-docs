# macro

Define and execute macro workflows for multi-step security testing. Macros chain multiple requests together, extract values from responses, and use them in subsequent steps.

## Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `action` | string | Yes | Action to perform: `define_macro`, `run_macro`, `delete_macro` |
| `params` | object | Yes | Action-specific parameters (see below) |

## Actions

### define_macro

Save a macro definition (upsert). If a macro with the same name exists, it is updated.

#### Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `name` | string | Yes | | Unique macro identifier |
| `description` | string | No | | Human-readable description |
| `steps` | array | Yes | | Ordered list of macro steps |
| `initial_vars` | object | No | | Pre-populated KV Store entries |
| `macro_timeout_ms` | integer | No | `300000` | Overall macro timeout in ms |

#### steps

Each step defines a request to send:

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `id` | string | Yes | | Unique step identifier |
| `flow_id` | string | Yes | | Recorded flow to use as template |
| `override_method` | string | No | | Override HTTP method |
| `override_url` | string | No | | Override request URL (supports `{{variable}}` templates) |
| `override_headers` | object | No | | Header overrides as key-value pairs (supports templates) |
| `override_body` | string | No | | Override request body (supports templates) |
| `on_error` | string | No | `"abort"` | Error handling: `"abort"`, `"skip"`, or `"retry"` |
| `retry_count` | integer | No | `3` | Retry count when on_error is `"retry"` |
| `retry_delay_ms` | integer | No | `1000` | Delay between retries in ms |
| `timeout_ms` | integer | No | `60000` | Step timeout in ms |
| `extract` | array | No | | Value extraction rules |
| `when` | object | No | | Step guard condition |

#### extract

Each extraction rule defines how to capture a value from the request or response:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Variable name to store the extracted value |
| `from` | string | Yes | `"request"` or `"response"` |
| `source` | string | Yes | `"header"`, `"body"`, `"body_json"`, `"status"`, `"url"` |
| `header_name` | string | Conditional | Header name (required when source is `"header"`) |
| `regex` | string | No | Regex pattern for extraction |
| `group` | integer | No | Capture group number |
| `json_path` | string | No | JSON path for body_json source |
| `default` | string | No | Default value if extraction fails |
| `required` | boolean | No | Fail the step if extraction fails |

#### when

Step guard condition -- the step is only executed if the condition is met:

| Field | Type | Description |
|-------|------|-------------|
| `step` | string | Reference to a previous step ID |
| `status_code` | integer | Required status code from the referenced step |
| `status_code_range` | [int, int] | Required status code range [min, max] |
| `header_match` | object | Maps header names to required regex patterns |
| `body_match` | string | Required regex pattern in the response body |
| `extracted_var` | string | Required extracted variable name (must be non-empty) |
| `negate` | boolean | Invert the condition |

#### Response

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Macro name |
| `step_count` | integer | Number of steps |
| `created` | boolean | `true` if new, `false` if updated |

### run_macro

Execute a stored macro.

#### Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `name` | string | Yes | Name of the macro to run |
| `vars` | object | No | Runtime variable overrides for the KV Store |

#### Response

| Field | Type | Description |
|-------|------|-------------|
| `macro_name` | string | Macro name |
| `status` | string | `"completed"`, `"error"`, or `"timeout"` |
| `steps_executed` | integer | Number of steps executed |
| `kv_store` | object | Final KV Store contents |
| `step_results` | array | Per-step results (id, status, status_code, duration_ms, error) |
| `error` | string | Error message (if any) |

### delete_macro

Remove a stored macro definition.

#### Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `name` | string | Yes | Name of the macro to delete |

#### Response

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Macro name |
| `deleted` | boolean | `true` if deleted |

## Examples

### Define a login macro with token extraction

```json
// macro
{
  "action": "define_macro",
  "params": {
    "name": "auth-flow",
    "description": "Login and get CSRF token",
    "steps": [
      {
        "id": "login",
        "flow_id": "recorded-login-flow",
        "override_body": "username=admin&password={{password}}",
        "extract": [
          {
            "name": "session_cookie",
            "from": "response",
            "source": "header",
            "header_name": "Set-Cookie",
            "regex": "PHPSESSID=([^;]+)",
            "group": 1
          }
        ]
      },
      {
        "id": "get-csrf",
        "flow_id": "recorded-csrf-flow",
        "override_headers": {"Cookie": "PHPSESSID={{session_cookie}}"},
        "extract": [
          {
            "name": "csrf_token",
            "from": "response",
            "source": "body",
            "regex": "name=\"csrf\" value=\"([^\"]+)\"",
            "group": 1
          }
        ]
      }
    ],
    "initial_vars": {"password": "admin123"}
  }
}
```

### Run a macro

```json
// macro
{
  "action": "run_macro",
  "params": {
    "name": "auth-flow",
    "vars": {"password": "override-password"}
  }
}
```

### Delete a macro

```json
// macro
{
  "action": "delete_macro",
  "params": {"name": "auth-flow"}
}
```

## Related pages

- [Macros](../features/macros.md) -- Macro feature guide
- [query](query.md) -- Query macros with `macros` and `macro` resources
- [resend](resend.md) -- Resend requests with macro hooks
- [fuzz](fuzz.md) -- Fuzz testing with macro hooks
