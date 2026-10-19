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
| `override_method` | string | Override HTTP method (supports `§variable§` templates) |
| `override_url` | string | Override request URL (supports `§variable§` templates) |
| `override_headers` | object | Header overrides (templates in values only; names match case-sensitively) |
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

Use `§variable§` syntax in `override_method`, `override_url`, `override_headers` (values), and `override_body` to reference KV Store values. These four fields are the only expansion sites -- `extract` and `when` are never expanded:

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

Values can be piped through encoders: `§payload | url_encode§`, `§credentials | base64§`. Available encoders are `url_encode`, `base64`, `base64_decode`, `hex`, `html_encode`, `lower`, `upper`, `md5`, and `sha256`. An unknown encoder fails the step.

!!! tip "Header names on HTTP/2 flows"
    An `override_headers` key replaces a recorded header only on a case-sensitive name match. HTTP/2 flows record lowercase header names, so write `cookie` instead of `Cookie` when the step's `flow_id` was captured over HTTP/2.

### Unresolved variable warnings

An unknown `§variable§` is left literally on the wire, but yorishiro-proxy reports it instead of sending it silently:

- **At definition time** -- `define_macro` returns `warnings[]` for every `§name§` that is defined by neither `initial_vars`, a preceding step's `extract`, nor a reserved `__` key. The macro is still saved; ignore the warning if you plan to supply the value through `run_macro` `vars`.
- **At run time** -- `run_macro` re-checks each step against the live KV Store just before sending. An unresolved name sets the step's `status` to `"warning"`, and `step_results[*].warnings` names the location plus the variable names that are available, which makes typos easy to spot.

Foreign syntaxes such as `{{var}}`, `${var}`, and `%var%` are never expanded. They are sent literally (so template-injection payloads pass through unmodified) and also produce a step warning that suggests the `§var§` rewrite.

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
- `step_results` -- per-step status (`completed`, `warning`, `skipped`, `error`), status codes, durations, errors, and warnings

## Hook integration

Macros power the `pre_macro` / `post_macro` hooks on the typed fuzz tools ([`fuzz_http`](../tools/fuzz-http.md#macro-hooks), [`fuzz_ws`](../tools/fuzz-ws.md#macro-hooks), [`fuzz_grpc`](../tools/fuzz-grpc.md#macro-hooks), [`fuzz_raw`](../tools/fuzz-raw.md#macro-hooks)). A hook references a stored macro by name and shares its KV Store with the fuzz variants.

```json
// fuzz_http
{
  "flow_id": "target-flow",
  "headers": [
    {"name": "Cookie", "value": "PHPSESSID=§session_cookie§"}
  ],
  "positions": [
    {"path": "raw_query", "payloads": ["id=1", "id=2", "id=3"]}
  ],
  "pre_macro": {"name": "auth-flow", "scope": "job"},
  "post_macro": {
    "name": "log-response",
    "run_interval": "on_status",
    "status_codes": [401, 403]
  }
}
```

### Scopes

| Scope | Fires | KV Store |
|-------|-------|----------|
| `iteration` (default) | Around every variant | Fresh per variant |
| `job` | Once before (pre) or after (post) the variant loop | Shared by every variant |

Job-scope extracts are copied into each variant's KV Store, so a single login can feed every variant.

### Run intervals

`run_interval` applies to `scope="iteration"` hooks only (combining it with `scope="job"` is rejected).

**`pre_macro`:**

| Interval | Description |
|----------|-------------|
| `always` | Run before every variant (default) |
| `once` | Run only before the first variant |
| `every_n` | Run every N variants (requires `n`) |
| `on_error` | Run when the previous variant failed (transport error or HTTP status >= 400 on `fuzz_http`; see each tool for its own error rule) |

**`post_macro`:**

| Interval | Description |
|----------|-------------|
| `always` | Run after every variant (default) |
| `on_status` | Run when the response status matches `status_codes` |
| `on_match` | Run when the response matches the `match_pattern` regex |

An iteration-scope `post_macro` receives the variant's response as reserved `__response_*` variables (for `fuzz_http`: `__response_status`, `__response_body`, `__response_headers__<lower(name)>__`). The key set differs per protocol -- see each fuzz tool's page.

### Error handling and overrides

- `on_error`: `skip` (default) skips the variant -- or the whole job for a job-scope `pre_macro` -- when the hook fails; `abort` stops the run; `continue` proceeds anyway.
- `vars`: static string-to-string overrides injected into the hook's KV Store. Keys starting with `__` are reserved and silently dropped.

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
- [Fuzzer](fuzzer.md) -- `pre_macro` / `post_macro` hooks for fuzz campaigns
- [Resender](resender.md) -- Single request resend
