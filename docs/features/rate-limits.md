# Rate limits and diagnostic budgets

Rate limits and diagnostic budgets control how aggressively the proxy sends requests. Rate limits cap the requests per second, while budgets cap the total number of requests or session duration. Both use the two-layer architecture (Policy + Agent) for safe operation.

## Rate limits

### Global RPS

Limit the total number of requests per second across all hosts:

```json
// security
{
  "action": "set_rate_limits",
  "params": {
    "max_requests_per_second": 10
  }
}
```

### Per-host RPS

Limit requests per second to each individual host:

```json
// security
{
  "action": "set_rate_limits",
  "params": {
    "max_requests_per_host_per_second": 5
  }
}
```

### Combined limits

Set both global and per-host limits:

```json
// security
{
  "action": "set_rate_limits",
  "params": {
    "max_requests_per_second": 50,
    "max_requests_per_host_per_second": 10
  }
}
```

### Clearing limits

Set a limit to `0` to remove it. Omitted fields also reset to 0 (no limit) because `set_rate_limits` uses full-replace semantics:

```json
// security
{
  "action": "set_rate_limits",
  "params": {
    "max_requests_per_second": 0,
    "max_requests_per_host_per_second": 0
  }
}
```

### Rate limit enforcement

Requests that exceed rate limits receive a `429 Too Many Requests` response with an `X-Blocked-By: rate_limit` header.

## Diagnostic budgets

Budgets limit the total scope of a testing session by capping either the number of requests or the duration.

### Request budget

Limit the total number of requests:

```json
// security
{
  "action": "set_budget",
  "params": {
    "max_total_requests": 1000
  }
}
```

### Duration budget

Limit the session duration using Go duration strings:

```json
// security
{
  "action": "set_budget",
  "params": {
    "max_duration": "30m"
  }
}
```

Supported duration formats: `"30m"`, `"1h"`, `"2h30m"`, `"0s"` (no limit).

### Combined budgets

```json
// security
{
  "action": "set_budget",
  "params": {
    "max_total_requests": 5000,
    "max_duration": "2h"
  }
}
```

### Budget exhaustion

When a budget is exhausted, the proxy automatically stops accepting new requests. The `stop_reason` field in the budget status indicates why:

- `"max_total_requests exceeded"`
- `"max_duration exceeded"`

## Two-layer architecture

Rate limits and budgets use the same two-layer design as [Target scope](target-scope.md):

### Policy layer

Set at startup from the config file. Defines the upper bounds that the agent cannot exceed. Immutable at runtime.

### Agent layer

Controlled by the AI agent via the `security` tool. Can set limits that are equal to or stricter than Policy limits, but never more permissive.

### Effective values

The effective limit is always the stricter of the two layers. For example, if the Policy sets 50 RPS and the Agent sets 10 RPS, the effective limit is 10 RPS.

## Viewing current state

### Rate limits

```json
// security
{
  "action": "get_rate_limits"
}
```

Response:

```json
{
  "policy": {
    "max_requests_per_second": 50,
    "max_requests_per_host_per_second": 20
  },
  "agent": {
    "max_requests_per_second": 10,
    "max_requests_per_host_per_second": 5
  },
  "effective": {
    "max_requests_per_second": 10,
    "max_requests_per_host_per_second": 5
  }
}
```

### Budget status

```json
// security
{
  "action": "get_budget"
}
```

Response:

```json
{
  "policy": {
    "max_total_requests": 5000,
    "max_duration": "2h0m0s"
  },
  "agent": {
    "max_total_requests": 1000,
    "max_duration": "30m0s"
  },
  "effective": {
    "max_total_requests": 1000,
    "max_duration": "30m0s"
  },
  "request_count": 142,
  "stop_reason": ""
}
```

## Budget via configure tool

You can also set budgets using the `configure` tool, which uses merge semantics by default (only update provided fields, keep others unchanged):

```json
// configure
{
  "budget": {
    "max_total_requests": 1000,
    "max_duration": "30m"
  }
}
```

This differs from the `security` tool's `set_budget` action, which uses full-replace semantics (omitted fields reset to 0).

## Config file setup

Set Policy layer limits in the config file:

```json
{
  "rate_limits": {
    "max_requests_per_second": 50,
    "max_requests_per_host_per_second": 20
  },
  "budget": {
    "max_total_requests": 10000,
    "max_duration": "4h"
  }
}
```

## Practical use cases

### Conservative scanning

For initial reconnaissance with minimal impact:

```json
// security
{
  "action": "set_rate_limits",
  "params": {
    "max_requests_per_second": 5,
    "max_requests_per_host_per_second": 2
  }
}
```

```json
// security
{
  "action": "set_budget",
  "params": {
    "max_total_requests": 500,
    "max_duration": "15m"
  }
}
```

### Intensive fuzzing

For targeted fuzzing within operator-approved limits:

```json
// security
{
  "action": "set_rate_limits",
  "params": {
    "max_requests_per_second": 100,
    "max_requests_per_host_per_second": 50
  }
}
```

## Related pages

- [Security tool reference](../tools/security.md) -- MCP tool parameter reference
- [Configure tool reference](../tools/configure.md) -- Budget via configure
- [Target scope](target-scope.md) -- Host-level access control
- [SafetyFilter](safety-filter.md) -- Content-level filtering
- [Config file](../configuration/config-file.md) -- Policy layer configuration
