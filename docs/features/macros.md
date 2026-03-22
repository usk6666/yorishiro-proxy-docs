# Macros

Macros let you define multi-step workflows that execute a sequence of HTTP requests with variable extraction, template expansion, and conditional logic. They are essential for testing authenticated flows, chained API operations, and any scenario requiring state between requests.

## Core concepts

A macro consists of:

- **Steps** -- ordered HTTP requests based on recorded flows
- **KV Store** -- a key-value store for passing data between steps via `§variable§` templates
- **Extraction rules** -- rules to extract values from responses into the KV Store
- **Guards** -- conditions that control whether a step executes

## Defining a macro

Use the `define_macro` action to save a macro definition. If a macro with the same name exists, it is updated (upsert):

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
        "override_body": "username=admin&password=§password§",
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
        "override_headers": {"Cookie": "PHPSESSID=§session_cookie§"},
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

## Step definition

Each step specifies a recorded flow as a template and optional mutations:

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Unique step identifier (required) |
| `flow_id` | string | Recorded flow to use as template (required) |
| `override_method` | string | Override HTTP method |
| `override_url` | string | Override request URL (supports `§variable§` templates) |
| `override_headers` | object | Header overrides (supports templates) |
| `override_body` | string | Override request body (supports templates) |
| `on_error` | string | Error handling: `"abort"` (default), `"skip"`, `"retry"` |
| `retry_count` | integer | Retry count when on_error is `"retry"` (default: 3) |
| `retry_delay_ms` | integer | Delay between retries in ms (default: 1000) |
| `timeout_ms` | integer | Step timeout in ms (default: 60000) |
| `extract` | array | Value extraction rules |
| `when` | object | Guard condition for conditional execution |

## Variable extraction

Extraction rules pull values from responses and store them in the KV Store for use in subsequent steps.

### Extraction sources

| Source | Description |
|--------|-------------|
| `header` | Extract from a response header (requires `header_name`) |
| `body` | Extract from the response body using regex |
| `body_json` | Extract from a JSON response using `json_path` |
| `status` | Extract the HTTP status code |
| `url` | Extract from the response URL (after redirects) |

### Extraction from headers

```json
{
  "name": "session_cookie",
  "from": "response",
  "source": "header",
  "header_name": "Set-Cookie",
  "regex": "PHPSESSID=([^;]+)",
  "group": 1
}
```

### Extraction from body with regex

```json
{
  "name": "csrf_token",
  "from": "response",
  "source": "body",
  "regex": "name=\"csrf\" value=\"([^\"]+)\"",
  "group": 1
}
```

### Extraction from JSON body

```json
{
  "name": "access_token",
  "from": "response",
  "source": "body_json",
  "json_path": "$.data.access_token"
}
```

### Extraction from status code

```json
{
  "name": "login_status",
  "from": "response",
  "source": "status"
}
```

### Extraction options

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Variable name to store in KV Store (required) |
| `from` | string | `"request"` or `"response"` (required) |
| `source` | string | Extraction source (required) |
| `header_name` | string | Header name (for `header` source) |
| `regex` | string | Regex pattern (for `header` and `body` sources) |
| `group` | integer | Capture group index (default: 0 = full match) |
| `json_path` | string | JSON path (for `body_json` source) |
| `default` | string | Default value if extraction fails |
| `required` | boolean | Fail the step if extraction fails |

## Template expansion

Use `§variable§` syntax in `override_url`, `override_headers`, and `override_body` to reference KV Store values:

```json
{
  "id": "api-call",
  "flow_id": "api-flow",
  "override_url": "https://api.target.com/v1/users/§user_id§",
  "override_headers": {
    "Authorization": "Bearer §access_token§",
    "Cookie": "session=§session_cookie§"
  },
  "override_body": "{\"csrf\": \"§csrf_token§\"}"
}
```

Variables are expanded at execution time, after any preceding step's extractions have populated the KV Store.

## Guards (conditional execution)

Guards control whether a step executes based on conditions from a previous step's result:

```json
{
  "id": "admin-action",
  "flow_id": "admin-flow",
  "when": {
    "step": "login",
    "status_code": 200,
    "body_match": "\"role\":\"admin\""
  }
}
```

### Guard fields

| Field | Type | Description |
|-------|------|-------------|
| `step` | string | Step ID to evaluate (must be a previous step) |
| `status_code` | integer | Require exact status code match |
| `status_code_range` | [int, int] | Require status code within range (inclusive) |
| `header_match` | object | Header name-to-regex mapping (AND logic) |
| `body_match` | string | Regex pattern to match in response body |
| `extracted_var` | string | Require this KV Store variable to exist and be non-empty |
| `negate` | boolean | Invert the guard condition |

### Negated guard example

Skip a step if the login returned a 403:

```json
{
  "id": "dashboard",
  "flow_id": "dashboard-flow",
  "when": {
    "step": "login",
    "status_code": 403,
    "negate": true
  }
}
```

## Running a macro

Execute a stored macro with optional runtime variable overrides:

```json
// macro
{
  "action": "run_macro",
  "params": {
    "name": "auth-flow",
    "vars": {"password": "different-password"}
  }
}
```

Runtime `vars` take precedence over `initial_vars` defined in the macro.

The result includes:

- `status` -- `"completed"`, `"error"`, or `"timeout"`
- `steps_executed` -- number of steps that ran
- `kv_store` -- final KV Store contents
- `step_results` -- per-step status codes, durations, and errors

## Hook integration

Macros power the hook system used by [Resender](resender.md) and [Fuzzer](fuzzer.md). Hooks execute macros at specific points during resend or fuzz operations.

### Pre-send hooks

Execute a macro before sending the main request. The macro's KV Store values are used for template expansion in the main request parameters:

```json
{
  "hooks": {
    "pre_send": {
      "macro": "refresh-auth",
      "run_interval": "always"
    }
  }
}
```

**Run intervals for pre_send:**

| Interval | Description |
|----------|-------------|
| `always` | Run before every request (default) |
| `once` | Run only before the first request |
| `every_n` | Run every N requests (requires `n` parameter) |
| `on_error` | Run when the previous request had an error (4xx/5xx) |

### Post-receive hooks

Execute a macro after receiving the main response:

```json
{
  "hooks": {
    "post_receive": {
      "macro": "log-response",
      "run_interval": "on_status",
      "status_codes": [401, 403],
      "pass_response": true
    }
  }
}
```

When `pass_response` is `true`, the macro receives `__response_status` and `__response_body` as variables.

**Run intervals for post_receive:**

| Interval | Description |
|----------|-------------|
| `always` | Run after every response (default) |
| `on_status` | Run when the response status matches `status_codes` |
| `on_match` | Run when the response body matches `match_pattern` regex |

## Retry policy

Steps can be configured with automatic retry on failure:

```json
{
  "id": "flaky-api",
  "flow_id": "api-flow",
  "on_error": "retry",
  "retry_count": 3,
  "retry_delay_ms": 2000,
  "timeout_ms": 10000
}
```

| Error handling | Description |
|----------------|-------------|
| `abort` | Stop the macro on error (default) |
| `skip` | Skip the failed step and continue |
| `retry` | Retry the step up to `retry_count` times |

## Deleting a macro

```json
// macro
{
  "action": "delete_macro",
  "params": {"name": "auth-flow"}
}
```

## Practical example: login, token, API access

This example demonstrates a complete authentication flow:

```json
// macro
{
  "action": "define_macro",
  "params": {
    "name": "full-auth-flow",
    "description": "Login, extract token, access protected API",
    "steps": [
      {
        "id": "login",
        "flow_id": "login-flow-id",
        "override_body": "{\"username\":\"§username§\",\"password\":\"§password§\"}",
        "extract": [
          {
            "name": "access_token",
            "from": "response",
            "source": "body_json",
            "json_path": "$.data.access_token",
            "required": true
          },
          {
            "name": "refresh_token",
            "from": "response",
            "source": "body_json",
            "json_path": "$.data.refresh_token"
          }
        ]
      },
      {
        "id": "get-profile",
        "flow_id": "profile-flow-id",
        "override_headers": {
          "Authorization": "Bearer §access_token§"
        },
        "when": {
          "step": "login",
          "status_code_range": [200, 299]
        },
        "extract": [
          {
            "name": "user_id",
            "from": "response",
            "source": "body_json",
            "json_path": "$.data.id"
          }
        ]
      },
      {
        "id": "admin-action",
        "flow_id": "admin-flow-id",
        "override_url": "https://api.target.com/admin/users/§user_id§",
        "override_headers": {
          "Authorization": "Bearer §access_token§"
        },
        "when": {
          "step": "get-profile",
          "extracted_var": "user_id"
        }
      }
    ],
    "initial_vars": {
      "username": "admin",
      "password": "admin123"
    },
    "macro_timeout_ms": 60000
  }
}
```

## Related pages

- [Macro tool reference](../tools/macro.md) -- MCP tool parameter reference
- [Resender](resender.md) -- Hook integration for single resends
- [Fuzzer](fuzzer.md) -- Hook integration for fuzz campaigns
