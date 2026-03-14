# Web UI overview

The yorishiro-proxy Web UI provides a browser-based interface for inspecting captured traffic, managing proxy settings, and running security testing tools. It is served automatically when you enable the Streamable HTTP transport.

## Enabling the Web UI

The Web UI is embedded in the yorishiro-proxy binary. To access it, start the MCP server with the `-mcp-http-addr` flag:

```bash
yorishiro-proxy -mcp-http-addr 127.0.0.1:9090
```

Once started, open `http://127.0.0.1:9090` in your browser. The Web UI and the MCP Streamable HTTP endpoint share the same address.

## Token authentication

When you set the `-mcp-http-token` flag, the Web UI requires token-based authentication:

```bash
yorishiro-proxy -mcp-http-addr 127.0.0.1:9090 -mcp-http-token my-secret-token
```

The browser will prompt you for the token on first visit. The token is stored in the browser's local storage for subsequent visits.

## How it works

The Web UI is a React single-page application that communicates with the proxy backend through MCP tool calls over Streamable HTTP. Every action you perform in the UI -- querying flows, starting the proxy, modifying intercept rules -- is executed as an MCP tool invocation under the hood. This means the Web UI has the same capabilities as any MCP client.

The architecture looks like this:

```
Browser (React SPA)
  → Streamable HTTP transport
    → MCP Server
      → MCP Tools (query, configure, resend, fuzz, etc.)
```

## Pages

The Web UI consists of the following pages:

| Page | Description |
|------|-------------|
| [Flows](flows.md) | Browse and filter captured HTTP, WebSocket, gRPC, and TCP flows |
| [Dashboard](dashboard.md) | Proxy status overview, traffic statistics, and live widgets |
| [Intercept](intercept.md) | Review and modify intercepted requests before forwarding |
| [Resender](resender.md) | Edit and resend captured requests with response comparison |
| [Fuzzer](fuzzer.md) | Create and manage fuzz testing campaigns |
| [Macros](macros.md) | Define and execute multi-step request sequences |
| [Security](security.md) | Manage target scope, rate limits, budgets, and safety filters |
| [Settings](settings.md) | Configure proxy listeners, TLS, protocols, and plugins |

## Related pages

- [Quick setup](../getting-started/quick-setup.md) -- Getting started with the proxy
- [MCP configuration](../getting-started/mcp-configuration.md) -- Configuring MCP transports
- [Architecture](../concepts/architecture.md) -- System architecture overview
