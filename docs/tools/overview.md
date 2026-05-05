# MCP tools overview

yorishiro-proxy exposes its entire feature set through MCP tools. Each tool handles a specific domain of functionality, and you interact with them by sending JSON objects through your MCP client.

## Per-protocol typed split (RFC-001 N9)

The resend and fuzz surfaces are split per protocol. Each tool owns a typed JSON schema mirroring the corresponding `Message` type (`HTTPMessage`, `WSMessage`, `GRPCStartMessage`/`GRPCDataMessage`, `RawMessage`), so AI agents address fields by name instead of round-tripping opaque URL strings or message-sequence indices. Per-variant `SafetyFilter` input gating is enforced at the same boundary across every typed tool.

## Tool catalog

| Tool | Purpose |
|------|---------|
| [`proxy_start`](proxy-start.md) | Start a proxy listener. Listen address, capture scope, TLS passthrough, intercept rules, auto-transform, TCP forwarding, protocol selection |
| [`proxy_stop`](proxy-stop.md) | Stop a single listener by name or all listeners |
| [`configure`](configure.md) | Modify runtime settings (upstream proxy, scope, TLS passthrough, intercept rules, auto-transform, limits, timeouts) |
| [`query`](query.md) | Retrieve flows, flow details, messages, status, config, CA cert, intercept queue, macros, fuzz jobs, fuzz results, technologies |
| [`resend_http`](resend-http.md) | Resend an HTTP/1.x or HTTP/2 request with `HTTPMessage`-typed schema |
| [`resend_ws`](resend-ws.md) | Resend a WebSocket frame with `WSMessage`-typed schema |
| [`resend_grpc`](resend-grpc.md) | Resend a gRPC RPC with `GRPCStart`/`Data`/`End`-typed schema |
| [`resend_raw`](resend-raw.md) | Resend a recorded raw byte payload (TCP or TLS upstream); covers what the legacy `tcp_replay` action did |
| [`fuzz_http`](fuzz-http.md) | Synchronous HTTP fuzz with `HTTPMessage`-typed positions |
| [`fuzz_ws`](fuzz-ws.md) | Synchronous WebSocket fuzz with `WSMessage`-typed positions |
| [`fuzz_grpc`](fuzz-grpc.md) | Synchronous gRPC fuzz with `GRPCStart`/`Data`-typed positions |
| [`fuzz_raw`](fuzz-raw.md) | Synchronous raw-byte fuzz; owns the from-scratch byte injection path |
| [`macro`](macro.md) | Multi-step workflows (define, run, delete) |
| [`intercept`](intercept.md) | Act on intercepted traffic (release, modify and forward, drop) with per-protocol typed payloads |
| [`manage`](manage.md) | Manage flow data and CA (delete, export, import, regenerate) |
| [`security`](security.md) | Security controls (target scope, rate limits, diagnostic budgets, SafetyFilter) |
| [`plugin_introspect`](plugin-introspect.md) | Read-only list of loaded plugins, their hook registrations, and redacted vars |

## Tool calling format

All MCP tool calls use JSON. Examples in this documentation follow this format:

```json
// tool_name
{
  "parameter": "value"
}
```

## Action-based tools

Several tools use an `action` parameter to select the operation:

| Tool | Actions |
|------|---------|
| `macro` | `define_macro`, `run_macro`, `delete_macro` |
| `intercept` | `release`, `modify_and_forward`, `drop` |
| `manage` | `delete_flows`, `export_flows`, `import_flows`, `regenerate_ca_cert` |
| `security` | `set_target_scope`, `update_target_scope`, `get_target_scope`, `test_target`, `set_rate_limits`, `get_rate_limits`, `set_budget`, `get_budget`, `get_safety_filter` |

The resend, fuzz, and plugin surfaces no longer use `action` dispatch -- each typed sibling tool is its own MCP entry point.

## Typical workflow

A common session follows this pattern:

1. **Start the proxy** with `proxy_start`
2. **Configure scope** with `configure` or at start time
3. **Capture traffic** by routing your application through the proxy
4. **Query flows** with `query` to find interesting requests
5. **Resend or fuzz** specific flows with the matching typed tool (`resend_http`, `fuzz_http`, etc.)
6. **Analyze results** with `query` to inspect responses (clients perform diff over query results -- there is no longer a `compare` action)
7. **Export findings** with `manage` for reporting
8. **Stop the proxy** with `proxy_stop`

## Related pages

- [Architecture](../concepts/architecture.md) -- How yorishiro-proxy is built
- [MCP-first design](../concepts/mcp-first-design.md) -- Why everything is an MCP tool
- [Quick setup](../getting-started/quick-setup.md) -- Getting started guide
