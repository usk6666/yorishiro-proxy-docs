# fuzz

Execute fuzz testing campaigns on recorded proxy data. Fuzz jobs run asynchronously -- you get a `fuzz_id` immediately and query results separately.

## Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `action` | string | Yes | Action to perform: `fuzz`, `fuzz_pause`, `fuzz_resume`, `fuzz_cancel` |
| `params` | object | Yes | Action-specific parameters (see below) |

## Actions

### fuzz

Start an asynchronous fuzz campaign against a recorded flow.

#### Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `flow_id` | string | Yes | | Template flow ID |
| `attack_type` | string | Yes | | `"sequential"` (one position at a time) or `"parallel"` (all positions simultaneously, zip) |
| `positions` | array | Yes | | Payload injection points |
| `payload_sets` | object | Yes | | Named payload sets |
| `concurrency` | integer | No | `1` | Number of concurrent workers |
| `rate_limit_rps` | number | No | `0` | Requests per second limit (0 = unlimited) |
| `delay_ms` | integer | No | | Fixed delay between requests in ms |
| `timeout_ms` | integer | No | `30000` | Per-request timeout in ms |
| `max_retries` | integer | No | `0` | Retry count per failed request |
| `stop_on` | object | No | | Automatic stop conditions |
| `tag` | string | No | | Tag to label the fuzz job |
| `hooks` | object | No | | Pre/post hooks for macro integration |

#### positions

Each position specifies where to inject payloads:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | Yes | Unique position identifier |
| `location` | string | Yes | `header`, `path`, `query`, `body_regex`, `body_json`, `cookie` |
| `name` | string | Conditional | Header name, query key, or cookie name (required for header/query/cookie) |
| `json_path` | string | Conditional | JSON path for body_json (e.g. `"$.password"`) |
| `mode` | string | No | `"replace"` (default), `"add"`, or `"remove"` |
| `match` | string | No | Regex for partial replacement. Capture groups replace only the group |
| `payload_set` | string | Conditional | Name of the payload set to use (not required for remove mode) |

#### payload_sets

Each payload set specifies how to generate payloads:

| Type | Fields | Description |
|------|--------|-------------|
| `wordlist` | `values` (string[]) | Explicit list of payload strings |
| `file` | `path` (string) | Relative path under `~/.yorishiro-proxy/wordlists/` |
| `range` | `start`, `end`, `step` (integer) | Numeric range |
| `sequence` | `start`, `end`, `step`, `format` (string) | Formatted sequence (e.g. `"user%04d"`) |
| `charset` | `charset` (string), `length` (integer) | All combinations of charset characters |
| `case_variation` | `input` (string) | All case variations of the input string |
| `null_byte_injection` | `input` (string) | Null byte injection variants |

An optional `encoding` array applies codec transformations as a pipeline (max 10 codecs). Available codecs: `base64`, `base64url`, `url_encode_query`, `url_encode_path`, `url_encode_full`, `double_url_encode`, `hex`, `html_entity`, `html_escape`, `unicode_escape`, `md5`, `sha256`, `lower`, `upper`.

#### stop_on

| Field | Type | Description |
|-------|------|-------------|
| `status_codes` | integer[] | Stop when any of these status codes is received |
| `error_count` | integer | Stop when cumulative error count reaches this value |
| `latency_threshold_ms` | integer | Stop when sliding window median latency exceeds this value |
| `latency_baseline_multiplier` | number | Stop when median exceeds baseline median times this multiplier |
| `latency_window` | integer | Sliding window size for latency detection (default: 10) |

#### Response

| Field | Type | Description |
|-------|------|-------------|
| `fuzz_id` | string | Fuzz job ID |
| `status` | string | Job status |
| `total_requests` | integer | Total number of requests to send |
| `tag` | string | Tag (if provided) |
| `message` | string | Status message |

### fuzz_pause

Pause a running fuzz job. Workers stop after completing their current request.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `fuzz_id` | string | Yes | Fuzz job ID to pause |

### fuzz_resume

Resume a paused fuzz job.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `fuzz_id` | string | Yes | Fuzz job ID to resume |

### fuzz_cancel

Cancel a running or paused fuzz job.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `fuzz_id` | string | Yes | Fuzz job ID to cancel |

Control action response: `fuzz_id`, `action`, `status`.

## Examples

### Fuzz with sequential attack

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
    },
    "tag": "auth-test"
  }
}
```

### Fuzz with parallel attack

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

### Fuzz with encoded payloads

```json
// fuzz
{
  "action": "fuzz",
  "params": {
    "flow_id": "abc-123",
    "attack_type": "sequential",
    "positions": [
      {"id": "pos-0", "location": "query", "name": "search", "payload_set": "xss"}
    ],
    "payload_sets": {
      "xss": {
        "type": "wordlist",
        "values": ["<script>alert(1)</script>", "<img onerror=alert(1)>"],
        "encoding": ["url_encode_query"]
      }
    }
  }
}
```

### Fuzz with charset generator

```json
// fuzz
{
  "action": "fuzz",
  "params": {
    "flow_id": "abc-123",
    "attack_type": "sequential",
    "positions": [
      {"id": "pos-0", "location": "query", "name": "pin", "payload_set": "pins"}
    ],
    "payload_sets": {
      "pins": {
        "type": "charset",
        "charset": "0123456789",
        "length": 4
      }
    }
  }
}
```

### Pause a fuzz job

```json
// fuzz
{
  "action": "fuzz_pause",
  "params": {"fuzz_id": "fuzz-abc-123"}
}
```

### Resume a fuzz job

```json
// fuzz
{
  "action": "fuzz_resume",
  "params": {"fuzz_id": "fuzz-abc-123"}
}
```

### Cancel a fuzz job

```json
// fuzz
{
  "action": "fuzz_cancel",
  "params": {"fuzz_id": "fuzz-abc-123"}
}
```

## Related pages

- [Fuzzer](../features/fuzzer.md) -- Fuzzer feature guide
- [query](query.md) -- Query fuzz results with `fuzz_jobs` and `fuzz_results` resources
- [resend](resend.md) -- Resend individual requests
- [macro](macro.md) -- Multi-step workflows for hook integration
