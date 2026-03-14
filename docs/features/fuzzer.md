# Fuzzer

The fuzzer executes automated testing campaigns by injecting payloads into recorded proxy requests. You define injection positions, payload sets, and execution parameters, then the fuzzer runs asynchronously and records all results as new flows.

## How it works

1. Select a recorded flow as the template
2. Define positions where payloads should be injected
3. Configure payload sets with the values to inject
4. Start the fuzz job -- it runs asynchronously and returns a `fuzz_id`
5. Query results with the `query` tool using `fuzz_results`

## Position definitions

Positions specify where payloads are injected in the request. Each position requires:

- **`id`** -- unique identifier (e.g. `"pos-0"`)
- **`location`** -- where to inject: `header`, `path`, `query`, `body_regex`, `body_json`, `cookie`
- **`payload_set`** -- which payload set to use

### Header position

Replace or add a header value:

```json
// fuzz
{
  "action": "fuzz",
  "params": {
    "flow_id": "abc-123",
    "attack_type": "sequential",
    "positions": [
      {
        "id": "pos-0",
        "location": "header",
        "name": "Authorization",
        "mode": "replace",
        "match": "Bearer (.*)",
        "payload_set": "tokens"
      }
    ],
    "payload_sets": {
      "tokens": {
        "type": "wordlist",
        "values": ["token1", "token2", "admin-token"]
      }
    }
  }
}
```

The `match` field uses regex for partial replacement. When a capture group is specified, only the group is replaced.

### Query position

Inject into a URL query parameter:

```json
{
  "id": "pos-0",
  "location": "query",
  "name": "search",
  "payload_set": "xss"
}
```

### Body JSON position

Inject into a JSON body field using JSON path:

```json
{
  "id": "pos-0",
  "location": "body_json",
  "json_path": "$.password",
  "payload_set": "passwords"
}
```

### Body regex position

Inject by matching a regex pattern in the body:

```json
{
  "id": "pos-0",
  "location": "body_regex",
  "match": "csrf_token=([^&]+)",
  "payload_set": "tokens"
}
```

### Cookie position

Inject into a specific cookie:

```json
{
  "id": "pos-0",
  "location": "cookie",
  "name": "session",
  "payload_set": "session_ids"
}
```

### Position modes

| Mode | Description |
|------|-------------|
| `replace` | Replace the current value (default) |
| `add` | Add a new entry (for headers, query params) |
| `remove` | Remove the entry entirely (no payload_set needed) |

## Payload sets

Payload sets define the values to inject at each position.

### Wordlist

Inline list of payload strings:

```json
{
  "type": "wordlist",
  "values": ["<script>alert(1)</script>", "' OR 1=1 --", "{{7*7}}"]
}
```

### File

Load payloads from a file under `~/.yorishiro-proxy/wordlists/`:

```json
{
  "type": "file",
  "path": "sqli/generic.txt"
}
```

### Range

Generate integer payloads from start to end:

```json
{
  "type": "range",
  "start": 1,
  "end": 100,
  "step": 1
}
```

### Sequence

Generate formatted sequential strings:

```json
{
  "type": "sequence",
  "start": 1,
  "end": 50,
  "format": "user%04d"
}
```

This produces `user0001`, `user0002`, ..., `user0050`.

### Charset

Generate all combinations of a character set with a given length:

```json
{
  "type": "charset",
  "charset": "0123456789",
  "length": 4
}
```

This generates all 4-digit PINs: `0000`, `0001`, ..., `9999`.

### Case variation

Generate all case variations of a string:

```json
{
  "type": "case_variation",
  "input": "Admin"
}
```

This produces `admin`, `Admin`, `ADMIN`, `aDmin`, etc.

### Null byte injection

Generate null byte injection variants of a string:

```json
{
  "type": "null_byte_injection",
  "input": "test.php"
}
```

## Encoding chains

Each payload set can include an `encoding` array that applies codec transformations to every payload before injection:

```json
{
  "type": "wordlist",
  "values": ["<script>alert(1)</script>", "<img onerror=alert(1)>"],
  "encoding": ["url_encode_query"]
}
```

Codecs are applied in pipeline order. For example, `["url_encode_query", "base64"]` first URL-encodes the payload, then Base64-encodes the result.

Available codecs: `base64`, `base64url`, `url_encode_query`, `url_encode_path`, `url_encode_full`, `double_url_encode`, `hex`, `html_entity`, `html_escape`, `unicode_escape`, `md5`, `sha256`, `lower`, `upper`.

Maximum chain length is 10 codecs.

## Attack types

### Sequential mode

Tests one position at a time while keeping other positions at their original values. Each position cycles through all payloads from its payload set independently.

Total requests = sum of all payload set sizes.

```json
// fuzz
{
  "action": "fuzz",
  "params": {
    "flow_id": "abc-123",
    "attack_type": "sequential",
    "positions": [
      {"id": "pos-0", "location": "query", "name": "username", "payload_set": "users"},
      {"id": "pos-1", "location": "body_json", "json_path": "$.password", "payload_set": "passwords"}
    ],
    "payload_sets": {
      "users": {"type": "wordlist", "values": ["admin", "root"]},
      "passwords": {"type": "wordlist", "values": ["pass1", "pass2", "pass3"]}
    }
  }
}
```

### Parallel mode

Applies payloads to all positions simultaneously using a zip strategy. Payload sets are consumed in lockstep.

Total requests = max payload set size.

```json
// fuzz
{
  "action": "fuzz",
  "params": {
    "flow_id": "abc-123",
    "attack_type": "parallel",
    "positions": [
      {"id": "pos-0", "location": "query", "name": "username", "payload_set": "users"},
      {"id": "pos-1", "location": "body_json", "json_path": "$.password", "payload_set": "passwords"}
    ],
    "payload_sets": {
      "users": {"type": "wordlist", "values": ["admin", "root", "user"]},
      "passwords": {"type": "wordlist", "values": ["pass1", "pass2", "pass3"]}
    }
  }
}
```

## Execution parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `concurrency` | integer | `1` | Number of concurrent workers |
| `rate_limit_rps` | number | `0` (unlimited) | Requests per second limit |
| `delay_ms` | integer | `0` | Fixed delay between requests in ms |
| `timeout_ms` | integer | `10000` | Per-request timeout in ms |
| `max_retries` | integer | `0` | Retry count per failed request |

## Async execution and job management

Fuzz jobs run asynchronously. The `fuzz` action returns a `fuzz_id` immediately.

### Starting a job

```json
// fuzz
{
  "action": "fuzz",
  "params": {
    "flow_id": "abc-123",
    "attack_type": "sequential",
    "positions": [
      {"id": "pos-0", "location": "query", "name": "q", "payload_set": "xss"}
    ],
    "payload_sets": {
      "xss": {"type": "wordlist", "values": ["<script>", "{{7*7}}", "' OR 1=1"]}
    },
    "concurrency": 5,
    "rate_limit_rps": 10,
    "tag": "xss-scan"
  }
}
```

### Pausing a job

```json
// fuzz
{
  "action": "fuzz_pause",
  "params": {"fuzz_id": "fuzz-abc-123"}
}
```

Workers stop after completing their current request.

### Resuming a job

```json
// fuzz
{
  "action": "fuzz_resume",
  "params": {"fuzz_id": "fuzz-abc-123"}
}
```

### Canceling a job

```json
// fuzz
{
  "action": "fuzz_cancel",
  "params": {"fuzz_id": "fuzz-abc-123"}
}
```

## Stop conditions

Define automatic stop conditions to end a fuzz job early:

```json
// fuzz
{
  "action": "fuzz",
  "params": {
    "flow_id": "abc-123",
    "attack_type": "sequential",
    "positions": [
      {"id": "pos-0", "location": "query", "name": "id", "payload_set": "ids"}
    ],
    "payload_sets": {
      "ids": {"type": "range", "start": 1, "end": 10000}
    },
    "stop_on": {
      "status_codes": [500, 503],
      "error_count": 10,
      "latency_threshold_ms": 5000,
      "latency_baseline_multiplier": 3.0,
      "latency_window": 10
    }
  }
}
```

| Condition | Description |
|-----------|-------------|
| `status_codes` | Stop when any of these HTTP status codes is received |
| `error_count` | Stop when cumulative error count reaches this value |
| `latency_threshold_ms` | Stop when sliding window median latency exceeds this value |
| `latency_baseline_multiplier` | Stop when current median exceeds baseline median times this multiplier |
| `latency_window` | Sliding window size for latency detection (default: `10`) |

## Querying results

Use the `query` tool with `fuzz_results` to check progress and retrieve results:

```json
// query
{
  "action": "fuzz_results",
  "params": {
    "fuzz_id": "fuzz-abc-123"
  }
}
```

Results include status codes, response sizes, timing, and the payloads used for each request.

## Hooks integration

The fuzzer supports the same pre/post hook system as the resender. Use hooks to refresh authentication tokens between fuzz iterations:

```json
// fuzz
{
  "action": "fuzz",
  "params": {
    "flow_id": "abc-123",
    "attack_type": "sequential",
    "positions": [
      {"id": "pos-0", "location": "body_json", "json_path": "$.password", "payload_set": "passwords"}
    ],
    "payload_sets": {
      "passwords": {"type": "wordlist", "values": ["test1", "test2"]}
    },
    "hooks": {
      "pre_send": {
        "macro": "refresh-auth",
        "run_interval": "every_n",
        "n": 10
      }
    }
  }
}
```

## Practical use cases

### XSS scanning

```json
// fuzz
{
  "action": "fuzz",
  "params": {
    "flow_id": "search-flow",
    "attack_type": "sequential",
    "positions": [
      {"id": "pos-0", "location": "query", "name": "q", "payload_set": "xss"}
    ],
    "payload_sets": {
      "xss": {
        "type": "wordlist",
        "values": [
          "<script>alert(1)</script>",
          "<img src=x onerror=alert(1)>",
          "{{7*7}}",
          "${7*7}"
        ],
        "encoding": ["url_encode_query"]
      }
    },
    "tag": "xss-scan"
  }
}
```

### Brute force PIN

```json
// fuzz
{
  "action": "fuzz",
  "params": {
    "flow_id": "pin-verify-flow",
    "attack_type": "sequential",
    "positions": [
      {"id": "pos-0", "location": "body_json", "json_path": "$.pin", "payload_set": "pins"}
    ],
    "payload_sets": {
      "pins": {
        "type": "charset",
        "charset": "0123456789",
        "length": 4
      }
    },
    "concurrency": 3,
    "rate_limit_rps": 5,
    "stop_on": {"status_codes": [200]}
  }
}
```

### IDOR enumeration

```json
// fuzz
{
  "action": "fuzz",
  "params": {
    "flow_id": "user-profile-flow",
    "attack_type": "sequential",
    "positions": [
      {"id": "pos-0", "location": "query", "name": "user_id", "payload_set": "ids"}
    ],
    "payload_sets": {
      "ids": {"type": "range", "start": 1, "end": 1000}
    },
    "tag": "idor-scan"
  }
}
```

## Related pages

- [Fuzz tool reference](../tools/fuzz.md) -- MCP tool parameter reference
- [Resender](resender.md) -- Single request resend with mutations
- [Comparer](comparer.md) -- Compare fuzz results against baselines
- [Built-in codecs](../reference/built-in-codecs.md) -- Available encoding codecs
