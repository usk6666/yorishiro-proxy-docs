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

Test authorization boundaries by replacing the Bearer token with another user's token:

```json
// resend
{
  "action": "resend",
  "params": {
    "flow_id": "<flow-id>",
    "override_headers": {
      "Authorization": "Bearer <other-user-token>"
    },
    "tag": "auth-token-swap"
  }
}
```

### Removing authentication

Test what happens when you remove the authentication header entirely:

```json
// resend
{
  "action": "resend",
  "params": {
    "flow_id": "<flow-id>",
    "remove_headers": ["Authorization"],
    "tag": "auth-removed"
  }
}
```

### Fuzz authentication headers

Test multiple authentication scenarios at once:

```json
// fuzz
{
  "action": "fuzz",
  "params": {
    "flow_id": "<flow-id>",
    "attack_type": "sequential",
    "positions": [
      {
        "id": "pos-0",
        "location": "header",
        "name": "Authorization",
        "payload_set": "auth-values"
      }
    ],
    "payload_sets": {
      "auth-values": {
        "type": "wordlist",
        "values": [
          "",
          "Bearer ",
          "Bearer invalid",
          "Bearer null",
          "Basic YWRtaW46YWRtaW4=",
          "Bearer <low-privilege-token>",
          "Bearer <expired-token>"
        ]
      }
    },
    "tag": "auth-fuzz"
  }
}
```

### Privilege escalation

Use a low-privilege user's token on admin endpoints:

```json
// resend
{
  "action": "resend",
  "params": {
    "flow_id": "<admin-endpoint-flow-id>",
    "override_headers": {
      "Authorization": "Bearer <regular-user-token>"
    },
    "tag": "privilege-escalation"
  }
}
```

Compare the responses:

```json
// resend
{
  "action": "compare",
  "params": {
    "flow_id_a": "<admin-response-flow-id>",
    "flow_id_b": "<regular-user-response-flow-id>"
  }
}
```

## Parameter tampering

### JSON body modification

Modify specific fields in a JSON request body using JSON path patches:

```json
// resend
{
  "action": "resend",
  "params": {
    "flow_id": "<flow-id>",
    "body_patches": [
      {"json_path": "$.user.role", "value": "admin"},
      {"json_path": "$.user.id", "value": 1}
    ],
    "tag": "role-escalation"
  }
}
```

### URL manipulation

Change the target URL to access different resources:

```json
// resend
{
  "action": "resend",
  "params": {
    "flow_id": "<flow-id>",
    "override_url": "https://api.target.com/api/v1/admin/users",
    "tag": "url-manipulation"
  }
}
```

### Method override

Test if the server accepts different HTTP methods:

```json
// resend
{
  "action": "resend",
  "params": {
    "flow_id": "<flow-id>",
    "override_method": "DELETE",
    "tag": "method-override"
  }
}
```

### IDOR testing with fuzzer

Enumerate object IDs to find insecure direct object references:

```json
// fuzz
{
  "action": "fuzz",
  "params": {
    "flow_id": "<flow-id>",
    "attack_type": "sequential",
    "positions": [
      {
        "id": "pos-0",
        "location": "path",
        "match": "/users/(\\d+)",
        "payload_set": "user-ids"
      }
    ],
    "payload_sets": {
      "user-ids": {"type": "range", "start": 1, "end": 100}
    },
    "rate_limit_rps": 5,
    "tag": "idor-test"
  }
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

When the API expects encoded input, use codec chains to encode payloads before injection:

```json
// fuzz
{
  "action": "fuzz",
  "params": {
    "flow_id": "<flow-id>",
    "attack_type": "sequential",
    "positions": [
      {
        "id": "pos-0",
        "location": "query",
        "name": "search",
        "payload_set": "xss"
      }
    ],
    "payload_sets": {
      "xss": {
        "type": "wordlist",
        "values": [
          "<script>alert(1)</script>",
          "<img onerror=alert(1)>"
        ],
        "encoding": ["url_encode_query"]
      }
    },
    "tag": "xss-encoded"
  }
}
```

Available codecs include `base64`, `url_encode_query`, `url_encode_path`, `hex`, `html_entity`, `double_url_encode`, and more. Codecs are applied in order as a pipeline.

## Automated fuzzing with stop conditions

Configure automatic stop conditions to terminate the fuzz job when anomalies are detected:

```json
// fuzz
{
  "action": "fuzz",
  "params": {
    "flow_id": "<flow-id>",
    "attack_type": "sequential",
    "positions": [
      {
        "id": "pos-0",
        "location": "body_json",
        "json_path": "$.search",
        "payload_set": "sqli"
      }
    ],
    "payload_sets": {
      "sqli": {
        "type": "wordlist",
        "values": [
          "normalvalue",
          "' OR SLEEP(3)-- ",
          "' OR SLEEP(3)#",
          "1 OR SLEEP(3)",
          "1; WAITFOR DELAY '0:0:3'--"
        ]
      }
    },
    "stop_on": {
      "latency_threshold_ms": 5000,
      "latency_baseline_multiplier": 3.0,
      "latency_window": 5
    },
    "tag": "sqli-blind"
  }
}
```

The job stops automatically when response latency exceeds the threshold, indicating a potential SQL injection.

## Managing fuzz jobs

### Check job status

```json
// query
{"resource": "fuzz_jobs"}
```

### Pause and resume

```json
// fuzz
{"action": "fuzz_pause", "params": {"fuzz_id": "<fuzz-id>"}}
```

```json
// fuzz
{"action": "fuzz_resume", "params": {"fuzz_id": "<fuzz-id>"}}
```

### Cancel a job

```json
// fuzz
{"action": "fuzz_cancel", "params": {"fuzz_id": "<fuzz-id>"}}
```

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
- [resend tool](../tools/resend.md) -- resend tool reference
- [fuzz tool](../tools/fuzz.md) -- fuzz tool reference
