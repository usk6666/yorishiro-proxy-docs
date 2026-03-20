# security

Configure runtime security settings including target scope rules, rate limits, diagnostic budgets, and SafetyFilter inspection. This tool is separate from `configure` to allow MCP clients to apply different approval policies for security changes.

## Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `action` | string | Yes | Action to perform (see below) |
| `params` | object | No | Action-specific parameters |

## Two-layer architecture

Target scope, rate limits, and budgets use a two-layer architecture:

- **Policy Layer** (immutable) -- set at startup from the configuration file. Defines the upper boundary. Cannot be modified at runtime.
- **Agent Layer** (mutable) -- controlled by this tool. Can further restrict access within Policy Layer boundaries.

### Evaluation order (target scope)

1. **Policy denies** -- always block (highest priority)
2. **Agent denies** -- block
3. **Policy allows** (if any) -- target must match at least one
4. **Agent allows** (if any) -- target must match at least one
5. All checks passed -- allow

When neither layer has rules, all targets are permitted (open mode).

## Actions

### set_target_scope

Replace all Agent Layer allow/deny rules. Use empty arrays to clear rules. Agent allow rules must fall within the Policy allow boundary.

#### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `allows` | array | Allow rules |
| `denies` | array | Deny rules |

### update_target_scope

Apply incremental changes to Agent Layer rules.

#### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `add_allows` | array | Allow rules to add |
| `remove_allows` | array | Allow rules to remove |
| `add_denies` | array | Deny rules to add |
| `remove_denies` | array | Deny rules to remove |

!!! warning
    `remove_denies` cannot remove Policy deny rules. Attempting to do so returns an error.

### get_target_scope

Returns both Policy and Agent Layer rules with enforcement mode. No parameters.

### test_target

Check a URL against current rules without making a request (dry run).

#### Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `url` | string | Yes | URL to test |

### set_rate_limits

Set Agent Layer rate limits. Omitted fields reset to 0 (no limit). Full-replace semantics.

#### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `max_requests_per_second` | number | Global rate limit in RPS (0 = no limit) |
| `max_requests_per_host_per_second` | number | Per-host rate limit in RPS (0 = no limit) |

### get_rate_limits

Returns Policy and Agent Layer rate limits with effective values. No parameters.

### set_budget

Set Agent Layer diagnostic budget limits. Omitted fields reset to 0 (no limit). Full-replace semantics.

#### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `max_total_requests` | integer | Max total requests for the session (0 = no limit) |
| `max_duration` | string | Max session duration, e.g. `"30m"` (`"0s"` = no limit) |

### get_budget

Returns Policy and Agent Layer budgets with effective values and current usage. No parameters.

### get_safety_filter

Returns the current SafetyFilter configuration and rules (read-only). SafetyFilter rules are part of the Policy Layer and cannot be modified at runtime. No parameters.

## Target rule fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `hostname` | string | Yes | Exact match or wildcard `*.example.com` |
| `ports` | integer[] | No | Match specific ports. Empty = all ports |
| `path_prefix` | string | No | Match URL path prefix. Empty = all paths |
| `schemes` | string[] | No | Match URL schemes (`http`, `https`). Empty = all schemes |

All specified fields must match for a rule to apply (AND logic).

## Response formats

### set_target_scope / update_target_scope

| Field | Type | Description |
|-------|------|-------------|
| `status` | string | `"updated"` |
| `allows` | array | Current Agent Layer allow rules |
| `denies` | array | Current Agent Layer deny rules |
| `mode` | string | `"open"` or `"enforcing"` |

### get_target_scope

| Field | Type | Description |
|-------|------|-------------|
| `policy` | object | Policy Layer: `allows`, `denies`, `source`, `immutable` |
| `agent` | object | Agent Layer: `allows`, `denies` |
| `effective_mode` | string | `"open"` or `"enforcing"` |

### test_target

| Field | Type | Description |
|-------|------|-------------|
| `allowed` | boolean | Whether the URL is allowed |
| `reason` | string | Reason for the decision |
| `layer` | string | Which layer decided (`"policy"` or `"agent"`) |
| `matched_rule` | object | The rule that matched (if any) |
| `tested_target` | object | Parsed URL components: hostname, port, scheme, path |

### set_rate_limits

| Field | Type | Description |
|-------|------|-------------|
| `status` | string | `"updated"` |
| `effective` | object | Merged Policy+Agent limits |
| `agent` | object | Current Agent Layer values |

### get_rate_limits

| Field | Type | Description |
|-------|------|-------------|
| `policy` | object | Policy Layer limits |
| `agent` | object | Agent Layer limits |
| `effective` | object | Merged effective limits |

### set_budget

| Field | Type | Description |
|-------|------|-------------|
| `status` | string | `"updated"` |
| `effective` | object | Merged Policy+Agent limits |
| `agent` | object | Current Agent Layer values |

### get_budget

| Field | Type | Description |
|-------|------|-------------|
| `policy` | object | Policy Layer budget |
| `agent` | object | Agent Layer budget |
| `effective` | object | Merged effective budget |
| `request_count` | integer | Requests made so far |
| `stop_reason` | string | Non-empty when budget is exhausted |

### get_safety_filter

| Field | Type | Description |
|-------|------|-------------|
| `enabled` | boolean | Whether SafetyFilter is active |
| `input_rules` | array | Input filter rules (block destructive payloads) |
| `output_rules` | array | Output filter rules (mask PII) |
| `immutable` | boolean | Always `true` -- rules cannot be changed at runtime |

## Examples

### Set target scope

```json
// security
{
  "action": "set_target_scope",
  "params": {
    "allows": [
      {"hostname": "api.target.com", "ports": [443], "schemes": ["https"]},
      {"hostname": "*.target.com"}
    ],
    "denies": [
      {"hostname": "admin.target.com"}
    ]
  }
}
```

### Update target scope incrementally

```json
// security
{
  "action": "update_target_scope",
  "params": {
    "add_allows": [{"hostname": "new-api.target.com"}],
    "remove_allows": [{"hostname": "old-api.target.com"}],
    "add_denies": [{"hostname": "staging.target.com"}]
  }
}
```

### Test a URL against scope rules

```json
// security
{
  "action": "test_target",
  "params": {
    "url": "https://api.target.com/v1/users"
  }
}
```

### Get current target scope

```json
// security
{
  "action": "get_target_scope"
}
```

### Set rate limits

```json
// security
{
  "action": "set_rate_limits",
  "params": {
    "max_requests_per_second": 10,
    "max_requests_per_host_per_second": 5
  }
}
```

### Set diagnostic budget

```json
// security
{
  "action": "set_budget",
  "params": {
    "max_total_requests": 1000,
    "max_duration": "30m"
  }
}
```

### Get current budget

```json
// security
{
  "action": "get_budget"
}
```

### Get SafetyFilter configuration

```json
// security
{
  "action": "get_safety_filter"
}
```

Example response:

```json
{
  "enabled": true,
  "input_rules": [
    {
      "id": "destructive-sql:drop",
      "name": "DROP statement",
      "pattern": "(compiled regex)",
      "targets": ["body", "url", "query"],
      "action": "block",
      "category": "destructive-sql"
    }
  ],
  "output_rules": [
    {
      "id": "credit-card:separated",
      "name": "Credit card (separated)",
      "pattern": "(compiled regex)",
      "targets": ["body"],
      "action": "mask",
      "replacement": "[MASKED:credit_card]",
      "category": "credit-card"
    }
  ],
  "immutable": true
}
```

## Related pages

- [Security model](../concepts/security-model.md) -- Security model concepts
- [Target scope](../features/target-scope.md) -- Target scope feature guide
- [Rate limits & budgets](../features/rate-limits.md) -- Rate limits and budget feature guide
- [SafetyFilter](../features/safety-filter.md) -- SafetyFilter feature guide
- [configure](configure.md) -- Runtime configuration including budget via merge semantics
