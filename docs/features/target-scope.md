# Target scope

Target scope controls which hosts and URLs the proxy is allowed to send requests to. It uses a two-layer architecture that separates policy-level controls (set by the operator) from agent-level controls (set by the AI agent at runtime).

## Two-layer architecture

### Policy layer

The Policy layer is set at startup from the configuration file and **cannot be modified at runtime**. It defines the upper boundary for what the agent can access.

### Agent layer

The Agent layer is controlled by the AI agent via the `security` tool. It can further restrict access within the Policy layer boundaries, but cannot expand beyond them.

### Design rationale

This separation ensures that:

- **Operators** define the maximum scope of testing (Policy layer)
- **AI agents** can self-restrict within that scope (Agent layer)
- **Safety is enforced** even if the agent attempts to bypass restrictions

## Evaluation order

When a request URL is checked against target scope:

1. **Policy denies** -- always block (highest priority)
2. **Agent denies** -- block
3. **Policy allows** (if any) -- target must match at least one, otherwise block
4. **Agent allows** (if any) -- target must match at least one, otherwise block
5. All checks passed -- allow

The priority order is: **deny > allow**, and within each type: **policy > agent**.

When neither layer has rules, all targets are permitted (open mode).

## Rule definition

Each target rule specifies:

| Field | Type | Description |
|-------|------|-------------|
| `hostname` | string | Exact match or wildcard `*.example.com` (required) |
| `ports` | array of int | Match specific ports (empty = all ports) |
| `path_prefix` | string | Match URL path prefix (empty = all paths) |
| `schemes` | array of string | Match URL schemes: `http`, `https` (empty = all schemes) |

All specified fields must match for a rule to apply (AND logic).

### Wildcard hostnames

Use `*.example.com` to match subdomains. This matches `sub.example.com` but not `example.com` itself:

```json
{"hostname": "*.target.com"}
```

## Config file policy setup

Set Policy layer rules in the config file:

```json
{
  "target_scope": {
    "allows": [
      {"hostname": "*.target.com"},
      {"hostname": "api.partner.com", "ports": [443], "schemes": ["https"]}
    ],
    "denies": [
      {"hostname": "*.internal.corp"},
      {"hostname": "admin.target.com"}
    ]
  }
}
```

Policy rules are loaded at startup and are immutable.

## MCP tool agent setup

### Set agent scope (full replace)

Replace all Agent layer rules:

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

Use empty arrays to clear all agent rules:

```json
// security
{
  "action": "set_target_scope",
  "params": {
    "allows": [],
    "denies": []
  }
}
```

!!! warning "Policy boundary enforcement"
    Agent allow rules must fall within the Policy allow boundary. If any agent allow rule is outside the policy scope, the entire operation is rejected.

### Update agent scope (incremental)

Apply incremental changes to Agent layer rules:

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

!!! note "Policy deny protection"
    You cannot remove Policy deny rules via `remove_denies`. Attempting to do so returns an error because policy rules are immutable.

### View current scope

```json
// security
{
  "action": "get_target_scope"
}
```

Returns both Policy and Agent layer rules with the enforcement mode:

```json
{
  "policy": {
    "allows": [{"hostname": "*.target.com"}],
    "denies": [{"hostname": "*.internal.corp"}],
    "source": "config file",
    "immutable": true
  },
  "agent": {
    "allows": [{"hostname": "api.target.com"}],
    "denies": [{"hostname": "admin.target.com"}]
  },
  "effective_mode": "enforcing"
}
```

## URL test

Test a URL against current rules without making a request:

```json
// security
{
  "action": "test_target",
  "params": {
    "url": "https://api.target.com/v1/users"
  }
}
```

The response tells you:

- Whether the URL is allowed
- Which layer made the decision
- Which rule matched
- The parsed URL components that were evaluated

```json
{
  "allowed": true,
  "reason": "",
  "layer": "agent",
  "matched_rule": {"hostname": "api.target.com"},
  "tested_target": {
    "hostname": "api.target.com",
    "port": 443,
    "scheme": "https",
    "path": "/v1/users"
  }
}
```

## Enforcement modes

| Mode | Description |
|------|-------------|
| `open` | No rules configured in either layer; all targets permitted |
| `enforcing` | At least one rule exists; targets are checked against rules |

## Where target scope is enforced

Target scope is checked at multiple points:

- **Resend** -- before sending the request and after URL override
- **Resend raw** -- before connecting to the target address
- **Fuzz** -- on the template flow URL and after payload injection
- **Macro** -- on each step's URL before execution and at send time after template expansion
- **Intercept** -- on modify_and_forward URL overrides

This multi-point enforcement prevents SSRF via payload injection or template expansion.

## Related pages

- [Security tool reference](../tools/security.md) -- MCP tool parameter reference
- [Security model](../concepts/security-model.md) -- Architecture-level security design
- [SafetyFilter](safety-filter.md) -- Input/output content filtering
- [Rate limits & budgets](rate-limits.md) -- Rate limiting and budget controls
