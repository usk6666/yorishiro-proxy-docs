# Flow comparison

The comparer lets you structurally compare two flows to identify differences in status codes, headers, body length, timing, and JSON content. It is designed as a triage tool for analyzing results from resend and fuzz operations.

## How it works

The compare action takes two flow IDs and produces a structured diff of their responses. It compares:

- **Status code** -- whether the HTTP status codes differ
- **Body length** -- byte-level size difference
- **Headers** -- added, removed, and changed headers
- **Timing** -- response duration difference in milliseconds
- **Body content** -- identical check, plus JSON key-level diff for JSON responses

## Basic comparison

```json
// resend
{
  "action": "compare",
  "params": {
    "flow_id_a": "original-flow-123",
    "flow_id_b": "mutated-flow-456"
  }
}
```

## Response structure

The compare result includes:

### Status code diff

```json
{
  "status_code": {
    "a": 200,
    "b": 403,
    "changed": true
  }
}
```

### Body length diff

```json
{
  "body_length": {
    "a": 1234,
    "b": 567,
    "delta": -667
  }
}
```

A positive delta means flow B has a larger body; negative means flow A is larger.

### Header diff

```json
{
  "headers_added": ["X-New-Header"],
  "headers_removed": ["X-Old-Header"],
  "headers_changed": {
    "Content-Length": {
      "a": "1234",
      "b": "567"
    }
  }
}
```

### Timing diff

```json
{
  "timing_ms": {
    "a": 150,
    "b": 2500,
    "delta": 2350
  }
}
```

### Body diff

For all content types:

```json
{
  "body": {
    "content_type": "application/json",
    "identical": false
  }
}
```

For JSON responses, the diff includes key-level analysis:

```json
{
  "body": {
    "content_type": "application/json",
    "identical": false,
    "json_diff": {
      "keys_added": ["error_message"],
      "keys_removed": ["data"],
      "keys_changed": ["status", "timestamp"]
    }
  }
}
```

!!! info "JSON key-level diff"
    The JSON diff operates on top-level keys only. It identifies which keys were added, removed, or changed between the two responses.

## Use cases

### Resend triage

After resending a request with mutations, compare the original and modified responses to identify what changed:

```json
// resend
{
  "action": "resend",
  "params": {
    "flow_id": "original-123",
    "body_patches": [{"json_path": "$.role", "value": "admin"}]
  }
}
```

Then compare:

```json
// resend
{
  "action": "compare",
  "params": {
    "flow_id_a": "original-123",
    "flow_id_b": "new-flow-id-from-resend"
  }
}
```

### Fuzz result analysis

Compare fuzz results against a baseline to find interesting responses:

```json
// resend
{
  "action": "compare",
  "params": {
    "flow_id_a": "baseline-flow",
    "flow_id_b": "fuzz-result-flow"
  }
}
```

Look for:

- **Status code changes** -- a 200 to 500 transition may indicate a vulnerability
- **Body length changes** -- significant size differences may reveal information disclosure
- **Timing anomalies** -- large timing deltas may indicate blind injection points
- **JSON key changes** -- new keys may reveal additional data exposure

### Before/after testing

Compare the same endpoint before and after a code change:

```json
// resend
{
  "action": "compare",
  "params": {
    "flow_id_a": "before-patch-flow",
    "flow_id_b": "after-patch-flow"
  }
}
```

## Getting full details

The comparer is designed as a triage tool. For full response bodies and detailed analysis, use the `query` tool:

```json
// query
{
  "action": "messages",
  "params": {
    "flow_id": "flow-to-inspect"
  }
}
```

## Related pages

- [Resend tool reference](../tools/resend.md) -- Compare action parameter reference
- [Resender](resender.md) -- Resend with mutations before comparing
- [Fuzzer](fuzzer.md) -- Generate multiple flows for comparison
