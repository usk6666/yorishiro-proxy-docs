# MCP tools overview

yorishiro-proxy exposes its entire feature set through 11 MCP tools. Each tool handles a specific domain of functionality, and you interact with them by sending JSON objects through your MCP client.

## Tool catalog

| Tool | Purpose | Key actions |
|------|---------|-------------|
| [`proxy_start`](proxy-start.md) | Start proxy listeners | Listen address, capture scope, TLS passthrough, intercept rules, auto-transform, TCP forwarding, protocol selection |
| [`proxy_stop`](proxy-stop.md) | Stop proxy listeners | Stop single listener by name or all listeners |
| [`configure`](configure.md) | Modify runtime settings | Upstream proxy, capture scope, TLS passthrough, intercept rules, auto-transform, connection limits, timeouts |
| [`query`](query.md) | Retrieve information | Flows, flow details, messages, status, config, CA cert, intercept queue, macros, fuzz jobs, fuzz results, technologies |
| [`resend`](resend.md) | Resend and replay requests | Resend with mutations, raw TCP resend, TCP replay, compare flows |
| [`fuzz`](fuzz.md) | Fuzz testing campaigns | Start fuzz jobs, pause, resume, cancel |
| [`macro`](macro.md) | Multi-step workflows | Define macros, run macros, delete macros |
| [`intercept`](intercept.md) | Act on intercepted traffic | Release, modify and forward, drop |
| [`manage`](manage.md) | Manage flow data and CA | Delete flows, export (JSONL/HAR), import, regenerate CA certificate |
| [`security`](security.md) | Security controls | Target scope, rate limits, diagnostic budgets, SafetyFilter |
| [`plugin`](plugin.md) | Manage Starlark plugins | List, reload, enable, disable |

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
| `resend` | `resend`, `resend_raw`, `tcp_replay`, `compare` |
| `fuzz` | `fuzz`, `fuzz_pause`, `fuzz_resume`, `fuzz_cancel` |
| `macro` | `define_macro`, `run_macro`, `delete_macro` |
| `intercept` | `release`, `modify_and_forward`, `drop` |
| `manage` | `delete_flows`, `export_flows`, `import_flows`, `regenerate_ca_cert` |
| `security` | `set_target_scope`, `update_target_scope`, `get_target_scope`, `test_target`, `set_rate_limits`, `get_rate_limits`, `set_budget`, `get_budget`, `get_safety_filter` |
| `plugin` | `list`, `reload`, `enable`, `disable` |

## Typical workflow

A common session follows this pattern:

1. **Start the proxy** with `proxy_start`
2. **Configure scope** with `configure` or at start time
3. **Capture traffic** by routing your application through the proxy
4. **Query flows** with `query` to find interesting requests
5. **Resend or fuzz** specific flows with `resend` or `fuzz`
6. **Analyze results** with `query` to inspect responses
7. **Export findings** with `manage` for reporting
8. **Stop the proxy** with `proxy_stop`

## Related pages

- [Architecture](../concepts/architecture.md) -- How yorishiro-proxy is built
- [MCP-first design](../concepts/mcp-first-design.md) -- Why everything is an MCP tool
- [Quick setup](../getting-started/quick-setup.md) -- Getting started guide
