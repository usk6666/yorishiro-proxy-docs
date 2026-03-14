# yorishiro-proxy

An AI-first MITM proxy tool controlled through MCP (Model Context Protocol).

yorishiro-proxy runs as an MCP server, giving AI agents full control over proxy operations through eleven MCP tools. Designed for use with Claude Code and other MCP-compatible agents, it enables automated security testing workflows without manual UI interaction. An embedded Web UI is also available for visual inspection and interactive use.

## Key features

- **Traffic interception & recording** -- MITM proxy with automatic CA certificate management
- **Resender** -- Replay requests with header/body/URL overrides, JSON patches, and raw HTTP editing
- **Fuzzer** -- Automated payload injection with sequential/parallel modes and async execution
- **Macro** -- Multi-step request sequences with variable extraction and template substitution
- **Intercept** -- Hold and inspect requests/responses in real time, then release, modify, or drop
- **Auto-transform** -- Automatic request/response modification rules for matching traffic
- **Target scope** -- Two-layer security boundary (Policy + Agent) to restrict reachable hosts
- **Multi-protocol** -- HTTP/1.x, HTTPS (MITM), HTTP/2, gRPC, WebSocket, Raw TCP, SOCKS5
- **Plugin system** -- Extend proxy behavior with Starlark scripts
- **Web UI** -- Embedded React/Vite dashboard for visual inspection and interactive testing
- **AI safety** -- SafetyFilter, rate limiting, and diagnostic budgets

## Getting started

New to yorishiro-proxy? Start here:

1. [Installation](getting-started/installation.md) -- Get the binary
2. [Quick setup](getting-started/quick-setup.md) -- Basic configuration
3. [MCP configuration](getting-started/mcp-configuration.md) -- Connect to your MCP client
4. [CA certificate](getting-started/ca-certificate.md) -- Trust the proxy CA
5. [First capture](getting-started/first-capture.md) -- Capture your first traffic

## MCP tools

All proxy operations are exposed through eleven MCP tools:

| Tool | Purpose |
|------|---------|
| [`proxy_start`](tools/proxy-start.md) | Start a proxy listener |
| [`proxy_stop`](tools/proxy-stop.md) | Stop a proxy listener |
| [`configure`](tools/configure.md) | Runtime configuration changes |
| [`query`](tools/query.md) | Retrieve flows, status, and configuration |
| [`resend`](tools/resend.md) | Replay requests with modifications |
| [`fuzz`](tools/fuzz.md) | Execute fuzz testing campaigns |
| [`macro`](tools/macro.md) | Multi-step macro workflows |
| [`intercept`](tools/intercept.md) | Act on intercepted requests |
| [`manage`](tools/manage.md) | Manage flow data and CA certificates |
| [`security`](tools/security.md) | Configure security policies |
| [`plugin`](tools/plugin.md) | Manage Starlark plugins |
