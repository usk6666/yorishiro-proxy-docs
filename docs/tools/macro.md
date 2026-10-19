# macro

Define and execute macro workflows for multi-step security testing. Macros chain multiple requests together, extract values from responses, and use them in subsequent steps.

!!! note "HTTP-only"
    Macros operate on HTTP flows only. `define_macro` and `run_macro` reject any step that references a non-HTTP recorded flow (WebSocket, gRPC, Raw TCP, etc.) at entry, directing callers to the protocol-specific resend tools: [`resend_ws`](resend-ws.md), [`resend_grpc`](resend-grpc.md), [`resend_raw`](resend-raw.md).

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
| `override_method` | string | No | | Override HTTP method (supports `§variable§` templates) |
| `override_url` | string | No | | Override request URL (supports `§variable§` templates) |
| `override_headers` | object | No | | Header overrides as key-value pairs (templates in values only). Keys match recorded header names case-sensitively |
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
| `warnings` | string[] | Non-fatal template diagnostics from a static check (see [Unknown variable warnings](#unknown-variable-warnings)). The macro is always saved |

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
| `step_results` | array | Per-step results: `id`, `status` (`"completed"`, `"warning"`, `"skipped"`, `"error"`), `status_code`, `duration_ms`, `error`, `warnings` |
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

## Variable substitution syntax

The only template syntax is `§name§` -- U+00A7 SECTION SIGN on both sides. References are replaced from the KV Store, which is populated by `initial_vars`, `run_macro` `vars`, earlier steps' `extract` results, and runtime-reserved keys.

```text
override_method:  "§http_verb§"
override_url:     "https://api.example.com/users/§user_id§"
override_headers: {"Cookie": "PHPSESSID=§session_cookie§"}
override_body:    "user=admin&token=§csrf_token§"
```

- The four `override_*` fields are the **only** expansion sites. In `override_headers`, only values are expanded; header names are copied verbatim. `extract` and `when` are never expanded.
- An `override_headers` key replaces a recorded header only when the names match **case-sensitively**. HTTP/2 flows record lowercase names, so use `cookie` rather than `Cookie` on an h2-recorded flow -- otherwise the override is added next to the recorded header instead of replacing it.
- `{{name}}`, `${name}`, and `%name%` are **not** expanded. They are sent literally, so template-injection payloads pass through unmodified. A residual token of these forms produces a step warning that suggests the `§name§` rewrite.
- Keys starting with `__` (such as `§__nonce§`, `§__iteration§`) are reserved for runtime state. User-supplied values for them are dropped.

### Encoder chain

Pipe a value through one or more encoders: `§name | encoder1 | encoder2§`. Available encoders: `url_encode`, `base64`, `base64_decode`, `hex`, `html_encode`, `lower`, `upper`, `md5`, `sha256`.

```text
override_url:  "https://example.com/?q=§payload | url_encode§"
override_body: "auth=§credentials | base64§"
```

An unknown encoder name fails the step. An unknown **variable** does not -- see below.

### Unknown variable warnings

A `§name§` that does not resolve (for example, the typo `§sesion_cookie§`) is still sent literally on the wire, but it is never silent:

- **`define_macro`** runs a static check and returns `warnings[]`. A name is reported when it is defined by neither `initial_vars`, a **preceding** step's `extract`, nor a reserved `__` key. This is a warning, not an error -- the macro is saved, and you can still supply the value through `run_macro` `vars`. A step's own `extract` does not count for that step, because extraction runs after the request is sent.
- **`run_macro`** re-checks each step against the live KV Store just before sending. The step's `status` becomes `"warning"` and its `warnings[]` lists the unresolved name together with the variable names that *are* available (names only -- values are never echoed):

```text
header:Cookie: §sesion_cookie§ — variable not found in KV Store;
available: csrf_token, session_cookie, __nonce
```

Only the `override_*` fields are scanned; a `§` that appears in the recorded base flow is left alone. Steps skipped by their `when` guard are not scanned.

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
- [fuzz_http](fuzz-http.md#macro-hooks) -- Run macros as `pre_macro` / `post_macro` fuzz hooks
- [query](query.md) -- Query macros with `macros` and `macro` resources
- [resend_http](resend-http.md), [resend_ws](resend-ws.md), [resend_grpc](resend-grpc.md), [resend_raw](resend-raw.md) -- typed per-protocol resend tools
- [fuzz_http](fuzz-http.md), [fuzz_ws](fuzz-ws.md), [fuzz_grpc](fuzz-grpc.md), [fuzz_raw](fuzz-raw.md) -- typed per-protocol fuzz tools
