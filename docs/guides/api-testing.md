# API testing

This guide covers how to use yorishiro-proxy for testing REST APIs, including authentication testing, parameter tampering, and automated fuzzing.

## Setup

### Start the proxy with API-focused scope

```json
// proxy_start
{
  "listen_addr": "127.0.0.1:8080",
  "capture_scope": {
    "includes": [
      {"hostname": "api.target.com", "url_prefix": "/api/"}
    ],
    "excludes": [
      {"url_prefix": "/api/health"}
    ]
  }
}
```

### Configure your HTTP client

Point your API client at the proxy:

```bash
export HTTP_PROXY=http://127.0.0.1:8080
export HTTPS_PROXY=http://127.0.0.1:8080
```

Or configure the proxy directly in your tool (e.g., `curl -x http://127.0.0.1:8080`).

## Capturing API traffic

### Browse and capture endpoints

Send requests through the proxy to build a map of the API surface:

```bash
curl -x http://127.0.0.1:8080 -k https://api.target.com/api/v1/users
curl -x http://127.0.0.1:8080 -k -X POST https://api.target.com/api/v1/users \
  -H "Content-Type: application/json" \
  -d '{"name": "test", "role": "user"}'
```

### List captured API flows

```json
// query
{
  "resource": "flows",
  "filter": {"url_pattern": "/api/", "method": "POST"},
  "limit": 50
}
```

### Inspect request and response details

```json
// query
{"resource": "flow", "id": "<flow-id>"}
```

The response includes full request/response headers, body, status code, and timing information.

## Authentication testing

### Token replacement

Test authorization boundaries by replacing the Bearer token with another user's token. The `headers` field is a wholesale replacement; in practice fetch the original via `query` first and rebuild the list with your change. The examples below show just the changed entries for brevity.

```json
// resend_http
{
  "flow_id": "<flow-id>",
  "headers": [
    {"name": "Authorization", "value": "Bearer <other-user-token>"}
  ],
  "tag": "auth-token-swap"
}
```

### Removing authentication

Test what happens when you remove the authentication header entirely. Rebuild `headers` from the recorded flow but omit the `Authorization` entry:

```json
// resend_http
{
  "flow_id": "<flow-id>",
  "headers": [
    {"name": "Host", "value": "api.target.com"},
    {"name": "Accept", "value": "application/json"}
  ],
  "tag": "auth-removed"
}
```

### Fuzz authentication headers

Test multiple authentication scenarios at once. The `headers[N].value` path targets the Nth header in the recorded flow; inspect the flow with `query` first to find the index of `Authorization`.

```json
// fuzz_http
{
  "flow_id": "<flow-id>",
  "positions": [
    {
      "path": "headers[0].value",
      "payloads": [
        "",
        "Bearer ",
        "Bearer invalid",
        "Bearer null",
        "Basic YWRtaW46YWRtaW4=",
        "Bearer <low-privilege-token>",
        "Bearer <expired-token>"
      ]
    }
  ],
  "tag": "auth-fuzz"
}
```

### Privilege escalation

Use a low-privilege user's token on admin endpoints:

```json
// resend_http
{
  "flow_id": "<admin-endpoint-flow-id>",
  "headers": [
    {"name": "Authorization", "value": "Bearer <regular-user-token>"}
  ],
  "tag": "privilege-escalation"
}
```

Compare the two responses by retrieving each flow and performing a structural diff on the client side:

```json
// query
{"resource": "flow", "id": "<admin-response-flow-id>"}
```

```json
// query
{"resource": "flow", "id": "<regular-user-response-flow-id>"}
```

Diff the response bodies and status codes locally to identify authorization differences.

## Parameter tampering

### JSON body modification

Modify specific fields in a JSON request body using JSON path patches:

```json
// resend_http
{
  "flow_id": "<flow-id>",
  "body_patches": [
    {"json_path": "$.user.role", "value": "admin"},
    {"json_path": "$.user.id", "value": 1}
  ],
  "tag": "role-escalation"
}
```

### URL manipulation

Change the target URL to access different resources by overriding individual HTTPMessage fields:

```json
// resend_http
{
  "flow_id": "<flow-id>",
  "scheme": "https",
  "authority": "api.target.com",
  "path": "/api/v1/admin/users",
  "tag": "url-manipulation"
}
```

### Method override

Test if the server accepts different HTTP methods:

```json
// resend_http
{
  "flow_id": "<flow-id>",
  "method": "DELETE",
  "tag": "method-override"
}
```

### IDOR testing with fuzzer

Enumerate object IDs to find insecure direct object references. The typed `path` position substitutes the entire request path, so build the full path strings client-side; numeric range expansion is also a client-side concern now.

```json
// fuzz_http
{
  "flow_id": "<flow-id>",
  "positions": [
    {
      "path": "path",
      "payloads": [
        "/users/1",
        "/users/50",
        "/users/100"
      ]
    }
  ],
  "tag": "idor-test"
}
```

Filter results to find successful accesses:

```json
// query
{
  "resource": "fuzz_results",
  "fuzz_id": "<fuzz-id>",
  "filter": {"status_code": 200}
}
```

## Encoded payloads

Individual query-parameter substitution is no longer a typed path; clients construct full `raw_query` strings and apply any URL or other encoding before passing the payloads:

```json
// fuzz_http
{
  "flow_id": "<flow-id>",
  "positions": [
    {
      "path": "raw_query",
      "payloads": [
        "search=%3Cscript%3Ealert(1)%3C%2Fscript%3E",
        "search=%3Cimg+onerror%3Dalert(1)%3E"
      ]
    }
  ],
  "tag": "xss-encoded"
}
```

For binary payloads set `encoding: "base64"` on the position; otherwise apply codecs (URL, hex, HTML entity, etc.) on the client before submitting.

## Automated fuzzing with stop conditions

JSON-path body substitution is no longer a typed path; supply the full body for each variant via the `body` position. Latency-based and pattern-based auto-stop conditions are no longer supported — only `stop_on_5xx` is built in; cancel longer runs from the client.

```json
// fuzz_http
{
  "flow_id": "<flow-id>",
  "positions": [
    {
      "path": "body",
      "payloads": [
        "{\"search\": \"normalvalue\"}",
        "{\"search\": \"' OR SLEEP(3)-- \"}",
        "{\"search\": \"' OR SLEEP(3)#\"}",
        "{\"search\": \"1 OR SLEEP(3)\"}",
        "{\"search\": \"1; WAITFOR DELAY '0:0:3'--\"}"
      ]
    }
  ],
  "stop_on_5xx": true,
  "tag": "sqli-blind"
}
```

Inspect per-variant `duration_ms` in the response to identify timing anomalies indicative of blind SQL injection.

## Managing fuzz jobs

### Check job status

```json
// query
{"resource": "fuzz_jobs"}
```

### Cancel a job

The typed fuzz tools are synchronous; cancel a long run by canceling the MCP call from the client.

## Analyzing results

### Get results with statistics

```json
// query
{
  "resource": "fuzz_results",
  "fuzz_id": "<fuzz-id>",
  "sort_by": "status_code",
  "limit": 100
}
```

The `summary.statistics` section includes status code distribution and body length/timing distributions (min, max, median, stddev).

### Find outliers

```json
// query
{
  "resource": "fuzz_results",
  "fuzz_id": "<fuzz-id>",
  "filter": {"outliers_only": true}
}
```

Outliers are detected by status code deviation, body length deviation (median +/- 2 stddev), and timing deviation.

### Search response bodies

```json
// query
{
  "resource": "fuzz_results",
  "fuzz_id": "<fuzz-id>",
  "filter": {"body_contains": "error"}
}
```

## Exporting results

Export API test flows in HAR format for sharing with other tools:

```json
// manage
{
  "action": "export_flows",
  "params": {
    "format": "har",
    "filter": {"url_pattern": "/api/"},
    "output_path": "/tmp/api-test-results.har"
  }
}
```

HAR files are compatible with browser DevTools, Burp Suite, and OWASP ZAP.

## Related pages

- [Vulnerability assessment](vulnerability-assessment.md) -- full assessment workflow
- [Resender](../features/resender.md) -- resender feature reference
- [Fuzzer](../features/fuzzer.md) -- fuzzer feature reference
- [resend_http](../tools/resend-http.md), [resend_ws](../tools/resend-ws.md), [resend_grpc](../tools/resend-grpc.md), [resend_raw](../tools/resend-raw.md) -- typed per-protocol resend MCP tools
- [fuzz_http](../tools/fuzz-http.md), [fuzz_ws](../tools/fuzz-ws.md), [fuzz_grpc](../tools/fuzz-grpc.md), [fuzz_raw](../tools/fuzz-raw.md) -- typed per-protocol fuzz MCP tools
