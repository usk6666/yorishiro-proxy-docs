# Multi-agent sharing

yorishiro-proxy supports simultaneous connections from multiple AI agents through its Streamable HTTP transport. This guide covers setting up shared proxy access with authentication, so that several agents can collaborate on the same assessment.

## How it works

By default, yorishiro-proxy starts an HTTP MCP transport on a random loopback port, allowing multiple agents to connect over HTTP and share the same proxy instance, flow store, and configuration. To use a fixed address for sharing, specify `-mcp-http-addr`.

The Streamable HTTP endpoint also serves the embedded Web UI, so you get both multi-agent MCP access and a visual dashboard on the same address.

## Setting up Streamable HTTP

### Use a fixed HTTP MCP address

By default, the HTTP MCP transport listens on a random port. For multi-agent setups, use a fixed address with `-mcp-http-addr`:

```json
{
  "mcpServers": {
    "yorishiro-proxy": {
      "command": "/path/to/bin/yorishiro-proxy",
      "args": [
        "server",
        "-stdio-mcp",
        "-insecure",
        "-log-file", "/tmp/yorishiro-proxy.log",
        "-mcp-http-addr", "127.0.0.1:3000"
      ]
    }
  }
}
```

When the proxy starts, it logs the access URL with an authentication token:

```
WebUI available url=http://127.0.0.1:3000/?token=<random-token>
```

### Set a fixed token

By default, the token is regenerated on each launch. To use a stable token that you can share with other agents, set it explicitly:

```json
{
  "mcpServers": {
    "yorishiro-proxy": {
      "command": "/path/to/bin/yorishiro-proxy",
      "args": [
        "server",
        "-stdio-mcp",
        "-insecure",
        "-log-file", "/tmp/yorishiro-proxy.log",
        "-mcp-http-addr", "127.0.0.1:3000",
        "-mcp-http-token", "my-shared-token"
      ]
    }
  }
}
```

You can also set the token via the `YP_MCP_HTTP_TOKEN` environment variable.

## Bearer token authentication

All Streamable HTTP requests require a Bearer token in the `Authorization` header. The first agent that launches yorishiro-proxy through stdio gets automatic access. Additional agents connecting over HTTP must include the token.

### Connecting an additional agent

Configure the second agent's MCP client to connect to the Streamable HTTP endpoint. The exact configuration depends on the MCP client, but the connection requires:

- **URL**: `http://127.0.0.1:3000/mcp`
- **Authorization**: `Bearer my-shared-token`

For Claude Code, you can configure a Streamable HTTP MCP server in `.mcp.json`:

```json
{
  "mcpServers": {
    "yorishiro-proxy-remote": {
      "type": "streamable-http",
      "url": "http://127.0.0.1:3000/mcp",
      "headers": {
        "Authorization": "Bearer my-shared-token"
      }
    }
  }
}
```

## Simultaneous connections

Multiple agents can connect to the same yorishiro-proxy instance and perform operations concurrently:

- **Shared flow store** -- all agents see the same captured flows
- **Shared configuration** -- changes made by one agent (e.g., capture scope, intercept rules) affect all agents
- **Independent operations** -- each agent can run its own resend, fuzz, and macro operations

### Use case: Parallel testing

One common pattern is to divide testing responsibilities across agents:

**Agent A** -- handles reconnaissance and traffic capture:

```json
// proxy_start
{
  "listen_addr": "127.0.0.1:8080",
  "capture_scope": {
    "includes": [{"hostname": "target.example.com"}]
  }
}
```

**Agent B** -- runs fuzzing campaigns on captured flows:

```json
// fuzz
{
  "action": "fuzz",
  "params": {
    "flow_id": "<flow-id-from-agent-a>",
    "attack_type": "sequential",
    "positions": [
      {
        "id": "pos-0",
        "location": "body_json",
        "json_path": "$.user_id",
        "payload_set": "ids"
      }
    ],
    "payload_sets": {
      "ids": {"type": "range", "start": 1, "end": 100}
    },
    "tag": "agent-b-idor"
  }
}
```

**Agent C** -- reviews results and performs targeted resends:

```json
// query
{
  "resource": "fuzz_results",
  "fuzz_id": "<fuzz-id>",
  "filter": {"outliers_only": true}
}
```

### Use case: Human + AI collaboration

A security researcher uses the Web UI for visual inspection while an AI agent drives automated testing:

1. The AI agent starts the proxy and captures traffic via MCP tools
2. The researcher opens the Web UI at `http://127.0.0.1:3000/?token=my-shared-token` to browse flows visually
3. The AI agent runs fuzz campaigns
4. The researcher reviews results in the Fuzz page of the Web UI
5. Both can use the Resender to replay interesting requests

## Session management

### Checking connection status

Any connected agent can check the proxy status:

```json
// query
{"resource": "status"}
```

This returns the number of active connections, total flows, uptime, and other health metrics.

### Configuration coordination

Since configuration is shared, agents should coordinate to avoid conflicting changes. For example, if one agent narrows the capture scope, it affects all agents.

Use `query config` to check the current configuration before making changes:

```json
// query
{"resource": "config"}
```

### Rate limits and budgets

Rate limits and diagnostic budgets apply globally across all agents. If Agent A sets a rate limit of 10 RPS, the combined traffic from all agents is subject to that limit.

```json
// security
{
  "action": "set_rate_limits",
  "params": {
    "max_requests_per_second": 20,
    "max_requests_per_host_per_second": 10
  }
}
```

Check current budget usage:

```json
// security
{"action": "get_budget"}
```

## Related pages

- [Architecture](../concepts/architecture.md) -- system architecture overview
- [MCP-first design](../concepts/mcp-first-design.md) -- design philosophy
- [MCP configuration](../getting-started/mcp-configuration.md) -- MCP setup guide
