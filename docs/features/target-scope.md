# Target scope

Target scope controls which hosts and URLs the proxy is allowed to send requests to. It uses a two-layer architecture that separates policy-level controls (set by the operator) from agent-level controls (set by the AI agent at runtime).

Target scope is the only scope mechanism in the proxy. It is enforced by `HostScopeStep` at the head of the canonical Pipeline chain (`HostScope → HTTPScope → Budget → Safety → PluginPre → Intercept → Transform → PluginPost → Record`), so connections targeting a denied host are filtered before any other Step runs. The check uses `Envelope.Context.TargetHost` (populated from the CONNECT or SOCKS5 target) and never inspects the typed `Message`.

## Two-layer architecture

### Policy layer

The Policy layer is set at startup from `-target-policy-file`, or else the config file's `target_scope_policy` section. No MCP tool can modify it; only the operator can replace it at runtime by sending `SIGHUP` (see [Reloading the Policy layer](#reloading-the-policy-layer)). It defines the upper boundary for what the agent can access.

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
| `hostname` | string | Exact name, wildcard `*.example.com`, IPv4 literal, or IPv6 literal (bare `::1` or bracketed `[::1]`). Required for Agent layer rules; a Policy layer rule may omit it to match any host |
| `ports` | array of int | Match specific ports, `1`-`65535` (empty = all ports) |
| `path_prefix` | string | Literal, case-sensitive URL path prefix. Must start with `/` (empty = all paths) |
| `schemes` | array of string | `http` (plaintext) or `https` (TLS), case-insensitive (empty = both) |

All specified fields must match for a rule to apply (AND logic).

### Rule validation

Both layers validate rules with the same checks. Because rules use AND logic, a condition that can never match makes the whole rule match nothing — for a deny rule, that silently disables it. Such rules are rejected instead:

| Field | Rejected | Write instead |
|-------|----------|---------------|
| `hostname` | `internal.corp.` (trailing dot), `host:443`, `https://host`, `*`, `*.10.0.0.1` (wildcard over an IP) | `internal.corp`, `host` + `ports`, a bare IP |
| `path_prefix` | `/api/*`, `admin`, `/search?q=` | `/api/`, `/admin`, `/search` |
| `schemes` | `ws`, `wss`, `tcp`, `""` | `http` / `https`, or omit |

A Policy layer that fails validation stops the server at startup (or is rejected on reload). An Agent layer rule that fails is rejected by the `security` tool.

### Schemes mean transport confidentiality

`schemes` selects plaintext (`http`) versus TLS (`https`) — it is not an application-protocol name. WebSocket traffic maps `ws` to `http` and `wss` to `https`; raw TCP and gRPC map their TLS state the same way. A CONNECT tunnel is scoped as `https` regardless of what it carries.

SOCKS5 is the exception: the scope check runs before any tunnel byte is observed, so a SOCKS5 target has no scheme and does not match rules that carry `schemes`. Scope SOCKS5 targets by `hostname` and `ports`.

### IPv6 literals

`"::1"`, `"[::1]"`, and `"0:0::1"` are the same rule: hosts are compared in canonical form. A zone-qualified target such as `[fe80::1%eth0]` does not match a zone-less rule.

### Wildcard hostnames

Use `*.example.com` to match subdomains. This matches `sub.example.com` but not `example.com` itself:

```json
{"hostname": "*.target.com"}
```

## Config file policy setup

Set Policy layer rules in the config file:

```json
{
  "target_scope_policy": {
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

Policy rules are loaded at startup and cannot be changed by the agent.

## Reloading the Policy layer

You can replace the running Policy layer — `allows`, `denies`, `rate_limits`, and `budget` together — without restarting. Edit the policy source and send the server `SIGHUP` (Unix only):

```bash
kill -HUP "$(pgrep -f 'yorishiro-proxy server')"
```

- **Requires a policy source.** The handler is installed only when the server was started with `-target-policy-file` or `-config`, and only after the boot policy loads (log line `target scope policy reload enabled`). Otherwise `SIGHUP` terminates the process.
- **Full replacement.** Reloading is equivalent to starting with that file. A section omitted from the new file (for example `rate_limits`) is cleared. Only the policy is reloaded; other config sections are not.
- **Atomic.** The file is fully validated first. Malformed JSON, an invalid rule, or a negative limit rejects the whole reload, keeps the previous policy, and logs `target scope policy reload rejected; previous policy retained`.
- **Prefer `-target-policy-file` for live edits.** The dedicated policy file is decoded strictly, so a misspelled key such as `denys` rejects the reload. The `-config` source is decoded leniently, so a typo there can reload as "no policy". When a reload removes every allow and deny rule, a warning is logged — check for it.
- **Applies to in-flight work.** Live connections that the new policy denies are closed, held intercepts are re-checked on release, and running fuzz jobs re-check every variant. A deny that only names a `path_prefix` closes no connection; it is enforced per request.
- **Budget.** The request counter is not reset; the duration limit is re-armed against the new value, measured from the original session start.
- **Audit.** A successful reload is logged with the source path, file SHA-256, and rule counts. `get_target_scope` reports the policy's `source` and `loaded_at`.

!!! warning "Closing the terminal"
    With a policy source configured, closing the terminal that launched the server also sends `SIGHUP` and reloads instead of stopping the server. Stop it with `SIGINT` or `SIGTERM`.

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
    "loaded_at": "2026-10-16T09:30:00Z",
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

- **Connection gates** -- CONNECT, SOCKS5, and plain-HTTP forward connections are checked on admission, and re-checked whenever the scope changes; connections that would no longer be admitted are closed
- **Proxy data path** -- `HostScopeStep` and `HTTPScopeStep` run on every Send-direction envelope flowing through the Pipeline, before any other Step
- **resend_http / resend_ws / resend_grpc** -- before sending and after URL override
- **resend_raw** -- before connecting to the target address
- **fuzz_http / fuzz_ws / fuzz_grpc / fuzz_raw** -- on the template envelope, and again for every variant on the mutated message, so a scope change during a run stops the remaining variants
- **macro** -- on each step's URL before execution and at send time after template expansion
- **intercept** -- on every release (manual, timeout auto-release, and `modify_and_forward`), against the envelope about to be forwarded

This multi-point enforcement prevents SSRF via payload injection or template expansion.

### Deny symptoms

| Path | What the client sees |
|------|----------------------|
| HTTPS via CONNECT tunnel | `200 Connection Established` (the tunnel acknowledgement), then the connection closes and the inner TLS handshake fails with EOF |
| Plain-HTTP forward proxy | `403 Forbidden` with body `yorishiro-proxy: target blocked by scope` |
| MCP tool request (`resend_*`, `fuzz_*`, `macro`) | MCP error with the violation reason (fuzz records it as an error row and continues) |
| Intercept release | The `intercept` call succeeds; the forwarded HTTP request gets `403` with `{"error":"blocked by target_scope","blocked_by":"target_scope"}`, a WebSocket / gRPC message is dropped, and the block is recorded as a flow with `blocked_by: target_scope` |

## Related pages

- [Security tool reference](../tools/security.md) -- MCP tool parameter reference
- [Security model](../concepts/security-model.md) -- Architecture-level security design
- [SafetyFilter](safety-filter.md) -- Input/output content filtering
- [Rate limits & budgets](rate-limits.md) -- Rate limiting and budget controls
