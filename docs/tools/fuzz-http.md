# fuzz_http

Synchronously fuzz an HTTP request with `HTTPMessage`-typed positions. The schema mirrors [`resend_http`](resend-http.md) plus a `positions[]` list -- each position is a typed path into the `HTTPMessage` shape. The cartesian product of all positions yields the variant sequence.

This tool is **synchronous**: it runs every variant in-process and returns the full per-variant result list when finished. There is no concurrency or rate limit -- pace from the client side. For larger asynchronous fuzz campaigns the legacy runner is being phased out in N9; this tool replaces it for typed HTTP fuzzing.

## Limits

- Maximum variants per call: **1000** (cartesian product across all positions)
- Maximum positions per call: **32**
- Maximum decoded payload size per position: **1 MiB**

Each variant is executed sequentially with a fresh dial. Immediately before the dial, each variant passes two gates on its post-substitution message:

1. **Target scope re-check** -- the variant's `authority`, `scheme`, `path`, `raw_query` (after position substitution and `§var§` expansion) and dial address are re-evaluated against the current [target scope](../features/target-scope.md). A scope change made mid-run (an agent `security` call or a `SIGHUP` policy reload) takes effect on the next variant; a variant already in flight completes.
2. **SafetyFilter input gating** -- same rules as live traffic.

A variant blocked by either gate is recorded with `error` (status `0`, no flow) and the run continues.

## Parameters

`fuzz_http` inherits every base field from [`resend_http`](resend-http.md) (`flow_id`, `method`, `scheme`, `authority`, `path`, `raw_query`, `headers`, `body`, `body_encoding`, `body_set`, `body_patches`, `override_host`, `tls_fingerprint`, `http_version`, `timeout_ms`, `tag`).

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| (base fields) | -- | -- | -- | See [`resend_http`](resend-http.md). When `flow_id` is empty, `method`, `scheme`, `authority`, `path` are required |
| `positions` | array | Yes | | Ordered position list (see below). At least one entry |
| `stop_on_5xx` | boolean | No | `false` | Abort remaining variants once any variant returns a 5xx response |
| `upstream_proxy` | object | No | | Per-variant upstream proxy override (see [Upstream proxy rotation](#upstream-proxy-rotation)) |
| `pre_macro` | object | No | | Macro hook run before a variant's send (see [Macro hooks](#macro-hooks)) |
| `post_macro` | object | No | | Macro hook run after a variant's response (see [Macro hooks](#macro-hooks)) |

### positions

Each position substitutes one typed path into the `HTTPMessage` shape:

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `path` | string | Yes | | Typed path: `method`, `scheme`, `authority`, `path`, `raw_query`, `body`, `headers[N].name`, `headers[N].value` |
| `payloads` | string[] | Yes | | List of values to substitute at this path. At least one element |
| `encoding` | string | No | `"text"` | `"text"` or `"base64"`. Applies to every payload |

Position payloads are written verbatim, including CR/LF. This is intentional: `fuzz_http` is the surface for request smuggling, header injection, and CRLF-injection fuzzing. The base-headers path still rejects CR/LF by design; per-position payloads bypass that guard.

`headers[N]` indexes the base header list (inherited or supplied). The `Host` header auto-injected from `authority` is not addressable via `headers[N]` -- fuzz the `authority` path instead.

### Wire protocol

The wire protocol is chosen once for the whole run with the same rule as [`resend_http`](resend-http.md): an explicit `http_version`, else the recorded flow's protocol (an `h2` / `h2c` flow is fuzzed over HTTP/2), else `http/1.1`. Every variant uses it on its own fresh connection.

Over HTTP/2, no `Host` header is injected (fuzz the target through the `authority` path) and a `scheme` position rewrites `:scheme` on the wire. If the upstream does not negotiate `h2`, each variant fails with an error naming the negotiated ALPN rather than silently falling back -- pass `"http_version": "http/1.1"` to fuzz over HTTP/1.1.

## Upstream proxy rotation

`upstream_proxy` routes each variant through an upstream proxy whose URL is re-expanded **per variant**. Use it for residential-proxy IP rotation, so that each variant exits from a different upstream IP.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `url_template` | string | Yes | Upstream proxy URL template (`http://` or `socks5://`). Supports the `§var§` template syntax with `§__iteration§` (zero-based variant index) and `§__nonce§` (fresh UUID per variant) |

The expanded URL tunnels the variant's dial via HTTP `CONNECT` or SOCKS5. CR/LF in the expanded URL is rejected. When a `pre_macro` hook is configured, the template also sees the variant's KV Store, so values extracted by the hook are available.

```json
// fuzz_http
{
  "flow_id": "abc-123",
  "positions": [
    {"path": "raw_query", "payloads": ["id=1", "id=2", "id=3"]}
  ],
  "upstream_proxy": {
    "url_template": "http://user-session-§__nonce§:password@residential.example:8080"
  }
}
```

## Macro hooks

`pre_macro` and `post_macro` run a stored [macro](macro.md) around variants. Both take the same object shape:

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `name` | string | Yes | | Stored macro name (defined with `macro` `define_macro`) |
| `scope` | string | No | `"iteration"` | `"iteration"` (fires once per variant) or `"job"` (fires once outside the variant loop) |
| `on_error` | string | No | `"skip"` | `"skip"`, `"abort"`, or `"continue"` (see [OnError semantics](#onerror-semantics)) |
| `vars` | object | No | | Static string-to-string KV Store overrides injected before the macro runs. Keys starting with `__` are silently dropped |
| `run_interval` | string | No | `"always"` | Firing cadence, `scope="iteration"` only. `pre_macro`: `always`, `once`, `every_n`, `on_error`. `post_macro`: `always`, `on_status`, `on_match` |
| `n` | integer | Conditional | | Required when `run_interval="every_n"` (must be >= 1) |
| `status_codes` | integer[] | Conditional | | Required when `run_interval="on_status"` |
| `match_pattern` | string | Conditional | | Response body regex. Required when `run_interval="on_match"` |

The hook macro shares the KV Store with the fuzz request: `§var§` templates in the base request (method, path, query, header values, body) resolve against values the `pre_macro` extracted. With `scope="iteration"`, `post_macro` also receives the response as reserved keys: `__response_status`, `__response_body`, and `__response_headers__<lower(name)>__`.

!!! note
    Supplying `headers` replaces the inherited header list wholesale. The examples on this page abbreviate the list to the templated header; in practice, include every header the request needs.

### run_interval

- The `pre_macro` and `post_macro` value sets are disjoint. Using a post-only value (`on_status`, `on_match`) on `pre_macro`, or a pre-only value (`once`, `every_n`, `on_error`) on `post_macro`, is rejected when the call is parsed.
- `on_error` fires when the **previous** variant ended with a transport error or an HTTP status >= 400. It never fires on the first variant. Use it for reactive flows such as "re-authenticate after a 401".
- Setting `run_interval` together with `scope="job"` is rejected: job-scope hooks fire exactly once by construction.

### Hook scopes

| Scope | Pre fires | Post fires | KV Store lifetime | `__response_*` keys | `§__iteration§` / `§__nonce§` |
|-------|-----------|------------|-------------------|---------------------|-------------------------------|
| `iteration` (default) | Before each variant's send | After each variant's response | Fresh per variant | Visible to post | Seeded each variant |
| `job` | Once before the variant loop | Once after the variant loop (including a `stop_on_5xx` exit) | Shared across the whole job | Not visible | Not seeded |

You can mix scopes: `pre_macro` and `post_macro` pick their scope independently. The job-scope KV Store is copied into each variant's store at the start of the variant, and the per-variant reserved keys then overwrite any conflicting job keys. A `pre_macro` with `scope="job"` that extracts `session_token` therefore makes `§session_token§` resolvable in every variant.

### OnError semantics

| `on_error` | `pre_macro`, `scope="iteration"` | `pre_macro`, `scope="job"` | `post_macro` (either scope) |
|------------|----------------------------------|----------------------------|-----------------------------|
| `skip` (default) | Skip the variant: no send, no post hook | Skip the whole job: `completed_variants=0`, `stopped_reason` explains why | Record the error; never aborts the run |
| `abort` | Abort the whole run with an error | Abort before any variant runs | Record the error; never aborts the run |
| `continue` | Log a warning and send anyway; unresolved `§var§` tokens may remain and are reported in the variant's `error` | Log a warning and run the variant loop | Same as `skip` |

Each hook invocation is recorded internally in the `fuzz_macro_results` table (job-scope hooks use variant index `-1`).

## Response

| Field | Type | Description |
|-------|------|-------------|
| `fuzz_id` | string | ID of the `fuzz_jobs` row for this run. Pass it to [`query`](query.md) `fuzz_results` to aggregate variants |
| `total_variants` | integer | Total variant count from the cartesian product |
| `completed_variants` | integer | Variants actually executed before completion or stop |
| `stopped_reason` | string | Empty when all variants ran; `"stop_on_5xx: ..."` or `"ctx cancelled: ..."` otherwise |
| `variants` | array | Per-variant result rows (see below) |
| `duration_ms` | integer | Total run duration in milliseconds |
| `tag` | string | Echo of the supplied tag (when set) |

Each variant row:

| Field | Type | Description |
|-------|------|-------------|
| `index` | integer | Variant index (zero-based, in execution order) |
| `stream_id` | string | New stream record id under which `RecordStep` persisted the variant's flows |
| `status_code` | integer | HTTP response status code (`0` if the variant errored) |
| `body_size` | integer | Response body byte length |
| `payloads` | object | Map of position path -> decoded payload string for this variant |
| `error` | string | Error message when the variant failed (network, timeout, target scope, SafetyFilter) |
| `duration_ms` | integer | Per-variant duration |

Full response payloads are not stored on the row. Retrieve them via the [`query`](query.md) tool keyed by `stream_id`.

## Pipeline placement

Each variant traverses the same self-contained `PluginStepPost -> RecordStep` pipeline as `resend_http` (`PluginStepPre` is bypassed per RFC-001 §9.3).

## Job and result persistence

Synchronous fuzz runs persist a row to `fuzz_jobs` (status `"running"`) and one row per variant to `fuzz_results`, so `query` with `resource = "fuzz_jobs"` and `resource = "fuzz_results"` can list and aggregate variants (status-code distribution, outliers, body filters) after the call returns. Per-variant flows carry `origin = "fuzz"` so analyses can exclude them from live capture via `query` `filter.origin = "proxy"`.

## Examples

### Fuzz a path segment

```json
// fuzz_http
{
  "flow_id": "abc-123",
  "positions": [
    {
      "path": "path",
      "payloads": ["/admin", "/admin/", "/Admin", "/.git/config"]
    }
  ]
}
```

### Two-position cartesian product

```json
// fuzz_http
{
  "flow_id": "abc-123",
  "positions": [
    {"path": "headers[0].value", "payloads": ["Bearer A", "Bearer B"]},
    {"path": "body", "payloads": ["{\"role\":\"user\"}", "{\"role\":\"admin\"}"]}
  ],
  "stop_on_5xx": true
}
```

### Log in once, fuzz with the session

```json
// fuzz_http
{
  "flow_id": "abc-123",
  "headers": [
    {"name": "Authorization", "value": "Bearer §session_token§"}
  ],
  "positions": [
    {"path": "raw_query", "payloads": ["id=1", "id=2", "id=3"]}
  ],
  "pre_macro": {"name": "login-once", "scope": "job"},
  "post_macro": {"name": "audit-summary", "scope": "job"}
}
```

`login-once` runs once before the loop and extracts `session_token`; every variant then sends it in the templated `Authorization` header.

### Fresh session per variant, flag auth failures

```json
// fuzz_http
{
  "flow_id": "abc-123",
  "headers": [
    {"name": "Cookie", "value": "SESSION=§session_cookie§"}
  ],
  "positions": [
    {"path": "body", "payloads": ["{\"id\":1}", "{\"id\":2}", "{\"id\":3}"]}
  ],
  "pre_macro": {"name": "auth-setup"},
  "post_macro": {"name": "alert-on-auth", "run_interval": "on_status", "status_codes": [401, 403]}
}
```

`auth-setup` runs before every variant (default `scope="iteration"`, `run_interval="always"`) and extracts `session_cookie` into that variant's KV Store. `alert-on-auth` runs only when a variant's response is `401` or `403`.

### CRLF smuggling fuzz

```json
// fuzz_http
{
  "flow_id": "abc-123",
  "positions": [
    {
      "path": "raw_query",
      "payloads": [
        "x=1\r\nX-Forwarded-For: 127.0.0.1",
        "x=1\r\nTransfer-Encoding: chunked"
      ]
    }
  ]
}
```

## Related pages

- [resend_http](resend-http.md) -- HTTP resend (single shot)
- [fuzz_ws](fuzz-ws.md) -- WebSocket fuzz
- [fuzz_grpc](fuzz-grpc.md) -- gRPC fuzz
- [fuzz_raw](fuzz-raw.md) -- Raw byte fuzz
- [Fuzzer](../features/fuzzer.md) -- Fuzzer feature guide
- [query](query.md) -- Inspect variant streams by `stream_id`
- [macro](macro.md) -- Define the macros used by `pre_macro` / `post_macro`
- [Upstream proxy](../configuration/upstream-proxy.md) -- Upstream proxy configuration
