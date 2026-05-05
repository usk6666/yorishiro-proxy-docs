# Fuzzer

The fuzzer executes automated testing campaigns by injecting payloads into recorded proxy requests. You define injection positions and payload lists, then the fuzz tool runs synchronously, records each variant as a new stream, and returns the per-variant result list.

Each protocol has its own typed fuzz tool: [`fuzz_http`](../tools/fuzz-http.md), [`fuzz_ws`](../tools/fuzz-ws.md), [`fuzz_grpc`](../tools/fuzz-grpc.md), [`fuzz_raw`](../tools/fuzz-raw.md). Pick the tool that matches the recorded flow's protocol.

## How it works

1. Select a recorded flow as the template
2. Define typed positions where payloads should be injected
3. Call the typed fuzz tool -- it runs synchronously and returns the per-variant result list
4. Inspect individual variant streams via the [`query`](../tools/query.md) tool keyed by `stream_id`

## Limits

- Maximum variants per call: **1000** (cartesian product across all positions)
- Maximum positions per call (HTTP): **32**
- Maximum decoded payload size per position: **1 MiB**

The variant sequence is the cartesian product of all positions' payloads. Each variant is executed sequentially with a fresh dial.

## Position definitions

Positions specify where payloads are injected. Each position has:

- **`path`** -- typed path into the protocol's `Message` shape
- **`payloads`** -- list of values to substitute at this path
- **`encoding`** -- `"text"` (default) or `"base64"`

The position `path` values for `fuzz_http` are: `method`, `scheme`, `authority`, `path`, `raw_query`, `body`, `headers[N].name`, `headers[N].value`. Cookies live inside the `Cookie` header; fuzz them via `headers[N].value`. The legacy `body_regex` and partial-match `match` modes are gone -- when you need surgical body edits, combine `body_patches` (inherited from [`resend_http`](../tools/resend-http.md)) with full-value position payloads.

### Header position

Replace a header value (find the header's index from the recorded flow):

```json
// fuzz_http
{
  "flow_id": "abc-123",
  "positions": [
    {
      "path": "headers[3].value",
      "payloads": ["Bearer token1", "Bearer token2", "Bearer admin-token"]
    }
  ]
}
```

### Query position

Inject into a query string -- the typed surface fuzzes the full `raw_query`:

```json
// fuzz_http
{
  "flow_id": "abc-123",
  "positions": [
    {
      "path": "raw_query",
      "payloads": ["search=foo", "search=<script>alert(1)</script>"]
    }
  ]
}
```

For more precise per-parameter substitution, pre-compute the candidate query strings and supply them as the payload list.

### Body position

Substitute the entire request body:

```json
// fuzz_http
{
  "flow_id": "abc-123",
  "positions": [
    {
      "path": "body",
      "payloads": [
        "{\"password\":\"pass1\"}",
        "{\"password\":\"pass2\"}"
      ]
    }
  ]
}
```

For partial JSON edits, combine `body_patches` (inherited from `resend_http`) with one or more `body` positions.

### Cookie

Cookies are sent via the `Cookie` request header. Locate the header's index in the recorded flow and fuzz `headers[N].value`:

```json
// fuzz_http
{
  "flow_id": "abc-123",
  "positions": [
    {
      "path": "headers[2].value",
      "payloads": ["session=abc", "session=def", "session=admin"]
    }
  ]
}
```

## Payload encoding

Each position has a single `encoding` (`"text"` default or `"base64"`). Codec chains (the legacy `["base64", "url_encode_query", ...]` arrays) are no longer applied at the fuzz tool layer -- pre-encode payloads from the client side or use `body_patches` with the `encoding` array for body edits. To inject pre-encoded binary payloads:

```json
// fuzz_http
{
  "flow_id": "abc-123",
  "positions": [
    {
      "path": "body",
      "encoding": "base64",
      "payloads": [
        "PHNjcmlwdD5hbGVydCgxKTwvc2NyaXB0Pg==",
        "JyBPUiAxPTEgLS0="
      ]
    }
  ]
}
```

## Attack patterns

The typed fuzz tools always cartesian-product positions × payloads, capped at 1000 variants. The legacy `sequential` / `parallel` distinction no longer exists -- if you need a single-position-at-a-time scan, issue one fuzz call per position. If you need lockstep "zip" semantics, pre-compute the lockstepped payload list and supply it as a single position.

### Cartesian product example

Two positions × two payloads each = four variants:

```json
// fuzz_http
{
  "flow_id": "abc-123",
  "positions": [
    {"path": "raw_query", "payloads": ["user=admin", "user=root"]},
    {"path": "body", "payloads": ["{\"password\":\"pass1\"}", "{\"password\":\"pass2\"}"]}
  ]
}
```

## Execution parameters

The typed tools are synchronous and serial. The legacy `concurrency`, `rate_limit_rps`, `delay_ms`, `max_retries` parameters have been removed -- pace from the client side by chunking your payload list across multiple calls. Per-variant timeouts use the inherited `timeout_ms` (default `30000`).

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `timeout_ms` | integer | `30000` | Per-variant timeout in ms |
| `tag` | string | | Tag stored on every variant Stream's `Tags` map |

## Synchronous execution

The typed fuzz tools run in-process and return the full per-variant result list when finished. There is no separate fuzz-job lifecycle to start, pause, resume, or cancel -- to interrupt a long run, cancel the MCP call from the client. Each variant is recorded as its own stream and remains queryable after the run.

### Running a campaign

```json
// fuzz_http
{
  "flow_id": "abc-123",
  "positions": [
    {
      "path": "raw_query",
      "payloads": ["q=<script>", "q={{7*7}}", "q=' OR 1=1"]
    }
  ],
  "tag": "xss-scan"
}
```

The response contains `variants[]`, each with a `stream_id`, `status_code`, `body_size`, the payload map, and per-variant `duration_ms`. Cancel the MCP call from the client to stop the run mid-campaign.

## Stop conditions

Each typed tool exposes a single boolean stop flag, picked to match the protocol:

| Tool | Flag | Triggers when |
|------|------|---------------|
| `fuzz_http` | `stop_on_5xx` | A variant returns a 5xx response |
| `fuzz_ws` | `stop_on_close` | A variant receives a Close frame from upstream |
| `fuzz_grpc` | `stop_on_non_ok` | A variant returns a non-OK gRPC status |
| `fuzz_raw` | `stop_on_error` | A variant fails (network error, timeout, pipeline drop) |

The legacy `error_count`, `latency_threshold_ms`, `latency_baseline_multiplier`, and `latency_window` conditions are gone -- inspect the returned `variants[]` rows after the call to apply your own thresholds.

```json
// fuzz_http
{
  "flow_id": "abc-123",
  "positions": [
    {
      "path": "raw_query",
      "payloads": ["id=1", "id=2", "id=3"]
    }
  ],
  "stop_on_5xx": true
}
```

## Querying results

Each variant's full request and response are stored under its `stream_id`. Use the [`query`](../tools/query.md) tool to retrieve them:

```json
// query
{
  "resource": "stream",
  "id": "<variant-stream-id>"
}
```

The fuzz response itself includes per-variant `status_code`, `body_size`, `duration_ms`, payload map, and any `error` -- enough to triage results without an extra round trip.

## Authentication refresh

Per-call macro hooks (`pre_send` / `post_receive`) have been removed. To refresh authentication mid-campaign, split the run into chunks and call your auth-refresh macro between chunks from the client side. See [Macros](macros.md) for the standalone macro execution model.

## Practical use cases

### XSS scanning

```json
// fuzz_http
{
  "flow_id": "search-flow",
  "positions": [
    {
      "path": "raw_query",
      "payloads": [
        "q=%3Cscript%3Ealert(1)%3C%2Fscript%3E",
        "q=%3Cimg+src%3Dx+onerror%3Dalert(1)%3E",
        "q=%7B%7B7*7%7D%7D",
        "q=%24%7B7*7%7D"
      ]
    }
  ],
  "tag": "xss-scan"
}
```

### Brute force PIN

For exhaustive enumerations like 4-digit PINs, generate the candidate list client-side and chunk across multiple fuzz calls (the per-call cap is 1000 variants):

```json
// fuzz_http
{
  "flow_id": "pin-verify-flow",
  "positions": [
    {
      "path": "body",
      "payloads": [
        "{\"pin\":\"0000\"}",
        "{\"pin\":\"0001\"}",
        "{\"pin\":\"0002\"}"
      ]
    }
  ],
  "stop_on_5xx": false,
  "tag": "pin-brute"
}
```

### IDOR enumeration

```json
// fuzz_http
{
  "flow_id": "user-profile-flow",
  "positions": [
    {
      "path": "raw_query",
      "payloads": ["user_id=1", "user_id=2", "user_id=3", "user_id=4"]
    }
  ],
  "tag": "idor-scan"
}
```

## Related pages

- [fuzz_http](../tools/fuzz-http.md), [fuzz_ws](../tools/fuzz-ws.md), [fuzz_grpc](../tools/fuzz-grpc.md), [fuzz_raw](../tools/fuzz-raw.md) -- typed per-protocol fuzz MCP tools
- [Resender](resender.md) -- Single request resend with mutations
- [Built-in codecs](../reference/built-in-codecs.md) -- Available encoding codecs
