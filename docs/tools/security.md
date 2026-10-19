# security

Configure runtime security settings including target scope rules, rate limits, diagnostic budgets, and SafetyFilter inspection. This tool is separate from `configure` to allow MCP clients to apply different approval policies for security changes.

## Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `action` | string | Yes | Action to perform (see below) |
| `params` | object | No | Action-specific parameters |

## Two-layer architecture

Target scope, rate limits, and budgets use a two-layer architecture:

- **Policy Layer** (immutable by the agent) -- set at startup from `-target-policy-file`, or else the config file's `target_scope_policy` section. Defines the upper boundary. No MCP tool can modify it; only the operator can replace it at runtime with a [`SIGHUP` reload](#reloading-the-policy-layer-operator-only). Policy rules are validated at startup with the same rules as this tool (except that a policy rule may omit `hostname`), and an invalid policy stops the server instead of loading a rule that silently matches nothing.
- **Agent Layer** (mutable) -- controlled by this tool. Can further restrict access within Policy Layer boundaries.

### Evaluation order (target scope)

1. **Policy denies** -- always block (highest priority)
2. **Agent denies** -- block
3. **Policy allows** (if any) -- target must match at least one
4. **Agent allows** (if any) -- target must match at least one
5. All checks passed -- allow

When neither layer has rules, all targets are permitted (open mode).

### Where target scope applies

A single shared evaluator applies both layers to **all** traffic:

| Traffic source | Deny symptom |
|----------------|--------------|
| HTTPS via CONNECT tunnel | The proxy returns `200 Connection Established`, then closes the connection; the client's inner TLS handshake fails with EOF |
| Plain-HTTP forward proxy | `403 Forbidden` with body `yorishiro-proxy: target blocked by scope` |
| MCP tool requests (`resend_*`, `fuzz_*`, `macro`) | MCP error response with the violation reason. Fuzz jobs re-check scope for every variant they send |
| Intercept release (`release`, `modify_and_forward`, timeout auto-release) | The `intercept` call succeeds, but scope is checked again at release time against the request actually forwarded (for `modify_and_forward`, the modified one). An HTTP request gets `403 Forbidden` with body `{"error":"blocked by target_scope","blocked_by":"target_scope"}`, a WebSocket / gRPC message is dropped, and the block is recorded as a flow with `blocked_by: target_scope` |

After `set_target_scope`, `update_target_scope`, or a policy reload, live connections that an identical new connection would no longer be admitted for are closed. A connection with held intercepts is revoked at once (new requests get `403`) but closed only after its holds resolve. A deny that only names a `path_prefix` closes no connection; it is enforced per request.

### Reloading the Policy Layer (operator only)

The operator can replace the running Policy Layer -- target scope `allows` / `denies`, `rate_limits`, and `budget` together -- by editing the policy source and sending the server process `SIGHUP` (Unix only). There is deliberately no MCP action for this.

- **Enabled only with a policy source** -- the handler is installed only when the server was started with `-target-policy-file` or `-config`. Without either, `SIGHUP` terminates the process as before. Wait for the Info log line `target scope policy reload enabled` before signaling.
- **Terminal hangup** -- closing the launching terminal also sends `SIGHUP`, which now triggers a reload instead of stopping the server. Stop it explicitly with `SIGTERM` or `SIGINT`.
- **Full, atomic replacement** -- the new source is validated completely before anything changes. A section omitted from the new file is cleared. On any validation error, the previous policy stays in force and `target scope policy reload rejected; previous policy retained` is logged.
- **Prefer `-target-policy-file` for live edits** -- it is decoded strictly, so a misspelled field rejects the reload. The `-config` source is decoded leniently, so a typo can reload as "no policy"; a Warn line reports when a reload drops every allow and deny rule.
- **Agent Layer** -- agent rules are kept unchanged. An agent allow rule the new policy no longer covers simply stops matching, so narrowing the policy never widens what the agent can reach.
- **Budget** -- the request counter is not reset; the duration limit is re-armed against the new value from the original session start.
- **Audit** -- a successful reload logs `target scope policy reloaded` with the source path, file `sha256`, and rule counts. `get_target_scope` reports `source` and `loaded_at`.

!!! warning "Protect the policy file"
    Keep the policy file in a directory that neither the proxy's user nor the agent can write, and never where `manage` export can write.

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

When the Input Filter blocks a native gRPC call, the RPC fails with `grpc-status: 7` (`PERMISSION_DENIED`) and `grpc-message: blocked by safety_filter`; the matched rule IDs travel comma-separated in the `x-yorishiro-matched-rules` trailer. A blocked message in a bidi or client stream ends the whole RPC. gRPC-Web messages are dropped without a status.

## Target rule fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `hostname` | string | Yes | Exact match or wildcard `*.example.com`. IPv4 and IPv6 literals are accepted; IPv6 may be bare (`::1`) or bracketed (`[::1]`), and every zone-less form of one address is the same host. A wildcard over an IP literal is rejected |
| `ports` | integer[] | No | Match specific ports. Empty = all ports |
| `path_prefix` | string | No | Literal, case-sensitive prefix of the request path (not a glob). Must start with `/`; `*`, `?`, and `#` are rejected. Matches when either the wire-form path or its percent-decoded form starts with the prefix (`/admin` also matches `/%61dmin`). Empty = all paths |
| `schemes` | string[] | No | Transport confidentiality: `http` = plaintext, `https` = TLS. No other value is accepted. Empty = both. See [What schemes means](#what-schemes-means) |

All specified fields must match for a rule to apply (AND logic).

### What `schemes` means

`schemes` is a **transport confidentiality** axis, not an application-protocol axis. It applies to every protocol the proxy scopes:

- A WebSocket target matches as `http` (`ws://`) or `https` (`wss://`). `"schemes": ["ws"]` is rejected.
- A raw TCP or gRPC target matches as `https` when the leg is TLS-terminated, and `http` when it is cleartext.
- A `CONNECT` tunnel matches as `https`, even if it turns out to carry another protocol.
- SOCKS5 targets are scope-checked before the tunnel carries any byte, so no `schemes`-bearing rule matches them. Scope them by `hostname` / `ports`.

A `schemes` condition narrows a rule in **both directions**: a deny rule with `"schemes": ["https"]` does not block the plaintext dial to the same host, and an allow rule with `"schemes": ["https"]` does not permit one. Omit `schemes` when you mean "this host, on any transport" -- the safer default for deny rules.

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
| `policy` | object | Policy Layer: `allows`, `denies`, `source`, `loaded_at`, `immutable` |
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
